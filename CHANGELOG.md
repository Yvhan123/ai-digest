Changelog

agents.py

ArticleSummary dataclass — simplified

Removed deprecated fields (key\_points, why\_this\_matters, technical\_specs, industry\_impact). The dataclass now has three fields: article\_id, summary, and suggested\_subject.

Collector — category-specific Tavily queries

Each active category now has a dedicated search query instead of a generic fallback. Queries are written to target the right kind of content per category and explicitly exclude content that belongs in other categories.

Collector — category-specific search parameters

ai\_research and ai\_research\_arxiv use topic: "news" to bias toward recent publications. ai\_research\_arxiv additionally pins results to arxiv.org via include\_domains. genai\_tips uses topic: "general" since quality practitioner content is not always breaking news. ai\_trends and ai\_innovations use topic: "news".

New category: ai\_research\_arxiv

Locks collection entirely to arXiv at every layer — Tavily include\_domains, deep research ARXIV\_ONLY\_FEEDS, a hard URL filter removing any non-arXiv entry that slips through, and an evaluator rule that rejects immediately if the URL does not contain arxiv.org.

Category consolidation



ai\_technology absorbed into ai\_innovations

tools\_updates absorbed into ai\_trends

policy\_ethics removed entirely

Active categories: ai\_trends, genai\_tips, ai\_innovations, ai\_research, ai\_research\_arxiv



Quality evaluator — per-category guidance rewritten

Each category now has precise accept/reject criteria tailored to its purpose and audience:



ai\_innovations: two equal pillars — capability breakthroughs and major industry releases, both scored on concrete technical detail

ai\_trends: requires evidence of genuine traction (adoption data, multiple organisations, widespread coverage); rejects single-company announcements

genai\_tips: calibrated for data scientists and AI practitioners; rejects beginner/consumer-facing content

ai\_research: accepts only publications and preprints; rejects all product and company news

ai\_research\_arxiv: rejects anything whose URL is not from arxiv.org



Summarizer — research-aware branching

ai\_research and ai\_research\_arxiv use a dedicated prompt that prescribes sentence structure (problem + method in sentence 1; concrete result or implication with numbers in sentence 2), preserves technical terms that carry meaning, and bans filler openers. General categories use a plain-English prompt focused on what happened and why it matters.

Composer — assembly only

Temperature lowered from 0.4 to 0.2. The prompt explicitly instructs the model to copy headlines and summaries verbatim and not rewrite them. The composer's role is layout and structure, not creative rewriting.

Standardizer — trim-only pass

Now explicitly forbidden from adding words or expanding summaries. Has a prioritised trim checklist: filler openers first, then redundant qualifiers, then wordy phrases, then as a last resort shortening a summary sentence. Temperature lowered to 0.1.

New function: generate\_digest\_headline()

Given the final list of composer items, generates a single teaser sentence that previews all stories without repeating any headline verbatim. Used as the masthead tagline in the HTML output.



pipeline.py

Parallel quality evaluation

All articles within a category are now evaluated by the Quality Agent concurrently using ThreadPoolExecutor(max\_workers=5). Previously sequential.

Parallel summarisation

All accepted articles are summarised concurrently under the same 5-worker cap. Previously sequential.

Parallel image collection

Images for all selected articles are fetched simultaneously instead of one at a time. For 3 articles this reduces image collection from \~30s to \~10s.

Parallel \_collect\_both

When collector\_type = "both", Tavily and Deep Research now run simultaneously in a 2-worker pool instead of sequentially.

max\_pool parameter

run\_collection\_pipeline accepts a max\_pool argument that controls how many articles are passed to the quality evaluator independently of --max-results. Exposed via --max-pool on the CLI.

Quota exhaustion handler

\_is\_quota\_exhausted() detects OpenAI insufficient\_quota errors and stops the pipeline immediately with a clear message pointing to the billing page, instead of silently logging a per-article error and continuing.

HTML output

After every compose run, a self-contained .html file is written to output/ alongside the existing .md file. Images are embedded as base64 strings so the file is fully portable. Generated via render\_newsletter\_html() in formatter.py and saved via save\_newsletter\_html() in storage.py.

Story deduplication

\_deduplicate\_by\_story() runs after quality sorting and before the final \[:max\_items] slice. Uses two independent signals:



Named entity overlap (primary, threshold 0.45) — extracts capitalised tokens and bigrams; catches the same story covered from different angles (e.g. three articles about the same model launch written from safety, capability, and business perspectives all share the same company and product names)

Text similarity (secondary, threshold 0.38) — combined title + summary word fingerprint; backstop for near-identical rewrites

Research categories (ai\_research, ai\_research\_arxiv) skip the entity signal and use text similarity only, since all papers share arxiv as an entity which would cause false positives across different papers.



generate\_digest\_headline wired in

Called after the standardizer with the final composer\_items list. Result passed to render\_newsletter\_html() as digest\_headline.



deep\_research.py

Parallel RSS feed fetching

All 30+ feeds are now fetched simultaneously using ThreadPoolExecutor(max\_workers=20). Previously sequential. Worst-case time drops from \~750s to \~25s. Feed order is preserved in the result so deduplication remains deterministic.

Expanded arXiv feeds (3 → 9)

Added cs.CL (NLP/LLMs), cs.CV (Computer Vision), cs.NE (Neural Computing), cs.RO (Robotics), cs.IR (Information Retrieval / RAG), eess.SP (Signal Processing / speech AI).

ARXIV\_ONLY\_FEEDS constant

A dedicated feed list used exclusively when category == "ai\_research\_arxiv". Bypasses DEFAULT\_AI\_FEEDS and user-configured feeds entirely.

\_sort\_arxiv\_first()

New sort function used for ai\_research and ai\_research\_arxiv that places all arXiv entries (matched by arxiv.org in the URL) at the top sorted by date, followed by everything else. Prevents arXiv papers from being buried behind general tech news.

Hard URL filter for ai\_research\_arxiv

After keyword filtering, entries whose URL does not contain arxiv.org are removed as a final backstop before articles are passed to the evaluator.

max\_results pool cap removed

The previous hardcoded min(max\_results \* 2, 30) cap is gone. max\_results is now passed directly to the collector, giving the user full control via --max-results and --max-pool.

CATEGORY\_KEYWORDS updated



ai\_trends: expanded with tool/framework adoption signals absorbed from the removed tools\_updates category

ai\_innovations: rewritten around capability and model release terms plus major lab names; previously overlapped too heavily with ai\_research keywords

ai\_research\_arxiv: empty list (URL filtering handles source restriction)

ai\_technology, tools\_updates, policy\_ethics: removed





formatter.py (new file)

Renders newsletter cards as a self-contained HTML string using a Jinja2 template. Accepts a list of card items (title, summary, url, image\_path), converts each image to base64, and passes the enriched data to the template. Also accepts an optional digest\_headline for the masthead teaser.



storage.py

save\_newsletter\_html() — new function that saves the rendered HTML newsletter to output/ as a self-contained .html file alongside the existing .md output.



config.py

Default categories updated to ai\_trends,genai\_tips,ai\_innovations,ai\_research,ai\_research\_arxiv. Removed ai\_technology, tools\_updates, and policy\_ethics.



run\_pipeline.py

\--max-pool argument added to both collect and collect-and-compose subcommands. Controls how many articles are passed to the quality evaluator independently of --max-results.

ai\_research\_arxiv added to help text for the --category argument.



ai\_digest/templates/newsletter\_card.html (new file)

Email-safe Jinja2 template using table-based layout (no flexbox — stripped by Gmail/Outlook). All styles are inlined. Design: white cards with an orange top-border accent (#e85d04), orange masthead banner containing "AI Digest", the run date, and the generated teaser headline. Responsive stacking on narrow screens. Footer includes a pilot feedback section with a link to the Microsoft Forms survey.

