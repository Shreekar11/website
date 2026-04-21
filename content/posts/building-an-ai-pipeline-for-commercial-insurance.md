---
title: "Building an AI Pipeline for Commercial Insurance"
date: "2026-04-21"
featured_image: "/posts/ai-pipeline-for-commercial-insurance.webp"
summary: "A deep dive into building an AI system to extract and understand commercial insurance policies as knowledge graphs, covering the domain constraints, pipeline architecture, and lessons learned."
description: "This post explores the challenges and solutions in processing commercial insurance documents using AI, from OCR and chunking to entity resolution and graph indexing, with insights into why policies are inherently graph-structured."
toc: false
readTime: true
autonumber: false
math: true
draft: false
---

## Table of Contents

- [Why I Built This](#why-i-built-this)
- [What I Had to Learn About Commercial Policies](#what-i-had-to-learn-about-commercial-policies)
- [The Pipeline I Designed, and Why](#the-pipeline-i-designed-and-why)
- [Orchestrating Long-Running Work with Temporal](#orchestrating-long-running-work-with-temporal)
- [Three Problems That Came Along the Way](#three-problems-that-came-along-the-way)
   - [Problem 1: Section Boundaries Don't Respect Pages](#problem-1-section-boundaries-dont-respect-pages)
   - [Problem 2: Entities in Different Extraction Batches Can't See Each Other](#problem-2-entities-in-different-extraction-batches-cant-see-each-other)
   - [Problem 3: Citations Break When the LLM Paraphrases](#problem-3-citations-break-when-the-llm-paraphrases)
- [Connecting Two Data Stores Without Glue Code](#connecting-two-data-stores-without-glue-code)
- [What I Think Is Still Genuinely Hard](#what-i-think-is-still-genuinely-hard)
- [Where This Could Go: Agentic Extraction Loop](#where-this-could-go-agentic-extraction-loop)



## Why I Built This

I built Insura-AI as a proof-of-work project while exploring the AI-native insurance space. The pitch was simple enough: commercial insurance documents are complex, and the underwriting workflow that processes them involves a lot of manual data extraction. An AI pipeline that could read a policy PDF and return structured, queryable data - coverages, limits, exclusions, which endorsements modify what - would save significant time per submission.

The engineering challenge sounded tractable. Document ingestion, LLM extraction, maybe some vector search on top. Few weeks in, I had a prototype that sort of worked on clean documents.

Two months in, I had a much better understanding of why this problem is harder than it looks, and a system that holds up against real commercial policies. This post is about what I learned along the way - the domain knowledge I had to pick up, the engineering decisions that followed from it, and the problems / challenges that came along the way to build a durable AI pipeline for insurance.



## What I Had to Learn About Commercial Policies

I started without deep insurance domain knowledge. I built up what I needed by reading about commercial policies, looking at real forms, and, more than anything, debugging extraction failures and asking why.

Here is what I think matters for building an extraction system.

### The Five Sections of a Commercial Policy

A commercial insurance policy is organized into five functional sections. Understanding what each one is - and how they relate - shapes every downstream engineering decision.

**Declarations** - The summary page. Named insured, effective / expiration dates, premium, coverage codes, policy number. It's the entry point, but it defers to everything else. When I started, I treated declarations as "metadata." I quickly learned they're also the most reliably structured section, which makes them useful as context anchors for other extraction tasks.

**Coverages** - The insurer's core promises. Each coverage has three numbers that define its scope:
- **Per-occurrence limit**: max payout per single event
- **Aggregate limit**: max payout across the entire policy period
- **Deductible**: how much the insured absorbs or out-of-the-pocket amount the insured pays before coverage activates

**Exclusions** - The takebacks. Two kinds: absolute (coverage eliminated entirely - war, intentional acts, nuclear liability) and conditional (specific scenarios carved out, like pollution exclusions). The tricky part: an exclusion only makes sense relative to the coverage it restricts. You can't extract exclusions in isolation and expect them to be useful.

**Conditions** - Obligations the insured must meet for coverage to activate. Notice requirements, cooperation clauses, subrogation rights. A conditions section looks like legal boilerplate, so I initially deprioritized it. But a missed notice requirement is one of the most common reasons valid claims get denied - conditions matter.

**Endorsements** - Amendments to the base policy. They can expand coverage, restrict it, add exclusions, modify limits. A standard policy might have 10-15. A complex manuscript policy can have 40-60. Each one references and modifies something somewhere else in the document. This is the part that caused me the most engineering pain, which I'll get to in Section 5.

### The Relationship Problem

The thing that changed how I thought about the data structure: a coverage clause is semantically incomplete without the exclusions that apply to it, and an exclusion is semantically incomplete without the endorsements that modify it, and both can be voided by conditions on a completely different page. We can see clearly, there's some relationship between these sections.

The relevant information for any single question - "what does this policy actually cover for pollution-related claims?" - is spread across four or five sections, on pages that may be 100+ pages apart. An extraction system that processes sections independently and concatenates the results will miss the relationships.

That's why the final indexing layer uses Neo4j alongside pgvector to build a knowledge base. Vector search handles semantic lookup ("find coverages similar to professional liability"). Graph traversal handles relational questions ("what exclusions apply to Coverage A?"). Neither store alone answers both kinds of questions. I'll cover how they're connected in Section 6.



## The Pipeline I Designed, and Why

The pipeline has six stages. Each one exists because of a specific problem I ran into with an earlier, simpler approach.


```
PDF Upload
    │
    ▼
[Stage 0] OCR + Layout Extraction
    │  Layout-aware text, tables, word coordinates
    ▼
[Stage 1] Page Classification + Manifest
    │  Section type per page, continuation detection, duplicate filtering
    ▼
[Stage 2] Hybrid Chunking
    │  Section-aware splits, endorsement semantic projection
    ▼
[Stage 3] LLM Extraction
    │  Per-section structured output, synthesis pass
    ▼
[Stage 4] Entity Resolution
    │  Deduplication, canonical keys, cross-batch relationship extraction
    ▼
[Stage 5] Indexing: pgvector + Neo4j + Citations
```

### Stage 0 - OCR: Layout Matters

I initially tried plain text extraction from PDFs. It worked until I hit a premium schedule - a table where the dollar amounts in the cells are completely meaningless without the column headers. "1,000,000" tells you nothing. "Per Occurrence: 1,000,000" tells you the limit.

I switched to Docling, IBM's layout-aware document intelligence library. It understands table structure, preserves reading order across columns, and handles the inconsistent formatting that shows up in real commercial policies - footnotes that modify clause meaning, headers that span columns, sections that start mid-page.

The OCR stage also extracts **word-level bounding boxes** - the physical location of every word on the page in PDF coordinate space:

```python
# Stored in document_pages.additional_metadata["word_coordinates"]
[
    {"t": "Coverage", "x0": 72.0, "y0": 680.5, "x1": 134.2, "y1": 692.0},
    {"t": "A",        "x0": 138.0, "y0": 680.5, "x1": 144.0, "y1": 692.0},
    {"t": "$1,000,000","x0": 310.0, "y0": 680.5, "x1": 398.0, "y1": 692.0},
]
```

These aren't used at this stage. They become essential in Stage 5 for citation resolution - tracing every extracted fact back to its physical location in the source PDF.

### Stage 1 - Page Classification: Rules Over LLMs

Each page needs to be classified by section type: declarations, coverages, exclusions, conditions, endorsements. The classification also needs to detect continuation pages - pages that belong to the prior section even though they carry no heading.

My first instinct was to use an LLM for this. I dropped it after testing. Insurance documents have distinctive typographic patterns: heading styles, numbered clause formats, table density, presence of ISO form codes. Rule-based matching on 200+ regex patterns against those signals handles ~95% of cases with zero token cost. The remaining 5% - ambiguous continuation pages - are scored separately (covered in Problem 1).

Using LLM tokens on classification is a budget problem. A 180-page policy generates 30–40 LLM extraction calls just for the entity extraction stage. Every token spent on page classification is a token not available for extraction. The tradeoff is clear.

The output of this stage is a `PageManifest` - which pages belong to which section types, which are duplicates (common in multi-form policies), and which are boilerplate to skip.

### Stage 2 - Hybrid Chunking: Section Structure + Token Limits

LLMs have finite context windows. A 200-page policy cannot be processed in one call. Fixed-size chunking - cut every N tokens, overlap by M - destroys semantic structure. A chunk that starts mid-coverage clause and ends mid-exclusion list is technically parseable but semantically incoherent.

Chunking in Insura-AI respects section boundaries. Chunks are assembled from the page manifest's section assignments. A coverage section becomes coverage-specific chunks. Exclusion pages become exclusion-specific chunks. The section type travels with each chunk - every downstream extractor knows what kind of content it is processing before it reads a word.

Super-chunks aggregate three individual chunks per LLM call, capped at 6,000 tokens. The stage also handles endorsement semantic projection, which I cover in Problem 1.

### Stage 3 - LLM Extraction: One Schema Per Section Type

Early on I used a generic extraction prompt: "extract all entities from this insurance document section." The results were generic. A coverage section and an exclusion section both produced undifferentiated entity blobs.

The fix was obvious once I'd spent time with the actual document structure: each section type needs its own Pydantic output schema tailored to what that section actually contains.

- Declarations extractor: policy number, named insured, effective dates, premium, coverage codes
- Coverage extractor: coverage name, per-occurrence limit, aggregate limit, deductible, defense costs treatment
- Exclusion extractor: title, description, absolute vs. conditional, carve-outs
- Conditions extractor: condition type, obligation, consequence of non-compliance

An extractor that knows it's looking at a coverages section can ask the right questions. An extractor looking at the same text through a generic schema produces generic answers.

After section-level extraction, a synthesis pass produces `effective_coverages` and `effective_exclusions` - coverage-centric records where endorsement modifications are folded into the base coverage record.

### Stage 4 - Entity Resolution: Same Entity, Different Names

LLMs are inconsistent with entity names. "CGL," "Commercial General Liability," "Comm. Gen. Liability," and "Commercial General Liability Coverage Part" all refer to the same thing. Without deduplication, the downstream graph has four nodes for one entity and no cross-references between them.

Entity resolution normalizes every extracted entity to a canonical key - a 32-character SHA256 hash of `entity_type:normalized_value`:

```python
def generate_canonical_key(entity_type: str, normalized_value: str) -> str:
    key_input = f"{entity_type}:{normalized_value}".lower()
    return hashlib.sha256(key_input.encode()).hexdigest()[:32]
```

The normalized value is produced via a field priority map per entity type followed by slugification. "Commercial General Liability" → `cov_commercial_general_liability`. This key is used as the primary identifier in both PostgreSQL and Neo4j - more on that in Section 6.

### Stage 5 - Indexing: Two Stores

Entity embeddings go into PostgreSQL via the pgvector extension (384-dimensional, `all-MiniLM-L6-v2`). Entity nodes and relationships go into Neo4j.

The reason for two stores: they answer different kinds of questions. "Find coverages similar to professional liability" is a semantic question - pgvector handles it. "What exclusions apply to Coverage A under this policy?" is a relational question - Neo4j handles it in a single Cypher pattern:

```cypher
MATCH (p:Policy {workflow_id: $workflow_id})
      -[:HAS_COVERAGE]->(c:Coverage)
      <-[:EXCLUDES]-(e:Exclusion)
RETURN p.policy_number, c.name, e.title, e.description
```

In SQL that same query requires nested CTEs. The graph database makes the natural structure of insurance data the natural database query.



## Orchestrating Long-Running Work with Temporal

Processing a commercial policy takes 2-10 minutes end-to-end: Docling OCR on potentially 100+ pages, 30-40 LLM extraction calls with rate limit backoffs, Neo4j graph construction across hundreds of canonical entities.

Any stage can fail mid-flight. A provider rate limit during extraction call 22 of 35. A transient database timeout during graph construction. An out-of-memory condition on the worker container during OCR.

To resolve these type of issues and build a more reliable and durable system / pipeline I went ahead with **Temporal**.

### Durable Execution

Every Temporal activity is persisted to Temporal's Event History store before the workflow advances. A worker crash does not lose progress - when the worker recovers, it replays the events present in the history and resumes from the last successfully completed activity. This is deterministic replay, not re-execution. OCR does not run again. Prior LLM calls do not run again.

### Per-Activity Isolation with Retry Policies

Each pipeline stage is a separate Temporal activity with its own timeout and retry configuration:

```python
ocr_data = await workflow.execute_activity(
    "extract_ocr",
    args=[workflow_id, document_id],
    start_to_close_timeout=timedelta(minutes=10),
    retry_policy=RetryPolicy(
        maximum_attempts=5,
        initial_interval=timedelta(seconds=5),
        maximum_interval=timedelta(seconds=60),
        backoff_coefficient=2.0,
    ),
)
```

OCR: 10-minute timeout, 5 retries, exponential backoff. Extraction activities: 30-minute timeout, 3 retries - LLM calls are expensive to retry aggressively. Graph construction: 15 minutes. A failure in extraction does not re-run OCR. Each activity is its own recoverable unit.

### Real-Time Progress Without Polling

Temporal query handlers let external callers inspect a running workflow's state without interrupting it. The API surfaces this as Server-Sent Events - the frontend receives live progress updates as the workflow advances:

```
data: {"stage": "ocr", "status": "completed", "pages_processed": 180}
data: {"stage": "extraction", "status": "running", "batch": 12, "total_batches": 35}
data: {"stage": "indexing", "status": "running", "entities_indexed": 247}
```

No polling. No WebSocket state. The workflow is the source of truth.

A pipeline that cannot partially recover from failure is a batch job. Temporal makes it a durable, inspectable, partially-resumable execution unit. That distinction matters in production.



## Three Problems That Came Along the Way

These weren't abstract engineering challenges. Each one was caused by something specific about how insurance documents are structured.

### Problem 1: Section Boundaries Don't Respect Pages

Here is a pattern that appears in virtually every real commercial policy:

A coverage clause starts mid-page 15. It continues through page 16. At the top of page 17, the clause concludes with a sublimit: "subject to a $100,000 sublimit for any single occurrence involving mold or fungus." No heading. No signal that this is still Coverage A. The page classifier labels page 17 as a coverage page - correct - but page 17 in isolation is semantically incomplete. The sentence that started on page 15 ends here.

The harder version of this problem is endorsements. An endorsement might read: "The following exclusion is added to Coverage A, Section I: This policy does not apply to bodily injury arising out of the use of any aircraft."

If I chunk endorsements as a standalone section and extract them in isolation, I get a list of amendments with no coverage context. The extracted exclusion is technically correct. It has no relationship to the coverage it modifies, because that coverage lived in a different chunk. The entity is orphaned.

**How I solved it - two parts:**

First: a **continuation scoring** system in the page classifier. Each page is scored against signals that suggest it continues the previous section: mid-sentence endings, numbered list items without a header, absence of any section-type heading pattern, structural markers that indicate "inside a clause" rather than "beginning of one." Pages that score high are assigned to the previous section's type rather than classified independently.

Second: **endorsement semantic projection** during chunking. Endorsements are analyzed for their effective semantic role - does this endorsement expand coverage? Restrict it? Add an exclusion? Based on the analysis, the endorsement chunk is projected into the effective section type and processed by the corresponding section extractor, not a generic endorsement extractor.

An endorsement adding an aircraft exclusion to Coverage A now produces an `Exclusion` entity with a `MODIFIES → Coverage A` relationship - correctly typed and correctly connected.

**Where this still falls short:** Semantic projection is heuristic. I analyze endorsement text for signals like "coverage is added," "the following exclusion is added," "this policy does not apply." This handles approximately 85% of endorsements correctly. Endorsements that both expand and restrict the same coverage in the same clause - which do exist - are ambiguous and sometimes misclassified. A fine-tuned classifier trained on labeled endorsement data would handle these more reliably.

### Problem 2: Entities in Different Extraction Batches Can't See Each Other

A 100-page policy produces ~30 semantic chunks, processed in batches of three. Coverage entities are extracted in batches 2-8. Exclusion entities are extracted in batches 9-15. The relationship between them - "Coverage A EXCLUDES Pollution" - requires seeing both entities in the same context. They never appear in the same LLM call. The relationship is lost.

Increasing batch size helps at the margins but does not fix the core problem. A 200-page policy with 40+ sections will always have entities that span extraction batches regardless of batch size configuration.

**How I solved it - three tiers:**

**Tier 1 (Strategic Overlap):** Declarations - the section that defines the top-level policy entity everything else relates to - are included as shared context in multiple batches. Every batch can see the policy-level entities. This captures ~70-80% of cross-batch relationships (the ones where one side is always the Policy node).

**Tier 2 (Enhanced Prompts):** Each extraction prompt includes explicit cross-section relationship patterns with concrete examples - `Policy HAS_COVERAGE Coverage`, `Coverage EXCLUDES Exclusion`, `Endorsement MODIFIES Coverage`. The LLM is instructed to identify these even when the related entity isn't in the current batch.

**Tier 3 (Cross-Batch Synthesis Pass):** After all semantic batches complete, a single focused LLM call sees all extracted entities organized by type and all existing relationships grouped by batch. Its only job: identify relationships that individual batches missed.

```python
# After all semantic batches complete:
entities_by_type = {}
for entity in canonical_entities:
    entity_type = entity.get("entity_type", "Unknown")
    entities_by_type.setdefault(entity_type, []).append(entity)

relationships_by_batch = {}
for rel in existing_relationships:
    batch_name = rel.get("attributes", {}).get("extraction_batch", "unknown")
    relationships_by_batch.setdefault(batch_name, []).append(rel)

cross_batch_relationships = await self._cross_batch_synthesis_pass(
    context=context,
    existing_relationships=batch_relationships,
    semantic_batches=semantic_batches,
)
```

Synthesis relationships are tagged with `extraction_batch="cross_batch_synthesis"` and merged before deduplication. Relationship completeness goes from ~70–80% (batches alone) to ~95%+ with the synthesis pass. Token overhead: +15–20%.

The reason this is worth the cost: a coverage with missing exclusions isn't an incomplete data record - it's a record that will cause an underwriter to incorrectly assess the risk. The extra LLM call is cheap compared to that.

### Problem 3: Citations Break When the LLM Paraphrases

Every extracted entity needs to trace back to its source in the PDF. For an underwriter reviewing AI-extracted data, the citation is the verification mechanism. "Per-occurrence limit: 1,000,000" is useful. "Per-occurrence limit: $1,000,000 [Coverage A, Page 12, Line 7]" is auditable.

The naive approach: take the extracted text, find it in the word coordinates from OCR. This works when the LLM copies the source clause verbatim. LLMs almost never do.

Source text: *"We will pay those sums that the insured becomes legally obligated to pay as damages because of 'bodily injury' or 'property damage' to which this insurance applies."*

LLM extraction: *"Coverage for bodily injury and property damage, including defense costs."*

Same meaning. Zero word overlap. Direct text match fails completely.

**How I solved it - three tiers:**

**Tier 1 (Direct Text Match):** Sliding-window fuzzy matching against word-level coordinates from OCR. Fuzzy threshold: 0.75. This works when the LLM stays close to source language - about 85% of citations.

**Tier 2 (Semantic Chunk Search):** When direct matching fails, the extracted text is embedded using the same `all-MiniLM-L6-v2` model used for entity embeddings. This embedding is cosine-searched against a chunk-level embedding index - each source chunk has been independently embedded during the indexing stage. The nearest chunk by cosine distance (threshold: 0.65) identifies the most likely source page. The citation mapper retries on that page with a lower fuzzy threshold (0.65), sufficient when the source page is already known.

**Tier 3 (Placeholder):** If both tiers fail, a full-page bounding box is used as a placeholder citation. This ensures 100% citation coverage - no extracted entity goes without at least page-level traceability. The `resolution_method` field on each citation record (`direct_text_match`, `semantic_chunk_match`, `placeholder`) tells downstream consumers how confident the citation is.

End result: ~95% of citations are Tier-1 or Tier-2 resolved. The 5% that fall through to placeholders are typically highly paraphrased synthesis outputs where the source text is genuinely distributed across multiple pages.



## Connecting Two Data Stores Without Glue Code

Entity embeddings live in PostgreSQL via pgvector. Entity nodes and relationships live in Neo4j. GraphRAG queries need to: (1) find semantically similar entities via vector search, (2) expand results by traversing graph relationships from those entities.

Vector search returns pgvector row IDs. Graph traversal needs Neo4j node identifiers. These are different identifiers from different stores. Without an explicit bridge, step 2 requires an application-level join that doesn't exist.

The bridge is the `canonical_entity_id` - the SHA256 key generated during entity resolution. It appears in four places:
- Primary key of `CanonicalEntity` in PostgreSQL
- Foreign key on `VectorEmbedding`: `vector_embeddings.canonical_entity_id → canonical_entities.id`
- Property on every Neo4j entity node: `n.canonical_id = canonical_entity_id`
- `vector_entity_ids` list property on Neo4j nodes (list of pgvector row IDs for all embeddings of this entity)

At query time:
```
Vector search → VectorEmbedding rows
                     │
                     │ .canonical_entity_id
                     ▼
               CanonicalEntity record
                     │
                     │ canonical_id property
                     ▼
               Neo4j entity node
                     │
                     │ graph traversal
                     ▼
               Related entities (Exclusions, Conditions, Endorsements...)
```

`HAS_EMBEDDING` edges in Neo4j make this bridge explicit in the graph itself, enabling pure-Cypher queries that traverse from entity to its embeddings and back.

The `canonical_key.py` utility is the single source of truth. Both the entity resolver (Stage 4) and the embedding generator (Stage 5) call the same function:

```python
def generate_canonical_key(entity_type: str, normalized_value: str) -> str:
    key_input = f"{entity_type}:{normalized_value}".lower()
    return hashlib.sha256(key_input.encode()).hexdigest()[:32]
```

One function, two call sites, consistent keys across both stores. Without this, the two retrieval systems are unconnected - GraphRAG becomes two separate queries with a manual join in application code.



## What I Think Is Still Genuinely Hard

Building Insura-AI clarified for me the difference between problems that are *engineered around* and problems that are *actually solved*. Several things in the current system are engineered around.

**Knowing When the Extraction Is Wrong**

The pipeline produces extractions. It doesn't know when it's guessing. A coverage with a missing per-occurrence limit - common in umbrella policies that defer to the underlying policy's limit - looks identical in the output to a coverage with a genuine $0 limit. The LLM fills in what it can; when it can't, it either hallucinates a value or omits the field silently.

A production tool needs a reliable signal for "I'm not sure about this." Confidence scores are easy to produce and hard to calibrate. Ambiguity detection - detecting when the document itself is unclear, when two model passes give different answers, or when a required field simply has no value to extract - is a more useful signal and significantly harder to build reliably.

**Entity Linking Across Documents**

Within a single policy, entity resolution works well. Across documents - a quote versus a binder versus a final policy, or two policies covering the same property from different carriers - entity linking becomes much harder.

Location entity in document A: "123 Main Street."
Location entity in document B: "123 Main St., Suite A."

Same building? Different units? A typo? The answer changes whether these two policies together constitute adequate coverage. At portfolio scale - hundreds of documents per account - this problem compounds rapidly.

**Learning From Human Corrections**

The current system has no feedback loop. A user who corrects an extracted coverage limit improves the UI state for their session. The correction does not update the canonical entity store. The knowledge graph is not updated. The pipeline does not learn.

Building a system that improves from human corrections without regressing what already works requires rethinking extraction as a loop, not a sequence. That's what Section 8 is about.



## Where This Could Go: Agentic Extraction Loop

The current pipeline is deterministic: same PDF in, same extracted entities out. That's a useful baseline. It's not a production underwriting tool.

An underwriter reviewing AI-extracted data wants to correct results, flag ambiguous extractions for human review, and have those corrections feed back into the system. The current architecture can support this with a relatively small set of additions - specifically, Temporal signals and queries extended into an interactive interface between the pipeline and the human operator.

### Human-in-the-Loop via Temporal Signals

After the extraction stage, instead of immediately proceeding to indexing, the workflow evaluates confidence across extracted entities. If any entity falls below a confidence threshold, it becomes a review item and the workflow suspends:

```python
@workflow.signal
def submit_correction(self, correction: ExtractionCorrection) -> None:
    self._pending_correction = correction
    self._correction_received = True

@workflow.query
def get_review_items(self) -> list[LowConfidenceItem]:
    return self._low_confidence_items

# Inside the workflow:
if low_confidence_items:
    self._low_confidence_items = low_confidence_items
    await workflow.wait_condition(
        lambda: self._correction_received,
        timeout=timedelta(hours=24),
    )
    extraction_output = merge_correction(
        extraction_output, self._pending_correction
    )
    # Workflow resumes and proceeds to indexing with corrected data
```

The workflow suspends durably - no resources consumed, survives worker restarts, no timeout risk for 24 hours. The frontend polls `get_review_items()` to surface review items. When the user submits a correction, `submit_correction()` resumes the workflow with the updated data. Indexing proceeds with human-validated extractions.

This is three additions to the existing workflow: a signal handler, a query handler, and a `wait_condition` call between extraction and indexing. The durable execution model that already handles failures handles the human pause for free.

### Self-Correction Loop

A natural extension: run two extraction passes on ambiguous sections with different prompt strategies. If both agree on a coverage entity's limits, confidence is high. If they disagree, that disagreement is itself a signal worth surfacing.

```
Extraction Pass A ──┐
                    ├── ValidationActivity ──→ Agreement? → Indexing
Extraction Pass B ──┘                     └── Disagreement? → wait_condition (human review)
```

Every correction the human makes is a durable workflow event. Over time, these corrections accumulate into a labeled dataset: the cases where the system was uncertain, the cases where it was confidently wrong, and the corrections that resolved each one. That's the training data needed to reduce the review burden over time - not as a designed data collection pipeline, but as a natural output of the extraction loop.

### Why Temporal Is the Right Foundation for This

Most pipeline orchestration treats durability as an afterthought. Temporal treats it as the primary concern. The workflow history is an automatic audit trail - every activity, every signal, every correction is a durable event. Debugging is deterministic replay. For regulated insurance workflows, that matters: a binding quote or claim decision needs a record of what data was used, what the AI extracted, and what a human corrected. Temporal provides this by construction, not as a feature someone has to build.

The jump from extraction pipeline to agentic extraction loop doesn't require rethinking the architecture. Temporal's signals and queries are already the interface between an autonomous pipeline and an external actor - whether that actor is a frontend API or a human reviewer.



## Closing Thoughts

I went into this project knowing how to build pipelines and knowing very little about commercial insurance. Two months later the proportion is different.

The engineering decisions I'm most confident about - section-aware chunking, the Temporal orchestration layer, the canonical key bridging two data stores - all came from learning something specific about the domain and following the implication. Section-aware chunking because endorsements modify coverages on different pages. Two data stores because insurance queries are both semantic and relational. Temporal because a 10-minute pipeline on shared infrastructure will fail. Canonical keys because the same entity appears under different names across extraction calls.

What I built is a capable document processing pipeline. What I'm less certain about is how much of the hard problem - ambiguity detection, learning from corrections, cross-document entity linking - can be solved by better engineering versus requiring something closer to genuine domain expertise baked into the model. I suspect the answer is both, and the interesting work is in the interface between them.



*Source code for Insura-AI is available on [GitHub](https://github.com/Shreekar11/insura-ai). Built with FastAPI, Temporal, Typescript, Next.js, PostgreSQL/pgvector, Neo4j, Docling, and Sentence Transformers.*
