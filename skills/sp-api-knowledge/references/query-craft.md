# Query Craft: searching the SP-API knowledge base well

The only knob you control on `knowledgeTools_searchSpApiDocs` is the **query string**. Retrieval quality
depends almost entirely on phrasing. These patterns come from the RICO/Neptune agent that has
run this KB in production.

## Rules
1. **Natural-language questions, not keyword fragments.** The index is tuned for well-formed
   questions.
   - Good: `"How do I subscribe to order change notifications?"`
   - Weak: `"order notification subscribe"`
2. **Keep each query under ~30 words.** Long queries dilute relevance.
3. **Preserve technical identifiers exactly**, operation names, parameters, error codes,
   capitalization (`getOrders`, `x-amzn-RateLimit-Limit`, `QuotaExceeded`). Don't paraphrase
   an identifier.
4. **One intent per query.** If the developer asked a multi-part question, search each part
   separately rather than cramming them together.
5. **Spend the budget on distinct phrasings, not pagination.** If the first phrasing is thin,
   reformulate 3-5 times with different angles before concluding the docs lack the topic:
   - the concept: `"How does the Feeds API upload flow work?"`
   - the operation: `"createFeedDocument request and response"`
   - the section: browse the section via `knowledgeTools_browseSpApiSection`
   - the symptom: `"Why does my feed stay in IN_PROGRESS status?"`

## Choosing the approach by intent
- **Concept / how-it-works** -> search, then `knowledgeTools_getSpApiDocuments` the top hits and synthesize.
- **Getting started / onboarding / registration** -> `knowledgeTools_browseSpApiSection` first (no search).
- **"What API should I use for <use case>"** -> search the use case, map to APIs, decision table.
- **Exact API contract / schema** -> `knowledgeTools_getSpApiOperation` by operationId, or search the schema doc.
- **Does-X-support / availability** -> identify dimensions (marketplace, API, feature variant),
  search each, present the breakdown; don't force a single yes/no when support is partial.

## Triaging results
- **Rank by `confidence`.** Anchor the answer on VERY_HIGH / HIGH; use MEDIUM / LOW only as
  leads to verify by fetching the doc.
- **Read before you answer.** Snippets are for triage; call `knowledgeTools_getSpApiDocuments` on the chosen
  `documentId`s and answer from the full content, with the `url` as the citation.
- **When searches disagree,** present the relevant findings rather than silently picking one.

## When the docs genuinely don't cover it
After 3-5 varied searches with no good hit: state what's missing, give any partial guidance
the docs do support, point to the official resources (see
[design-and-code.md](design-and-code.md)), and **do not** fill the gap from general knowledge.
