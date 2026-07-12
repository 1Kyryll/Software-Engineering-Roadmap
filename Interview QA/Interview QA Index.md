---
tags:
  - interview-qa
---

# Interview Q&A Index

Practice questions with my original answers, a review of what was right/wrong, and a complete spoken answer. Format per note: **Question → My original answer → Review → Say this in the interview** (the last section is a full, self-contained answer to rehearse verbatim).

Up: [[Home]]

## Questions
- [[QA - Cache Eviction Policy for Sessions vs Feed]] — ✅ answered, mostly right; terminology + LFU-decay nuance corrected
- [[QA - Indexing Random UUIDs]] — ⚠️ answered, contained a real error (O(n) claim) — corrected
- [[QA - Paginating 50 Million Rows]] — ✅ answered, right idea; complexity claims tightened
- [[QA - Composite vs Separate Indexes]] — was open, now answered
- [[QA - N Plus One Query Problem]] — was open, now answered
- [[QA - Preventing Double Booking]] — ⚠️ answered, led with distributed locks; corrected to DB-first (my own ADR 001!)

## Connects to
- [[Data Index]] — indexes, pagination, UUIDs all live there in depth
- [[Caching Index]] — eviction policies in depth
- [[System Design MOC]]
