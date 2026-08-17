# Memory vs No-Memory Comparison

## Overall Summary

| Metric | Memory-enabled | No-memory | Delta |
| --- | ---: | ---: | ---: |
| **Evidence hit rate** | 100.0% | 18.2% | **+81.8%** |
| Passed cases | 11/11 | 2/11 | **+9** |
| Avg retrieval latency (ms) | 1888.0 | 0.0 | +1888.0 |
| Avg token reduction | 15.0% | 81.8% | -66.8% |

---

## By Layer Breakdown (Practice Set)

| Layer | Memory | No-Memory | Delta |
| --- | ---: | ---: | ---: |
| short_term | 2/2 (100%) | 2/2 (100%) | 0% |
| long_term | 4/4 (100%) | 0/4 (0%) | **+100%** |
| episodic | 2/2 (100%) | 0/2 (0%) | **+100%** |
| semantic | 2/2 (100%) | 0/2 (0%) | **+100%** |
| mixed | 1/1 (100%) | 0/1 (0%) | **+100%** |

**Observation:** Short-term cases pass without memory because evidence is in current thread. All durable memory layers fail without memory.

---

## Token Reduction Analysis

| Implementation | Avg Token Reduction | Interpretation |
| --- | ---: | --- |
| Memory-enabled | 15.0% | Selective retrieval, evidence preserved |
| No-memory | 81.8% | Retrieves nothing = 0 tokens = 100% reduction |

### Why High Token Reduction ≠ Good

```
No-memory: 0 tokens retrieved = 81.8% reduction = 18.2% hit rate
Memory: 485 tokens retrieved = 15.0% reduction = 100% hit rate

Conclusion: Token reduction is meaningful ONLY when hit rate is high.
```

**Key insight:** No-memory baseline appears to "save tokens" but actually retrieves nothing. This is not optimization — it's information loss.

---

## Latency Breakdown

| Layer | Avg Latency (ms) | Reason |
| --- | ---: | --- |
| short_term | 0.3 | Local, no network |
| long_term | 3155.4 | Zep API + Context Block processing |
| episodic | 465.2 | Zep API + graph search |
| semantic | 1965.3 | Zep API + standalone graph |
| mixed | 3285.0 | Multiple API calls + assembly |

---

## No-Memory Failures Explained

| Case | Layer | Evidence Missing | Reason |
| --- | --- | --- | --- |
| E02 | long_term | Python | Context Block not retrieved |
| E03 | long_term | benchmark report, 16:00 | Open loop not in current thread |
| E04 | episodic | ClientSession, ASYNC-FIX-20 | Past session not accessible |
| E05 | episodic | connection churn | Reflection from previous session |
| E06 | semantic | Idempotency-Key | Shared KB not loaded |
| E07 | mixed | Python + Idempotency-Key | Both layers require memory |
| E08 | long_term | TypeScript, NestJS | Cross-session update missing |
| E09 | long_term | Java, Spring Boot | User isolation not maintained |
| E11 | semantic | CONN-POOL-FIRST | Playbook not in context |

---

## Golden Set: Memory Impact

| Metric | Memory | No-Memory (estimated) | Delta |
| --- | ---: | ---: | ---: |
| Evidence hit rate | 100.0% | ~5% | +95% |
| Passed cases | 20/20 | ~1/20 | +19 |
| Avg latency (ms) | 1049.4 | 0.0 | +1049.4 |
| Token reduction | 6.3% | ~90% | -83.7% |

**Golden insight:** Long noisy prompts (3-5x practice length) don't affect retrieval because evidence is stored as markers in memory layers, not derived from query matching.

---

## Conclusion

### Why Memory Matters

1. **Cross-session recall:** Without memory, only current thread is accessible (E01, E10 pass; everything else fails)

2. **User isolation:** Memory systems with user_id scoping prevent data leakage between users

3. **Recency handling:** Context Block automatically manages fact overrides based on timestamp and scope

4. **Trajectory preservation:** Episodic memory captures not just facts but action→outcome sequences

5. **Shared knowledge:** Semantic graphs provide domain-wide information without per-user duplication

### When No-Memory Works

| Case Type | Why It Works |
| --- | --- |
| In-thread questions | Evidence still in conversation |
| Recent context | Within token window |
| Single-turn queries | No history needed |

### When Memory Is Required

| Case Type | Examples |
| --- | --- |
| Cross-session recall | E02-E03, G03-G09 |
| Past trajectories | E04-E05, G10-G11 |
| Domain knowledge | E06, E11, G12-G15 |
| Mixed requirements | E07, G16-G20 |
| User-specific context | E08-E09, G08-G09, G19 |

---

## Key Takeaways

1. **Token reduction is a vanity metric** without evidence hit rate. 81.8% reduction with 18.2% hit rate = failure.

2. **Memory latency is acceptable trade-off.** 1-3 seconds for accurate retrieval vs 0 seconds for nothing.

3. **Layer specialization works.** Each layer handles its domain: short-term (local) → long-term (preferences) → episodic (trajectories) → semantic (shared KB).

4. **User isolation is not optional.** User-scoped graphs are essential for privacy and correctness.

5. **Mixed queries require assembly.** Token budget with priority ensures critical evidence survives.

---

*Generated: 2026-08-17*
