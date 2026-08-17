# Golden Evaluation Analysis - Lab 17

## Summary

| Metric | Value |
|--------|-------|
| Total Cases | 20 |
| Passed | 20 |
| Hit Rate | 100% |
| Avg Latency | 1049.4 ms |
| Avg Token Reduction | 6.3% |
| Golden Points | **10/10** |

---

## Case-by-Case Analysis

### G01: Short-term Memory - Hard Compaction

**Query (paraphrased):** "Which constraints still valid after 12 filler messages?"

**Evidence Required:** `["HOLD-ALPHA-0900", "HOLD-BETA-STAGING", "09:00"]`

**Result:** PASS (0.3 ms)

**Memory Technique:**
- **Sliding Window Strategy** với compaction
- **Durable Notes Pattern Detection**

**Tại sao pass:**
```
Input: 16 messages (2 constraints + 12 fillers + 2 acknowledgments)
       ↓
ShortTermMemory.compact() được gọi 10 lần
       ↓
Old messages evicted nhưng constraints được giữ lại trong durable_notes
       ↓
Output: <DURABLE_NOTES>
        - user: Constraint HOLD-ALPHA-0900: standup is 09:00 sharp...
        - user: Constraint HOLD-BETA-STAGING: writes go to staging DB only.
       </DURABLE_NOTES>
```

**Key pattern detection:**
```python
DURABLE_PATTERNS = ("todo", "deadline", "constraint", "HOLD-...")
# Constraint HOLD-ALPHA-0900 match vì:
# 1. Chứa "constraint" trong content
# 2. Chứa uppercase marker pattern r"\b[A-Z][A-Z0-9-]{5,}\b"
```

---

### G02: Short-term Memory - In-thread Recall

**Query (paraphrased):** "What was my project name and teaching style mentioned in this thread?"

**Evidence Required:** `["ORCHID-27", "vi du ngan"]`

**Result:** PASS (0.1 ms)

**Memory Technique:**
- **Sliding Window** - giữ recent messages
- **No compaction needed** - chỉ 5 messages, dưới threshold

**Tại sao pass:**
```
Messages: 5 messages (dưới max_recent_messages=6)
          ↓
Không trigger compaction
          ↓
Tất cả messages được giữ nguyên
          ↓
Output: <RECENT_TURNS>
        user: Ten du an ca nhan cua toi la ORCHID-27...
        assistant: Da hieu: demo ca nhan ORCHID-27, uu tien Python...
        user: Khi giai thich code, hay dung vi du ngan.
        </RECENT_TURNS>
```

---

### G03: Long-term Memory - Personal Preference

**Query (paraphrased):** "What's my preferred language for personal projects and what's my project name?"

**Evidence Required:** `["Python", "ORCHID-27"]`
**Forbidden:** `["LOTUS-88"]`

**Result:** PASS (1057.4 ms)

**Memory Technique:**
- **Zep Context Block** với `get_user_context()`
- **User Isolation**

**Tại sao pass:**
```
Step 1: prime_eval_thread(user_id="minh-lab17", query)
        → Tạo temporary thread với query context

Step 2: get_user_context(thread_id="eval-g03")
        → Zep trả Context Block với:
          - USER_SUMMARY: "Minh prefers Python... project ORCHID-27"
          - FACTS: Python preference, ORCHID-27 entity
          - ENTITIES: ORCHID-27 với summary

Step 3: Score
        → "Python" found ✓
        → "ORCHID-27" found ✓
        → "LOTUS-88" NOT found ✓ (user isolation)
```

**Lý do không leak LOTUS-88:**
- Zep user graph scoped theo `user_id="minh-lab17"`
- LOTUS-88 thuộc về `user_id="lan-lab17"` - hoàn toàn tách biệt

---

### G04: Long-term Memory - Open Loop Recall

**Query (paraphrased):** "What open loops or deadlines do I still have?"

**Evidence Required:** `["LAB-REPORT-1600", "benchmark report", "16:00"]`

**Result:** PASS (1153.2 ms)

**Memory Technique:**
- **Context Block** với fact extraction
- **Open Loop Tracking**

**Tại sao pass:**
```
Zep extraction từ session messages:
  - "TODO: hoan thanh benchmark report truoc thu Sau luc 16:00"
  - → Fact: "Minh Nguyen needs to complete the benchmark report 
            before Friday at 16:00. LAB-REPORT-1600."
  - → Entity: "benchmark report" với marker LAB-REPORT-1600

Context Block chứa:
  - Entity: "benchmark report" (summary: deadline Friday 16:00)
  - Fact: "LAB-REPORT-1600 is an open loop"
```

---

### G05: Long-term Memory - Teaching Style Preference

**Query (paraphrased):** "How do I prefer to learn async concepts?"

**Evidence Required:** `["timeline"]`

**Result:** PASS (1204.1 ms)

**Memory Technique:**
- **Context Block** với preference extraction

**Tại sao pass:**
```
Original message:
"Khi giai thich code, hay dung vi du ngan"
"Khi gap chu de nay, hay giai thich bang timeline"

→ Extracted as fact:
  "Minh prefers to be explained using timeline for coroutine/Task"

Context Block chứa:
  - Fact: "The user prefers to use a timeline when explaining 
           coroutine and Task"
```

**Note:** Đây là teaching style preference (cách học), KHÔNG phải stack preference.

---

### G06: Long-term Memory - Scoped Recency

**Query (paraphrased):** "What's the mandatory stack for my company project BLUEBIRD-42?"

**Evidence Required:** `["BLUEBIRD-42", "TypeScript", "NestJS"]`
**Forbidden:** `["LOTUS-88"]`

**Result:** PASS (1362.3 ms)

**Memory Technique:**
- **Context Block** với **Recency Rule**

**Tại sao pass:**
```
Timeline:
  Stage 1: "Minh thich Python" (2026-08-01)
  Stage 3: "BLUEBIRD-42 dung TypeScript/NestJS" (2026-08-05)

Query: "Backend BLUEBIRD-42 dung gi?"

→ Zep Context Block relevance-ranking:
  1. TypeScript/NestJS (2026-08-05, scope="BLUEBIRD-42") → HIGH relevance
  2. Python (2026-08-01, scope="ORCHID-27") → LOW relevance

Recency Rule: Fact mới với scope="BLUEBIRD-42" override generic preference
```

**Output chứa:**
```
USER_SUMMARY: "BLUEBIRD-42 requires TypeScript with NestJS; 
               Python is preferred for personal demos ORCHID-27"

FACTS:
  - Minh Nguyen updated that Python is not to be used for 
    the backend of project BLUEBIRD-42. (2026-08-05)
```

---

### G07: Long-term Memory - Both Scoped Preferences

**Query (paraphrased):** "Compare my personal project stack vs company project stack"

**Evidence Required:** `["ORCHID-27", "Python", "BLUEBIRD-42", "TypeScript", "NestJS"]`
**Forbidden:** `["LOTUS-88"]`

**Result:** PASS (1731.6 ms)

**Memory Technique:**
- **Context Block** với multi-scope extraction

**Tại sao pass:**
```
Context Block chứa CẢ HAI scopes:

1. Personal scope (ORCHID-27):
   - "Minh prefers Python for personal demos"
   - "Project ORCHID-27"

2. Company scope (BLUEBIRD-42):
   - "BLUEBIRD-42 requires TypeScript with NestJS"
   - "Python is prohibited for BLUEBIRD-42"

Query yêu cầu cả hai → Context Block cung cấp cả hai
```

---

### G08: Long-term Memory - User Isolation (Lan's Recall)

**Query (paraphrased):** "What's Lan's preferred backend stack?"

**Evidence Required:** `["LOTUS-88", "Java", "Spring Boot"]`
**Forbidden:** `["ORCHID-27", "BLUEBIRD-42", "ASYNC-FIX-20", "LAB-REPORT-1600"]`

**Result:** PASS (2759.2 ms)

**Memory Technique:**
- **User Isolation** - mỗi user có separate graph

**Tại sao pass:**
```
Query với user_id="lan-lab17"
         ↓
Zep user graph của Lan's1:
  - Session LOTUS-88: Java + Spring Boot
  - Preferences extracted

Context Block chứa:
  - LOTUS-88 is Lan's project
  - Java + Spring Boot for backend

NOT contains:
  - ORCHID-27 (Minh's project) ✗
  - BLUEBIRD-42 (Minh's company) ✗
```

---

### G09: Long-term Memory - Reverse Isolation (Minh's Projects)

**Query (paraphrased):** "List only my backend projects, NOT Lan's"

**Evidence Required:** `["ORCHID-27"]`
**Forbidden:** `["LOTUS-88", "Spring Boot"]`

**Result:** PASS (1709.5 ms)

**Memory Technique:**
- **User Isolation** + **Negative Evidence**

**Tại sao pass:**
```
Query với user_id="minh-lab17"
         ↓
Zep user graph của Minh:
  - ORCHID-27 (personal project)
  - BLUEBIRD-42 (company project)
  - Preferences, facts, etc.

Context Block chứa:
  - ORCHID-27 ✓
  - TypeScript + NestJS for BLUEBIRD-42 ✓

NOT contains:
  - LOTUS-88 ✗
  - Spring Boot ✗
```

---

### G10: Episodic Memory - Trajectory with Failed Attempt

**Query (paraphrased):** "What did I try first for async HTTP and what finally worked?"

**Evidence Required:** `["ClientSession", "concurrency=20", "ASYNC-FIX-20"]`

**Result:** PASS (316.7 ms)

**Memory Technique:**
- **Episodic Search** với `scope="episodes"`

**Tại sao pass:**
```
Step 1: graph.search(user_id="minh-lab17", scope="episodes", limit=15)
         ↓
Step 2: Render với episode_char_cap=180
         ↓
Step 3: Episodes contain raw messages:

Episode: "Hom nay toi debug async HTTP. Toi da thu tang timeout 
         len 60s nhung van fail."

Episode: "Cach hieu qua la reuse aiohttp ClientSession va dat 
         concurrency=20. Ma su co ASYNC-FIX-20."

Episode: "Da ghi nhan trajectory: increase timeout khong hieu qua; 
         ClientSession + concurrency=20 giai quyet connection churn."

Markers found: ClientSession ✓, concurrency=20 ✓, ASYNC-FIX-20 ✓
```

**Tại sao dùng episodes thay vì auto-extracted:**
- Episodes giữ nguyên raw content + markers
- Auto-extracted facts sẽ drop `ASYNC-FIX-20` marker

---

### G11: Episodic Memory - Reflection Recall

**Query (paraphrased):** "What was the root cause reflection for the async incident?"

**Evidence Required:** `["connection churn", "timeout threshold", "ASYNC-FIX-20"]`

**Result:** PASS (429.6 ms)

**Memory Technique:**
- **Episodic Memory** với reflection extraction

**Tại sao pass:**
```
Episode chứa reflection:
"Reflection: loi chinh la connection churn, khong phai timeout 
threshold. Ma su co ASYNC-FIX-20."

→ Token-by-token matching:
  - "connection churn" ✓
  - "timeout threshold" ✓
  - "ASYNC-FIX-20" ✓
```

---

### G12: Semantic Memory - Payment Retry Policy

**Query (paraphrased):** "What's the payment retry policy with all details?"

**Evidence Required:** `["Idempotency-Key", "exponential-backoff", "max-3-retries", "PAYMENT-RULE-3"]`

**Result:** PASS (1714.8 ms)

**Memory Technique:**
- **Semantic Search** với **Standalone Graph**
- **Scope="episodes"** để giữ markers

**Tại sao pass:**
```
Step 1: graph.search(graph_id="vinuni-lab17-domain-kb", 
                    scope="episodes", limit=8)
         ↓
Step 2: Standalone graph (NOT user-scoped)
         ↓
Step 3: Semantic KB document:
{
  "id": "kb-payment-retry",
  "entity": "Payment API Retry Policy",
  "summary": "For POST /payments, every retryable request MUST send 
              the same Idempotency-Key. Retry only HTTP 429 or 
              transient 5xx errors, use exponential-backoff, and 
              stop after max-3-retries. Marker: PAYMENT-RULE-3."
}

All markers found: Idempotency-Key ✓, exponential-backoff ✓, 
                  max-3-retries ✓, PAYMENT-RULE-3 ✓
```

**Tại sao dùng graph_id thay vì user_id:**
- Semantic KB là shared knowledge, không thuộc user nào
- Dùng user_id → không tìm thấy

---

### G13: Semantic Memory - Incident Playbook

**Query (paraphrased):** "What's the incident playbook step before increasing timeout?"

**Evidence Required:** `["connection pooling", "CONN-POOL-FIRST"]`

**Result:** PASS (250.2 ms)

**Memory Technique:**
- **Semantic Search** với incident playbook

**Tại sao pass:**
```
Semantic document:
{
  "id": "kb-async-http",
  "entity": "Async HTTP Incident Playbook",
  "summary": "When async HTTP calls time out, inspect connection 
              pooling, downstream saturation and concurrency before 
              increasing timeout. Reuse a long-lived client session 
              where possible. Marker: CONN-POOL-FIRST."
}

Found: connection pooling ✓, CONN-POOL-FIRST ✓
```

---

### G14: Semantic Memory - Privacy Policy

**Query (paraphrased):** "What's the privacy consent rule and deletion verification?"

**Evidence Required:** `["opt-in", "DELETE-VERIFY-ALL"]`

**Result:** PASS (268.2 ms)

**Memory Technique:**
- **Semantic KB** với privacy rules

**Tại sao pass:**
```
Semantic document:
{
  "id": "kb-memory-privacy",
  "entity": "Agent Memory Privacy Rule",
  "summary": "Do not persist personal data without explicit opt-in. 
              A deletion request must remove user-scoped memory and 
              be verified across every store. Marker: DELETE-VERIFY-ALL."
}

Found: opt-in ✓, DELETE-VERIFY-ALL ✓
```

---

### G15: Semantic Memory - Context Budget Policy

**Query (paraphrased):** "What's the token budget allocation for memory layers?"

**Evidence Required:** `["BUDGET-10-4-3-3"]`

**Result:** PASS (250.9 ms)

**Memory Technique:**
- **Semantic KB** với lab design note

**Tại sao pass:**
```
Semantic document:
{
  "id": "kb-context-budget",
  "entity": "Memory Context Budget",
  "summary": "Reserve bounded context for memory. This lab uses 
              short-term 10 percent, long-term 4 percent, episodic 
              3 percent, semantic 3 percent; trim lower-priority 
              memory first. Marker: BUDGET-10-4-3-3."
}

Found: BUDGET-10-4-3-3 ✓
```

---

### G16: Mixed - Personal Stack + Payment Policy

**Query (paraphrased):** "Write payment retry in my preferred language with official policy"

**Evidence Required:** `["Python", "Idempotency-Key", "PAYMENT-RULE-3"]`
**Forbidden:** `["LOTUS-88"]`

**Result:** PASS (1425.0 ms)

**Memory Technique:**
- **Long-term** (Context Block) + **Semantic** (Payment Policy)
- **Token Budget Assembly**

**Tại sao pass:**
```
Step 1: retrieve_long_term(user_id="minh-lab17")
         → USER_SUMMARY: "Minh prefers Python for personal demos"
         → Evidence: "Python" ✓

Step 2: retrieve_semantic(graph_id=...)
         → kb-payment-retry document
         → Evidence: "Idempotency-Key" ✓, "PAYMENT-RULE-3" ✓

Step 3: assemble_context() với budget:
         Budget: STM 10%, LT 4%, EP 3%, SEM 3%
         
         LONG_TERM: 643 tokens → trimmed to 320 tokens (with key content)
         SEMANTIC: 278 tokens → trimmed to 244 tokens (within budget)
         
         Result chứa cả Python (Minh) + Idempotency-Key (policy)
```

**Forbidden check:**
- LOTUS-88 NOT in long-term context (Minh's user graph) ✓

---

### G17: Mixed - Company Stack + Payment Policy

**Query (paraphrased):** "Write payment retry for company backend with company stack"

**Evidence Required:** `["TypeScript", "NestJS", "Idempotency-Key"]`
**Forbidden:** `["LOTUS-88"]`

**Result:** PASS (1565.2 ms)

**Memory Technique:**
- **Long-term** (Context Block với scoped facts) + **Semantic**

**Tại sao pass:**
```
Step 1: retrieve_long_term(user_id="minh-lab17")
         → USER_SUMMARY: "BLUEBIRD-42 requires TypeScript with NestJS"
         → Evidence: "TypeScript" ✓, "NestJS" ✓

Step 2: retrieve_semantic(graph_id=...)
         → kb-payment-retry document
         → Evidence: "Idempotency-Key" ✓

Step 3: assemble_context() merge

Result: TypeScript + NestJS (BLUEBIRD-42 scope) + Idempotency-Key (policy)
```

---

### G18: Mixed - Episodic + Semantic (Playbook)

**Query (paraphrased):** "Combine my async fix experience with the incident playbook"

**Evidence Required:** `["ClientSession", "CONN-POOL-FIRST"]`

**Result:** PASS (544.3 ms)

**Memory Technique:**
- **Episodic** (trajectory) + **Semantic** (playbook)

**Tại sao pass:**
```
Step 1: retrieve_episodic(user_id="minh-lab17")
         → Episodes từ minh-s2
         → Evidence: "ClientSession" ✓

Step 2: retrieve_semantic(graph_id=...)
         → kb-async-http document
         → Evidence: "CONN-POOL-FIRST" ✓

Step 3: assemble_context() merge
         → Result chứa cả hai
```

---

### G19: Mixed - Lan's Stack + Payment Policy

**Query (paraphrased):** "Write payment retry for Lan's project with Lan's stack"

**Evidence Required:** `["Java", "Spring Boot", "Idempotency-Key"]`
**Forbidden:** `["ORCHID-27", "BLUEBIRD-42"]`

**Result:** PASS (1387.3 ms)

**Memory Technique:**
- **Long-term** (Lan's user graph) + **Semantic** (shared KB)
- **User Isolation**

**Tại sao pass:**
```
Step 1: retrieve_long_term(user_id="lan-lab17")  # Lan's user graph
         → USER_SUMMARY: "Lan prioritizes Java + Spring Boot"
         → Evidence: "Java" ✓, "Spring Boot" ✓

Step 2: retrieve_semantic(graph_id=...)  # Shared KB
         → kb-payment-retry document
         → Evidence: "Idempotency-Key" ✓

Step 3: assemble_context()

Forbidden NOT found:
  - ORCHID-27 (Minh's project) ✗
  - BLUEBIRD-42 (Minh's company) ✗
```

---

### G20: Mixed - Three-Layer Assembly

**Query (paraphrased):** "Combine personal language + payment policy + async fix experience"

**Evidence Required:** `["Python", "Idempotency-Key", "ClientSession"]`
**Forbidden:** `["LOTUS-88"]`

**Result:** PASS (1858.3 ms)

**Memory Technique:**
- **Three-layer assembly**: Long-term + Episodic + Semantic
- **Token Budget Management**

**Tại sao pass:**
```
Step 1: retrieve_long_term(user_id="minh-lab17")
         → USER_SUMMARY: "Minh prefers Python"
         → Evidence: "Python" ✓

Step 2: retrieve_episodic(user_id="minh-lab17")
         → Episodes from minh-s2
         → Evidence: "ClientSession" ✓

Step 3: retrieve_semantic(graph_id=...)
         → kb-payment-retry document
         → Evidence: "Idempotency-Key" ✓

Step 4: assemble_context() với priority:
         1. Short-term (0%)
         2. Long-term (4% = 320 tokens) → "Python" retained ✓
         3. Episodic (3% = 240 tokens) → "ClientSession" retained ✓
         4. Semantic (3% = 240 tokens) → "Idempotency-Key" retained ✓

Result: All three evidence tokens found
```

---

## Memory Techniques Summary Table

| Case | Layer | Techniques | Key Challenge |
|------|-------|------------|---------------|
| G01 | short_term | Sliding + Compaction + Durable Notes | 2 constraints survive 12 fillers |
| G02 | short_term | Sliding Window | In-thread recall, no compaction |
| G03 | long_term | Context Block + User Isolation | Paraphrased query, no Lan leak |
| G04 | long_term | Context Block + Fact Extraction | Open loop with synthetic marker |
| G05 | long_term | Context Block + Preference Extraction | Teaching style (not stack) |
| G06 | long_term | Context Block + Recency Rule | Scope-specific override |
| G07 | long_term | Context Block + Multi-scope | Both personal + company scopes |
| G08 | long_term | Context Block + User Isolation | Lan's recall, no Minh data |
| G09 | long_term | Context Block + Negative Evidence | List own projects only |
| G10 | episodic | Graph Search + Episodes + Trajectory | Failed attempt + working fix |
| G11 | episodic | Graph Search + Episodes + Reflection | Root cause reflection |
| G12 | semantic | Standalone Graph + Episodes | Full payment policy |
| G13 | semantic | Standalone Graph + Episodes | Incident playbook |
| G14 | semantic | Standalone Graph + Episodes | Privacy policy |
| G15 | semantic | Standalone Graph + Episodes | Context budget rule |
| G16 | mixed | LT + SEM + Budget Assembly | Personal stack + policy |
| G17 | mixed | LT + SEM + Budget Assembly | Company scope + policy |
| G18 | mixed | EP + SEM + Budget Assembly | Experience + playbook |
| G19 | mixed | LT + SEM + Budget Assembly + Isolation | Lan's stack + policy |
| G20 | mixed | LT + EP + SEM + Budget Assembly | Three layers |

---

## Key Observations

### 1. No-Memory Baseline Comparison

| Aspect | Practice (11 cases) | Golden (20 cases) |
|--------|---------------------|-------------------|
| Memory Hit Rate | 100% | 100% |
| No-Memory Hit Rate | 18.2% | ~10% (estimated) |
| Improvement | +81.8% | +90% |

### 2. Token Budget Effectiveness

Golden cases có:
- Avg token reduction: 6.3% (practice) → 6.3% (golden)
- Điều này cho thấy budget được apply đúng nhưng vẫn giữ được evidence

### 3. Long Prompt Impact

Golden queries:
- Dài hơn practice 3-5 lần
- Chứa nhiều distractor context
- **Nhưng vẫn pass** vì:
  - Retrieval dựa trên **evidence markers**, không phải query matching
  - Context Block / Episodes / Semantic KB chứa markers cố định

### 4. Recency và Scope

Cases G06, G07 chứng minh:
- Fact mới với scope cụ thể override fact cũ
- BLUEBIRD-42 → TypeScript (Stage 3) override Python (Stage 1) cho project đó
- Nhưng Python vẫn tồn tại cho ORCHID-27

### 5. User Isolation là Absolute

Cases G03, G08, G09, G19 chứng minh:
- Minh không bao giờ thấy LOTUS-88 (Lan's)
- Lan không bao giờ thấy ORCHID-27, BLUEBIRD-42 (Minh's)
- Zep user graph hoàn toàn tách biệt

---

## Conclusion

**20/20 PASS** chứng minh rằng:

1. **Short-term memory** với sliding window và durable notes hoạt động hiệu quả cho compaction
2. **Long-term memory** với Context Block xử lý tốt recency, scope, và user isolation
3. **Episodic memory** giữ được trajectories và reflections với markers
4. **Semantic memory** với standalone graph cung cấp shared knowledge hiệu quả
5. **Mixed assembly** với token budget kết hợp multiple layers đúng cách
6. **Ground truth evaluation** với exact substring matching đáng tin cậy hơn LLM judgment

---

*Golden Evaluation completed: 2026-08-17*
*Total Points Earned: 10/10*
