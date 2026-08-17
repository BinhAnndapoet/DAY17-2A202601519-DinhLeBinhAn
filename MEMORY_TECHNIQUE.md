# Memory Techniques trong Lab 17

Document này tổng hợp tất cả các kỹ thuật memory được test và implement trong Lab 17 - Multi-Memory Agent với Zep Cloud V3.

---

## Mục lục

1. [Tổng quan kiến trúc](#1-tổng-quan-kiến-trúc)
2. [Short-term Memory](#2-short-term-memory)
3. [Long-term Memory](#3-long-term-memory)
4. [Episodic Memory](#4-episodic-memory)
5. [Semantic Memory](#5-semantic-memory)
6. [Context Assembly & Token Budget](#6-context-assembly--token-budget)
7. [Privacy & Compliance](#7-privacy--compliance)
8. [Evaluation Methodology](#8-evaluation-methodology)

---

## 1. Tổng quan kiến trúc

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           USER QUERY                                        │
│                    "Backend cua BLUEBIRD-42 dung gi?"                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          MEMORY ROUTER                                      │
│              Route query → appropriate memory layer(s)                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         │                          │                          │
         ▼                          ▼                          ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  SHORT-TERM     │      │  LONG-TERM      │      │  EPISODIC       │
│  (Local)        │      │  (Zep Context)  │      │  (Zep Graph)    │
│                 │      │                 │      │                 │
│ Session context │      │ Cross-session   │      │ Past trajectory │
│ + Durable notes │      │ preferences     │      │ + reflections   │
└─────────────────┘      └─────────────────┘      └─────────────────┘
                                                            │
                                                            ▼
                                               ┌─────────────────┐
                                               │  SEMANTIC        │
                                               │  (Zep Graph)     │
                                               │                 │
                                               │ Domain knowledge │
                                               │ (shared KB)     │
                                               └─────────────────┘
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       CONTEXT ASSEMBLER                                      │
│              Merge layers + Token Budget + Priority Trimming                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ASSEMBLED CONTEXT                                     │
│                                                                              │
│  <SHORT_TERM>...</SHORT_TERM>                                               │
│  <LONG_TERM>...Python preference...</LONG_TERM>                            │
│  <SEMANTIC>...TypeScript/NestJS...</SEMANTIC>                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AGENT RESPONSE                                      │
│              "Cho project BLUEBIRD-42, backend bat buoc dung TypeScript..."  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Short-term Memory

### 2.1 Khái niệm

Short-term memory (hay working memory) lưu trữ context của conversation hiện tại. Trong lab này, đây là **local memory** không qua network.

### 2.2 Ba strategies được so sánh

| Strategy | Mô tả | Ưu điểm | Nhược điểm |
|----------|-------|----------|-------------|
| **Buffer** | Giữ tất cả messages | Audit trail đầy đủ | Token tăng tuyến tính, overflow |
| **Summary** | Nén old messages thành summary | Tiết kiệm tokens | Cần LLM cho summary |
| **Sliding** | Summary + last K turns + durable notes | Balance tốt | Phức tạp hơn |

### 2.3 Sliding Window Implementation

```python
class ShortTermMemory:
    def __init__(self, strategy="sliding", max_recent_messages=6, pressure_tokens=500):
        self.strategy = strategy
        self.max_recent_messages = max_recent_messages
        self.pressure_tokens = pressure_tokens
        self.messages = []      # Recent messages
        self.summary = ""       # Compressed old content
        self.durable_notes = [] # Must-keep constraints

    def add(self, role, content):
        self.messages.append(ChatMessage(role, content))
        if self.strategy != "buffer" and self.detect_pressure():
            self.compact()

    def detect_pressure(self):
        # Trigger compaction khi:
        # 1. Quá max messages HOẶC
        # 2. Token estimate vượt threshold
        return (len(self.messages) > self.max_recent_messages or
                tokens(self.messages) > self.pressure_tokens)
```

### 2.4 Compaction

```python
def compact(self):
    if self.strategy == "summary":
        old = self.messages[:-2]
        keep = self.messages[-2:]
    else:  # sliding
        keep_n = max(2, self.max_recent_messages)
        old = self.messages[:-keep_n]
        keep = self.messages[-keep_n:]

    # Extract durable notes từ old messages
    new_notes = self.extract_durable_notes(old)
    for note in new_notes:
        if note not in self.durable_notes:
            self.durable_notes.append(note)

    # Update summary
    self.summary = self._summarize(old)
    self.messages = keep
```

### 2.5 Durable Notes Pattern Detection

```python
DURABLE_PATTERNS = (
    "todo", "deadline", "constraint", "decision",
    "must", "bat buoc", "khong duoc quen",
    "open loop", "preference", "uu tien",
)

def extract_durable_notes(self, messages):
    notes = []
    for msg in messages:
        text = msg.content.lower()
        has_marker = bool(re.search(r"\b[A-Z][A-Z0-9-]{5,}\b", text))
        if any(p in text for p in DURABLE_PATTERNS) or has_marker:
            notes.append(f"{msg.role}: {text}")
    return notes
```

### 2.6 Render Output

```python
def render(self):
    parts = []
    if self.summary:
        parts.append(f"<SESSION_SUMMARY>\n{self.summary}\n</SESSION_SUMMARY>")
    if self.durable_notes:
        parts.append("<DURABLE_NOTES>\n- " + "\n- ".join(self.durable_notes) + "\n</DURABLE_NOTES>")
    if self.messages:
        parts.append("<RECENT_TURNS>\n" + self.messages_text() + "\n</RECENT_TURNS>")
    return "\n".join(parts)
```

### 2.7 Test cases

| Case | Query | Evidence | Kỹ thuật được test |
|------|-------|----------|-------------------|
| E01 | "Ten du an ca nhan toi vua nhac la gi?" | ORCHID-27 | Recent in-thread recall |
| E10 | "Deadline review cu la khi nao?" | REVIEW-DEADLINE-1600, Friday, 16:00 | Compaction + durable notes |

---

## 3. Long-term Memory

### 3.1 Khái niệm

Long-term memory lưu trữ preferences, facts, và decisions bền vững qua nhiều sessions. Trong lab này, dùng **Zep Context Block** - một managed memory system tự động tổng hợp và relevance-rank.

### 3.2 Zep Context Block

```python
# src/memory_student.py
def retrieve_long_term(self, user_id, thread_id, query):
    # Step 1: Prime - tạo context cho relevance ranking
    prime_eval_thread(self.client, user_id, thread_id, query)

    # Step 2: Lấy Context Block
    context = self.client.thread.get_user_context(thread_id=thread_id)
    return context.context
```

### 3.3 Prime Thread

```python
def prime_eval_thread(client, user_id, thread_id, query):
    # Xóa thread cũ (nếu có)
    safe_call(client.thread.delete, thread_id=thread_id)

    # Tạo thread mới gắn với user
    client.thread.create(thread_id=thread_id, user_id=user_id)

    # Add evaluation query
    # Zep dùng query này để relevance-rank facts
    client.thread.add_messages(
        thread_id,
        messages=[Message(role="user", name="Evaluation User", content=query)],
        ignore_roles=["user"]
    )
```

### 3.4 Tại sao cần Prime?

**Không prime:**
```
Query: "Backend BLUEBIRD-42 dung gi?"
→ Zep không biết user đang hỏi về project nào
→ Trả về generic facts
```

**Có prime:**
```
Query: "Backend BLUEBIRD-42 dung gi?"
→ Zep thấy query mention "BLUEBIRD-42"
→ Relevance-rank: ưu tiên facts liên quan đến BLUEBIRD-42
→ Trả về TypeScript/NestJS (vì đó là fact liên quan)
```

### 3.5 Recency Rule

```python
# Minh nói ở stage 1:
"Minh thich Python"  # timestamp: 2026-08-01

# Minh nói ở stage 3:
"BLUEBIRD-42 dung TypeScript"  # timestamp: 2026-08-05

# Query: "Backend BLUEBIRD-42 dung gi?"
# → Zep trả về TypeScript vì:
#   1. Fact có scope "BLUEBIRD-42"
#   2. Fact mới hơn (08-05 > 08-01)
```

**Recency rule:** Fact mới với cùng scope override fact cũ.

### 3.6 User Isolation

```python
# Minh query
retrieve_long_term(user_id="minh-lab17", ...)
# → Chỉ thấy: ORCHID-27, Python, BLUEBIRD-42, ASYNC-FIX-20

# Lan query
retrieve_long_term(user_id="lan-lab17", ...)
# → Chỉ thấy: LOTUS-88, Java, Spring Boot
# → KHÔNG thấy: ORCHID-27 (của Minh)
```

**Tại sao?** Mỗi user có user_id riêng trong Zep → data hoàn toàn isolated.

### 3.7 Test cases

| Case | Query | Evidence | Kỹ thuật được test |
|------|-------|----------|-------------------|
| E02 | "Demo ca nhan Minh uu tien gi?" | Python | Cross-session recall |
| E03 | "Minh con open loop nao?" | benchmark report, 16:00 | Durable task recall |
| E08 | "Backend BLUEBIRD-42 dung gi?" | TypeScript, NestJS, BLUEBIRD-42 | **Recency** |
| E09 | "Lan uu tien gi cho LOTUS-88?" | Java, Spring Boot, LOTUS-88 | **User isolation** |

---

## 4. Episodic Memory

### 4.1 Khái niệm

Episodic memory lưu trữ **trajectories** - các sự kiện đã xảy ra với outcome và reflection. Khác với facts đơn lẻ, episodic nhớ **chuỗi hành động → kết quả**.

### 4.2 Implementation

```python
def retrieve_episodic(self, user_id, query):
    from .utils import cap_query
    results = self.client.graph.search(
        user_id=user_id,        # User-scoped
        query=cap_query(query), # Cắt nếu > 400 chars
        scope="episodes",       # Raw episodes, giữ markers
        limit=15,               # Lấy 15 episodes
    )
    return render_graph_search(results, episode_char_cap=180)
```

### 4.3 Tại sao scope="episodes"?

| Scope | Trả về | Giữ markers? |
|-------|--------|--------------|
| `episodes` | Raw source messages | ✅ Có |
| `edges` | Extracted facts | ❌ Không |
| `nodes` | Entities | ❌ Không |
| `auto` | Zep auto-decides | ❌ Không |

**Ví dụ:**
```python
# scope="episodes" → trả về:
"Ma su co ASYNC-FIX-20. Cach hieu qua la reuse aiohttp ClientSession..."

# scope="auto" → trả về:
"Effective approach was reusing ClientSession."  # Marker đã mất!
```

### 4.4 Character Cap

```python
def render_graph_search(results, episode_char_cap=180):
    for episode in results.episodes:
        content = episode.content
        if episode_char_cap:
            content = content[:episode_char_cap]  # Cắt 180 chars
        parts.append(f"EPISODE: {content}")
```

**Tại sao cắt?**
- Mỗi episode có thể rất dài (full conversation)
- 15 episodes × 1000 chars = 15,000 chars > 240 tokens budget
- Cắt 180 chars/episode → fit trong episodic budget

### 4.5 Trajectory Pattern

```python
# E04 - "Fix async HTTP cách nào?"
# Episodic search trả về trajectory:

Episode 1: "Hom nay toi debug async HTTP. Toi da thu tang timeout len 60s nhung van fail."
Episode 2: "Cach hieu qua la reuse aiohttp ClientSession va dat concurrency=20. Ma su co ASYNC-FIX-20."
Episode 3: "Da ghi nhan trajectory: increase timeout khong hieu qua; ClientSession + concurrency=20 giai quyet connection churn."

# Agent có thể reconstruct:
# 1. Tried: increase timeout → failed
# 2. Worked: reuse ClientSession + concurrency=20
# 3. Reflection: root cause was connection churn, not timeout
```

### 4.6 Test cases

| Case | Query | Evidence | Kỹ thuật được test |
|------|-------|----------|-------------------|
| E04 | "Fix async HTTP cách nào?" | ClientSession, concurrency=20, ASYNC-FIX-20 | Trajectory recall |
| E05 | "Reflection là gì?" | connection churn, timeout threshold | Episode reflection |

---

## 5. Semantic Memory

### 5.1 Khái niệm

Semantic memory lưu trữ **domain knowledge** - tri thức chung không thuộc user nào. Khác với episodic (cá nhân), semantic là **shared** và **impersonal**.

### 5.2 Standalone Graph vs User Graph

```
┌─────────────────────────────────────────────────────────────────┐
│                     ZEP CLOUD                                   │
│                                                                 │
│  USER GRAPHS (per user)                                        │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │  minh-lab17  │  │  lan-lab17   │                            │
│  │  - sessions  │  │  - sessions  │                            │
│  │  - facts     │  │  - facts     │                            │
│  └──────────────┘  └──────────────┘                            │
│         │                │                                      │
│         └────────────────┴────────────────────────────────────┐  │
│                          │                                     │  │
│                          ▼                                     │  │
│               ┌─────────────────────┐                          │  │
│               │  STANDALONE GRAPH   │                          │  │
│               │  (shared domain KB) │                          │  │
│               │                     │                          │  │
│               │  - Payment Rules    │                          │  │
│               │  - Incident Playbook│                          │  │
│               │  - API Guidelines   │                          │  │
│               └─────────────────────┘                          │  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 Implementation

```python
def retrieve_semantic(self, graph_id, query):
    from .utils import cap_query
    q = cap_query(query)
    try:
        results = self.client.graph.search(
            graph_id=graph_id,        # Standalone graph, NOT user_id!
            query=q,
            scope="episodes",        # Giữ markers
            limit=8,
        )
    except Exception:
        # Fallback nếu episodes không supported
        results = self.client.graph.search(
            graph_id=graph_id,
            query=q,
            scope="nodes",           # Entities/facts
            limit=8,
        )
    return render_graph_search(results)
```

### 5.4 Tại sao dùng graph_id?

```python
# SAI - dùng user_id cho semantic
results = client.graph.search(
    user_id="minh-lab17",  # ❌ User-scoped, không tìm thấy shared KB
    ...
)

# ĐÚNG - dùng graph_id cho semantic
results = client.graph.search(
    graph_id="vinuni-lab17-domain-kb",  # ✅ Standalone graph
    ...
)
```

### 5.5 Knowledge Documents

```jsonl
// data/knowledge.jsonl
{"id":"kb-payment-retry","entity":"Payment API Retry Policy","summary":"For POST /payments, send Idempotency-Key, retry HTTP 429/5xx, exponential-backoff, max-3-retries. Marker: PAYMENT-RULE-3."}
{"id":"kb-async-http","entity":"Async HTTP Incident Playbook","summary":"When async HTTP calls time out, inspect connection pooling before increasing timeout. Marker: CONN-POOL-FIRST."}
```

### 5.6 Test cases

| Case | Query | Evidence | Kỹ thuật được test |
|------|-------|----------|-------------------|
| E06 | "Quy tac retry payment?" | Idempotency-Key, max-3-retries, exponential-backoff | Domain knowledge retrieval |
| E11 | "Incident playbook?" | connection pooling, CONN-POOL-FIRST | Playbook lookup |

---

## 6. Context Assembly & Token Budget

### 6.1 Khái niệm

Khi cần nhiều layers cùng lúc (mixed query), cần **assembly** - kết hợp outputs từ nhiều memory layers vào một context duy nhất, với token budget và priority.

### 6.2 Token Budget Allocation

```python
@dataclass(frozen=True)
class LayerBudget:
    short_term: float = 0.10   # 800 tokens (10%)
    long_term: float = 0.04   # 320 tokens (4%)
    episodic: float = 0.03    # 240 tokens (3%)
    semantic: float = 0.03    # 240 tokens (3%)
    # Total: 20% = 1600 tokens cho memory
    # Remaining 80% = 6400 tokens cho system/working context
```

### 6.3 Priority Order

```python
class ContextBudgetManager:
    priority = ("short_term", "long_term", "episodic", "semantic")

    def assemble(self, layers):
        rendered = []
        breakdown = {}
        for layer in self.priority:
            raw = layers.get(layer, "")
            limit = self.layer_limit(layer)    # e.g., 320 for long_term
            trimmed = self.trim(raw, limit)    # Cắt nếu vượt limit
            breakdown[layer] = {...}
            if trimmed.strip():
                rendered.append(f"<{layer.upper()}>\n{trimmed}\n</{layer.upper()}>")
        return "\n\n".join(rendered), breakdown
```

### 6.4 Priority Trimming Logic

```
Layer:  SHORT_TERM  →  LONG_TERM  →  EPISODIC  →  SEMANTIC
Budget:   800 tokens     320 tokens    240 tokens    240 tokens
Priority:    HIGH          MEDIUM         LOW           LOW
Trim:       NEVER         FIRST          SECOND        THIRD

Example:
- Semantic raw: 500 tokens (vượt 240 limit)
- Semantic after trim: 240 tokens + [...trimmed...]
- Nếu không trim, total sẽ vượt context window
```

### 6.5 Mixed Layer Example (E07)

```python
# Query: "Hay chon retry policy phu hop voi preference ca nhan cua Minh"
# → Cần cả: Python (long-term) + Idempotency-Key (semantic)

layers = {
    "short_term": "",
    "long_term": retrieve_long_term(user_id, thread_id, query),
    "episodic": "",
    "semantic": retrieve_semantic(graph_id, query),
}

assembled, breakdown = memory_impl.assemble_context(layers)
```

**Output:**
```xml
<LONG_TERM>
<USER_SUMMARY>
Minh prefers Python for personal demos like ORCHID-27...
</USER_SUMMARY>
</LONG_TERM>

<SEMANTIC>
EPISODE: {"id":"kb-payment-retry","entity":"Payment API Retry Policy",
"summary":"...Idempotency-Key...max-3-retries..."}
</SEMANTIC>
```

### 6.6 Test case

| Case | Query | Evidence | Kỹ thuật được test |
|------|-------|----------|-------------------|
| E07 | "Chon retry policy phu hop preference Minh" | Python + Idempotency-Key | **Multi-layer assembly** |

---

## 7. Privacy & Compliance

### 7.1 Consent Gate

```python
def require_memory_consent(user_id):
    record = load_consent().get(user_id)
    if not record or not record["memory_opt_in"]:
        raise PermissionError(f"No durable-memory opt-in for {user_id}")
    return record
```

**Tại sao?**
- GDPR requirement: Không lưu data khi chưa có permission
- Lab muốn teach Privacy-by-Design

### 7.2 PII Minimization

```python
EMAIL_RE = re.compile(r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}")
PHONE_RE = re.compile(r"(?<!\d)(?:\+?\d[\d -]{7,}\d)(?!\d)")

def minimize_pii(text):
    text = EMAIL_RE.sub("[EMAIL_REDACTED]", text)
    text = PHONE_RE.sub("[PHONE_REDACTED]", text)
    return text
```

**Tại sao?**
- Zep là external service: data leaves infrastructure
- Chỉ gửi những gì cần thiết cho benchmark

### 7.3 Right to be Forgotten

```python
def forget(user_id):
    # Xóa từ Zep Cloud
    client.user.delete(user_id=user_id)

    # Xóa từ Redis cache
    redis_client.delete_pattern(f"user:{user_id}:*")

    # Semantic graph KHÔNG bị xóa
    # (shared domain knowledge, không có PII)
```

### 7.4 Verification

```python
def verify_forget(user_id):
    # Zep
    zep_user = client.user.get(user_id=user_id)
    assert zep_user is None  # "Zep user absent: True"

    # Redis
    remaining = redis_client.keys(f"user:{user_id}:*")
    assert len(remaining) == 0  # "Redis user keys remaining: 0"
```

---

## 8. Evaluation Methodology

### 8.1 Ground Truth Structure

```json
{
  "id": "E06",
  "expected_layer": "semantic",
  "query": "Quy tac retry POST payment la gi?",
  "must_contain_all": ["Idempotency-Key", "max-3-retries", "exponential-backoff"],
  "must_not_contain": []
}
```

### 8.2 Scoring Logic

```python
def score_case(case, retrieved):
    text = normalize(retrieved)  # lowercase, strip whitespace
    missing = [x for x in case["must_contain_all"] if x not in text]
    forbidden = [x for x in case["must_not_contain"] if x in text]
    return not missing and not forbidden  # All-or-nothing
```

### 8.3 Tại sao exact substring matching?

| Phương pháp | Ưu điểm | Nhược điểm |
|-------------|----------|-------------|
| **Exact substring** | Không hallucination, rõ ràng | Không linh hoạt |
| LLM judge | Linh hoạt, hiểu semantics | Có thể hallucinate |

**Lab dùng exact substring vì:**
- Retrieval phải **thực sự lấy được** evidence
- LLM có thể generate marker mà không retrieve
- Benchmark đánh giá **memory system**, không phải **LLM generation**

### 8.4 Token Reduction

```python
retrieved_tokens = estimate_tokens(retrieved)
full_tokens = estimate_tokens(source_text)
reduction = 1.0 - retrieved_tokens / full_tokens
```

**Ví dụ:**
- Semantic: 107 tokens retrieved vs 459 full → 76.7% reduction
- No-memory: 0 tokens vs 565 full → 100% reduction

**Caveat:** High reduction không tốt nếu missing information!

### 8.5 Summary của tất cả test cases

| Case | Layer | Evidence | Kỹ thuật |
|------|-------|----------|-----------|
| E01 | short_term | ORCHID-27 | In-thread recall |
| E02 | long_term | Python | Cross-session preference |
| E03 | long_term | benchmark report, 16:00 | Open-loop recall |
| E04 | episodic | ClientSession, concurrency=20, ASYNC-FIX-20 | Trajectory |
| E05 | episodic | connection churn, timeout threshold | Reflection |
| E06 | semantic | Idempotency-Key, max-3-retries, exponential-backoff | Domain KB |
| E07 | mixed | Python + Idempotency-Key | Multi-layer assembly |
| E08 | long_term | BLUEBIRD-42, TypeScript, NestJS | **Recency** |
| E09 | long_term | LOTUS-88, Java, Spring Boot (not ORCHID-27) | **User isolation** |
| E10 | short_term | REVIEW-DEADLINE-1600, Friday, 16:00 | Compaction |
| E11 | semantic | connection pooling, CONN-POOL-FIRST | Incident playbook |

---

## 9. So sánh các Memory Types

| Aspect | Short-term | Long-term | Episodic | Semantic |
|--------|------------|-----------|----------|----------|
| **Scope** | Session | User | User | Shared |
| **Content** | Current conversation | Facts, preferences | Trajectories | Domain knowledge |
| **Technology** | Local Python | Zep Context Block | Zep Graph Search | Zep Standalone Graph |
| **Retrieval** | Direct access | get_user_context | graph.search | graph.search |
| **Updates** | On every message | Periodic | After session | Rare |
| **Forgotten** | When session ends | On request | On request | Never (shared) |
| **Use case** | What just happened? | What's my preference? | What did we try before? | What's the policy? |

---

## 10. Key Takeaways

1. **Memory có layers khác nhau** - không có memory nào phù hợp cho mọi use case
2. **Short-term cần compaction** - buffer không scale
3. **Long-term cần recency + scope** - fact mới override fact cũ cùng scope
4. **Episodic nhớ trajectories** - không chỉ facts mà cả hành động → kết quả
5. **Semantic là shared** - không thuộc user nào, dùng graph_id
6. **Token budget cần priority** - trim layer thấp priority trước
7. **Privacy là first-class** - consent, minimization, right-to-be-forgotten
8. **Evaluation cần ground truth** - exact substring > LLM judge

---

*Tài liệu tham khảo: Lab 17 - Multi-Memory Agent with Zep*
