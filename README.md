# AI Mentor Bootcamp — kerla sai tejaswi
<img width="518" height="209" alt="image" src="https://github.com/user-attachments/assets/9b90735f-9046-4148-8b94-367b776092da" />


## Day 4 — Productivity sprint

**Company:** <COMPANY>
**Time:** 45 minutes (timeboxed)

### Edit notes (3 lines)

1. Gamma confabulated a "hiring 50,000 freshers in 2025" stat on slide 6. Source said 40,000. Edited.
2. Slide 4 listed "Kubernetes" as a required skill — actually nice-to-have per the JD. Edited.
3. Slide 1 (cover) — replaced Gamma's generic "Your Career Awaits" with a company-specific line.


## Day 5 Lab 5B — Hugging Face Pulls

### Models tested
- `facebook/bart-large-mnli` — zero-shot classification
- `distilbert-base-uncased-finetuned-sst-2-english` — sentiment

### Timing comparison

| | min | avg | Notes |
|---|-----|-----|-------|
| HF Inference API | 0.8s | 1.2s | Cold-start: 20s |
| Local in Colab | 2.1s | 3.4s | Download: 60s on first run |

### When to use each (3-line reflection)

1. **API:** for low-volume, occasional calls. Avoids download. Cold-start risk on first call after idle.
2. **Local:** for batch processing 100+ items, where you want predictable latency and don't pay per call.
3. **Production rule of thumb:** if your usage exceeds the API free tier (~30K requests/month at HF), self-host. Otherwise API.


## Day 6 Lab 6A — Errors handled

1. **Markdown fence wrapping** (`\`\`\`json ... \`\`\``)
   The retry prompt asks Gemini to output raw JSON without fences. Triggers on ~5-10% of calls.

2. **Hallucinated phone number when source has none**
   `Optional[str] = None` in Pydantic — model returns `null`, schema validates.

3. **Empty / whitespace-only input**
   Pydantic raises ValidationError with "Field required". Caller catches.

**Hallucination on garbage input:** Gemini sometimes invents a plausible résumé from non-résumé text. Defence: validate input before sending (e.g., minimum length, presence of email-like pattern).


## Day 6 — Capstone Sprint 1: PlacementDataProcessor

### Engineer Answer

1. **PROBLEM** — JDs from Naukri / LinkedIn are messy text — placement cells need structured data to filter ("which JDs want Java + CGPA 7+?"). Manual extraction is unscalable for 50+ JDs.

2. **ARCHITECTURE** — JD URL → BeautifulSoup scraper (extract clean text) → Gemini structured-output call (response_schema=JD Pydantic) → JSON Lines file. Validation at each step; retry on schema fail.

3. **TRADE-OFFS** —
   - Cost: free Gemini ~1 JD/sec on average; ~30K tokens/day quota → ~5K JDs/day.
   - Accuracy: Pydantic catches schema violations but not semantic errors (e.g., model says skill is "Python" when source says "Python 3.12 specifically").
   - Latency: ~2-5s per JD (Gemini call dominant).
   - Complexity: scraping fragile (sites block automation). Cached fallback is mandatory.

4. **SCALE** —
   - 10 JDs/day: trivial. Today's lab.
   - 100 JDs/day: still in free quota. Add overnight batch + sleep between calls.
   - 10K JDs/day: free tier breaks. Move to paid Gemini OR self-host an open model.

5. **INTERVIEW ANSWER** — "I built a structured-output pipeline that turns scraped JDs into clean filterable JSON, using free Gemini and Pydantic. Schema-first design with retry-on-failure made it production-shaped on a free-tier API."

### Files
- `Day6_PlacementProcessor.ipynb` — the notebook
- `data/jds.jsonl` — output of this sprint, input for Day 7 RAG

### Pair: <Mentor 1 name> + <Mentor 2 name>

## Day 7 Lab 7A — ChromaDB Hello-World

- Embedded 10 CSE Sem 5 paragraphs with all-MiniLM-L6-v2 (384-dim, free)
- Indexed in persistent ChromaDB collection `hello_syllabus`
- Ran 3 semantic queries — observed: top-1 match is relevant when query topic is in corpus, irrelevant when not
- Plotted PCA 2D — visible OS / DBMS clusters

**Reflection:** Semantic search returns nearest, not exact. RAG must enforce citations to catch out-of-corpus queries (this afternoon's Sprint 2).


## Day 9 Lab 9A — Trace as a story

1. **Human asked:** "What is TCS's 2026 hiring quota?"
2. **Agent thought:** "I don't know recent figures. I should search."
3. **Agent acted:** called `web_search('TCS 2026 hiring quota')`.
4. **Agent observed:** got back search results mentioning 40-50K range.
5. **Agent answered:** synthesised "Based on search results, TCS plans to hire 40-50K freshers..."

This is the ReAct loop. Every agent we build follows this pattern.

## Day 9 Lab 9A — Hello-LangGraph

- 1-tool ReAct agent with DuckDuckGo web_search
- 4-message trace on a live-fact question (TCS 2026 hiring)
- Failure case: bad URL → agent reported "could not find" / agent hallucinated [pick one]

### Reflection (3 lines)

1. The trace IS the explanation. Print every step.
2. The doc-string IS the prompt. Bad doc-string = bad tool selection.
3. Real agents handle tool failures gracefully — define failure modes in the doc-string.

## Day 9 — Capstone Sprint 4: Career Agent

### 3 tools wired
1. **jd_fetcher** — wraps Day 6's fetch_jd. Returns clean text or ERROR string.
2. **skills_gap** — pure function set difference. Deterministic.
3. **answer_scorer** — Gemini-backed scoring 1-10 with rationale.

### 3 successful runs
| # | Student | Tools used | Outcome |
|---|---------|-----------|---------|
| 1 | Ravi Kumar (CSE) → TCS | skills_gap, answer_scorer | Skill-gap: Spring Boot, AWS |
| 2 | Sneha Reddy (ECE) → Cognizant | skills_gap | Strong match — focus on interview practice |
| 3 | Arun Pillai (IT) → Amazon | skills_gap, answer_scorer | Strong match — score 8/10 on sample |

### 1 failure-recovery analysis

Bad URL passed to jd_fetcher. Agent received `ERROR:` from tool. Agent correctly responded:
"I could not fetch the JD URL. Please provide a working URL." No hallucinated JD content.
This is the safe behaviour. If the agent had hallucinated, the fix would be to tighten the doc-string of jd_fetcher.

### Engineer Answer

1. **PROBLEM** — A static RAG cannot take actions. Students need an assistant that fetches JDs, computes skill gaps, and evaluates their answers — autonomously.

2. **ARCHITECTURE** — LangGraph ReAct agent with 3 specialised tools. Each tool is a plain Python function with a precise doc-string. Agent reasons about which tool to call.

3. **TRADE-OFFS** —
   - Cost: 5-15 LLM calls per task (each ~1-3K tokens). ~20K tokens per student session.
   - Latency: 5-10s per task (LLM calls dominant).
   - Reliability: tools must return predictable strings. ERROR returns are part of the contract.
   - Complexity: doc-strings are now part of the prompt. Bad doc-string = wrong tool.

4. **SCALE** —
   - 1 student / minute: free quota OK.
   - 50 students / day: hits free quota. Switch to paid.
   - 1K students / day: needs caching + parallel inference.

5. **INTERVIEW ANSWER** — "I built a 3-tool LangGraph agent that takes a student profile and produces tailored placement prep — JD analysis, skill gap, answer scoring. Each tool is a plain function; the agent picks which to call. Failure recovery is built into tool contracts."
