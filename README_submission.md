# Lab 17 Submission - Multi-Memory Agent với Zep

## 3 Câu Phân tích (LAB.md mục 5.2)

**1. Layer quan trọng nhất trong bộ test này:**
Long-term memory (Context Block) là layer quan trọng nhất vì chiếm nhiều cases nhất (7/20 golden + 4/11 practice), bao gồm cả user isolation (G08: Lan không được thấy Minh's data, G09: Minh không thấy Lan's data) và recency (G06: BLUEBIRD-42 → TypeScript override).

**2. Trade-off Context Block/Zep vs Redis+Qdrant:**
Zep Cloud V3 cung cấp managed memory với Context Block tự động relevance-ranking, user graph, và standalone semantic graph — giảm boilerplate đáng kể so với tự build Redis (KV + TTL) + Qdrant (vector search). Tuy nhiên, local baseline cho phép kiểm soát hoàn toàn data locality và chi phí. Trong lab, trade-off rõ ràng: Zep nhanh hơn để implement nhưng phụ thuộc cloud; Redis+Qdrant linh hoạt hơn nhưng cần tự quản lý. Quan trọng: semantic layer cần dùng `graph_id` thay vì `user_id` — đây là điểm khác biệt then chốt.

**3. Guardrail chống memory poisoning:**
Memory poisoning xảy ra khi attacker inject thông tin sai vào persistent memory để manipulate agent behavior. Guardrails trong lab: (1) Consent-gated ingestion (consent.json), (2) User isolation với user_id scoping trong mọi Zep call, (3) Right-to-be-forgotten cho phép xóa hoàn toàn user memory (privacy drill), (4) Heartbeat không được phép tự thêm quyền mới.

## 4 Câu Phân tích Benchmark

**Câu 1: Layer nào có hit rate thấp nhất?**
Không có layer nào fail trong cả practice (11/11) và golden (20/20). Tất cả đều đạt 100%. Tuy nhiên, semantic layer có latency cao nhất (avg ~600-1700ms) do phải query standalone graph. Long-term layer có latency trung bình ~1500ms với Context Block.

**Câu 2: Query nào retrieve nhiều token nhất?**
G20 (3-layer mixed) retrieve 831 tokens - cao nhất trong golden. G09 retrieve 845 tokens trong practice. Đây là cases cần kết hợp nhiều layers, Context Block đầy đủ với user summary + episodes + facts + entities + threads.

**Câu 3: E07/G16/G19 (mixed) cần kết hợp memory nào? Evidence bắt buộc?**
- E07: Long-term (Python preference) + Semantic (Idempotency-Key) → cả hai đều bắt buộc
- G16: Long-term (Python) + Semantic (Idempotency-Key, PAYMENT-RULE-3) → thêm marker
- G19: Long-term (Lan: Java + Spring Boot) + Semantic (Idempotency-Key) → user isolation rõ ràng
- G20: Long-term + Episodic + Semantic → 3 layers, không layer nào riêng lẻ đủ

**Câu 4: Token reduction và tại sao no-memory có reduction cao nhưng hit rate thấp?**
No-memory baseline có 81.8% token reduction vì nó retrieve gần như NOTHING. Con số này deceptive: dropping all context = cheap nhưng incorrect. Memory-enabled đạt 6.3-15% reduction với 100% hit rate — trade-off chấp nhận được. Golden set với long prompts vẫn đạt 100% vì retrieval dựa trên markers trong memory, không phải query matching.

## Golden Set Analysis (20/20 PASS)

**Key observations:**

1. **Long prompts không ảnh hưởng retrieval:** G01-G20 dùng prompts dài gấp 3-5 lần practice, nhưng vẫn pass 100% vì retrieval dựa trên evidence markers (như `PAYMENT-RULE-3`, `CONN-POOL-FIRST`), không phải query similarity.

2. **Recency hoạt động đúng:** G06-G07 cho thấy BLUEBIRD-42 (Stage 3) → TypeScript override Python (Stage 1) cho project đó, nhưng Python vẫn tồn tại cho ORCHID-27. Context Block xử lý scope-based recency tự động.

3. **User isolation là tuyệt đối:** G03, G08, G09, G19 chứng minh Minh không bao giờ thấy LOTUS-88, Lan không bao giờ thấy ORCHID-27/BLUEBIRD-42. Zep user graph scoped hoàn toàn theo user_id.

4. **Three-layer assembly hoạt động:** G20 kết hợp long-term (Python) + episodic (ClientSession) + semantic (Idempotency-Key) với token budget 10/4/3/3% — tất cả evidence được giữ lại trong priority order.

5. **Durable notes cho constraints:** G01 với 2 constraints (HOLD-ALPHA-0900, HOLD-BETA-STAGING) survive 12 filler messages nhờ durable notes pattern detection (`constraint`, uppercase markers).

## E08/E10 Recency & Compaction

**E08 - Recency:**
Stage 3 update ghi đè preference: BLUEBIRD-42 backend PHẢI dùng TypeScript/NestJS (không phải Python). Context Block của Zep tự động xử lý recency: fact mới có timestamp 2026-08-05 ghi đè fact cũ 2026-08-01 cho cùng scope. Query về BLUEBIRD-42 trả về TypeScript, không phải Python.

**E10/G01 - Compaction:**
Buffer strategy giữ tất cả messages (không evict). Sliding với max=4 cần 12 compactions để giữ constraint REVIEW-DEADLINE-1600. G01 cho thấy 2 constraints survive 14 filler turns. Compaction không "tóm tắt văn hóa" — nó phải ưu tiên state, decision, TODO, constraint. Durable notes là cơ chế bắt buộc: marker constraint không bị evict dù nhiều filler turns đã qua.

## Technical Summary

| Metric | Practice | Golden |
|--------|----------|--------|
| Hit Rate | 100% (11/11) | 100% (20/20) |
| Avg Latency | 1888 ms | 1049 ms |
| Token Reduction | 15.0% | 6.3% |
| Total Points | 56 + 24 = 80 | +10 bonus = 90 |

**Kỹ thuật đã implement thành công:**
- Short-term: Sliding window + compaction + durable notes
- Long-term: Context Block + prime thread + user isolation + recency
- Episodic: Graph search scope="episodes" + trajectory
- Semantic: Standalone graph + graph_id + markers
- Mixed: Token budget 10/4/3/3% + priority assembly
- Privacy: Consent gate + PII minimization + right-to-be-forgotten
