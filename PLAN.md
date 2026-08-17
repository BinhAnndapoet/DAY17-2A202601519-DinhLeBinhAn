# Plan: Lab 17 - Multi-Memory Agent với Zep

## Tổng quan Lab

Lab xây dựng agent có 4 memory layer, đánh giá bằng ground truth retrieval (11 test case E01-E11).

### Memory Layers cần implement:
| Layer | Nguồn | TODO | Case |
|-------|-------|------|------|
| Short-term | Local (buffer/sliding) | Không cần code | E01, E10 |
| Long-term | Zep Context Block | TODO 1/4 | E02, E03, E08, E09 |
| Episodic | Zep user graph | TODO 2/4 | E04, E05 |
| Semantic | Zep standalone graph | TODO 3/4 | E06, E11 |
| Assembly | ContextBudgetManager | TODO 4/4 | E07 |

### Mục tiêu:
- **Pass: >= 56/80** và **hit rate >= 80%** (9/11)
- **Golden set**: +10 (optional)
- **UI demo**: +10 (optional)

---

## Phase 1: Chuẩn bị môi trường (0-15 phút)

### 1.1 Setup Docker
```bash
cp .env.example .env
# Thêm ZEP_API_KEY vào .env

docker compose build
docker compose up -d redis qdrant
```

### 1.2 Smoke test
```bash
docker compose run --rm app python -m src.smoke
```

### 1.3 Seed data (1 lần duy nhất)
```bash
docker compose run --rm app python -m src.seed
```

---

## Phase 2: Quan sát Short-term Memory (15-45 phút)

### 2.1 Demo 3 strategies
```bash
docker compose run --rm app python -m src.demo_short_term
```

### 2.2 Thử giảm max_recent_messages (6→4)
- File: `src/demo_short_term.py`
- Kiểm tra durable notes còn giữ được constraint

### 2.3 Ghi nhận cho README_submission.md
- 2-3 câu về compaction giữ constraint nào, tại sao buffer không đủ

---

## Phase 3: TODO 1/4 - Long-term Memory (45-85 phút)

### 3.1 Implement `retrieve_long_term`

**File**: `src/memory_student.py`

**Code cần viết**:
```python
def retrieve_long_term(self, user_id: str, thread_id: str, query: str) -> str:
    prime_eval_thread(self.client, user_id, thread_id, query)
    context = self.client.thread.get_user_context(thread_id=thread_id)
    return context.context
```

### 3.2 Test riêng long-term
```bash
docker compose run --rm app python -m src.evaluate \
  --impl student --reuse-seeded --only-layer long_term
```

### 3.3 Checklist Pha B
- [ ] `prime_eval_thread` được gọi trước
- [ ] Return `context.context` (string)
- [ ] E02: `Python` PASS
- [ ] E03: `benchmark report` + `16:00` PASS
- [ ] E08: `BLUEBIRD-42` + `TypeScript` + `NestJS` PASS
- [ ] E09: `LOTUS-88` + `Java` + `Spring Boot`, **không có** `ORCHID-27`

---

## Phase 4: TODO 2/4 - Episodic Memory (85-105 phút)

### 4.1 Implement `retrieve_episodic`

**Code cần viết**:
```python
def retrieve_episodic(self, user_id: str, query: str) -> str:
    results = self.client.graph.search(
        user_id=user_id,
        query=cap_query(query),
        scope="episodes",
        limit=5,
    )
    return render_graph_search(results, episode_char_cap=180)
```

### 4.2 Test riêng episodic
```bash
docker compose run --rm app python -m src.evaluate \
  --impl student --reuse-seeded --only-layer episodic
```

### 4.3 Checklist Pha C
- [ ] Search bằng `user_id`, **không** `graph_id`
- [ ] `scope="episodes"`, `limit=5`
- [ ] E04: `ClientSession`, `concurrency=20`, `ASYNC-FIX-20` PASS
- [ ] E05: `connection churn`, `timeout threshold` PASS

---

## Phase 5: TODO 3/4 - Semantic Memory (105-125 phút)

### 5.1 Implement `retrieve_semantic`

**Code cần viết**:
```python
def retrieve_semantic(self, graph_id: str, query: str) -> str:
    try:
        results = self.client.graph.search(
            graph_id=graph_id,
            query=cap_query(query),
            scope="episodes",
            limit=8,
        )
    except Exception:
        # Fallback
        results = self.client.graph.search(
            graph_id=graph_id,
            query=cap_query(query),
            scope="nodes",
            limit=8,
        )
    return render_graph_search(results)
```

### 5.2 Test riêng semantic
```bash
docker compose run --rm app python -m src.evaluate \
  --impl student --reuse-seeded --only-layer semantic
```

### 5.3 Checklist Pha D
- [ ] Search `graph_id=semantic_graph_id`, **không** `user_id`
- [ ] E06: `Idempotency-Key`, `max-3-retries`, `exponential-backoff` PASS
- [ ] E11: `connection pooling`, `CONN-POOL-FIRST` PASS

---

## Phase 6: TODO 4/4 - Assembly + Router (125-145 phút)

### 6.1 Implement `assemble_context`

**Code cần viết**:
```python
def assemble_context(self, layers: dict[str, str]) -> tuple[str, dict[str, dict[str, int]]]:
    return self.budget.assemble(layers)
```

### 6.2 Full benchmark
```bash
docker compose run --rm app python -m src.evaluate --impl no_memory
docker compose run --rm app python -m src.evaluate --impl student --reuse-seeded
docker compose run --rm app python -m src.compare_reports
```

### 6.3 Checklist Pha E
- [ ] E07 PASS: retrieved có cả `Python` và `Idempotency-Key`
- [ ] Có `reports/benchmark.md`, `reports/benchmark_no_memory.md`, `reports/comparison.md`

---

## Phase 7: Mini-drill Control Plane (145-155 phút)

### 7.1 Đọc control plane files
- `control_plane/AGENTS.md`
- `control_plane/CONTEXT_LAYERS.md`
- `control_plane/SOUL.md`
- `control_plane/MEMORY.md`
- `control_plane/MEMORY_SCHEMA.md`
- `control_plane/TASKS.md`

### 7.2 Demo scripts
```bash
docker compose run --rm app python -m src.episodic_maintenance
docker compose run --rm app python -m src.heartbeat --dry-run
docker compose run --rm app python -m src.compiled_kb --reset
```

---

## Phase 8: Privacy Drill (155-170 phút)

### 8.1 Chạy benchmark TRƯỚC khi xóa
```bash
# Đảm bảo đã có benchmark.md
```

### 8.2 Forget user
```bash
docker compose run --rm app python -m src.forget --user-id minh-lab17
```

### 8.3 Verify
```bash
docker compose run --rm app python -m src.forget \
  --user-id minh-lab17 --verify-only
```

### 8.4 Screenshot 2 lệnh trên

### 8.5 Re-seed nếu cần golden
```bash
docker compose run --rm app python -m src.seed
```

---

## Phase 9: 60 phút cuối (Golden + UI)

### 9.1 Golden set (nếu có)
```bash
# Copy data/golden_eval.json (giảng viên phát lúc phút 110)
docker compose run --rm app python -m src.evaluate \
  --impl student --reuse-seeded --golden
```

### 9.2 UI Demo (bonus)
```bash
# GEMINI_API_KEY trong .env
make ui  # http://localhost:8501
```

---

## Phase 10: Submission

### 10.1 Artefact bắt buộc (thiếu = fail)
1. `src/memory_student.py` - 4 ham hoàn thành
2. `reports/benchmark.md` + `reports/benchmark.json` - từ `--impl student`
3. `README_submission.md` (<=400 từ) gồm:
   - 3 câu mục 5.2 (layer quan trọng / trade-off / poisoning)
   - 4 câu phân tích benchmark
   - 2-4 câu về E08 recency và E10 compaction

### 10.2 Artefact minh chứng (6d)
4. `reports/comparison.md`
5. `reports/benchmark_no_memory.md`
6. Screenshots: `long_term.png`, `episodic.png`, `semantic.png`, `privacy.png`

### 10.3 Artefact bonus
7. `reports/golden_benchmark.json` + `reports/golden_benchmark.md`
8. UI: `src/demo_ui.py` + screenshot/video

### 10.4 Không được nộp
- `.env`, `ZEP_API_KEY`, `data/golden_eval.json`
- `src/memory_reference.py` đổi tên

---

## Bảng điểm tóm tắt

| Khoi | Diem | Nguon cham |
|------|------|------------|
| 11 case E01-E11 | 56 | reports/benchmark.json |
| Privacy drill | 6 | screenshot |
| 4 cau + comparison | 6 | README_submission.md |
| 3 cau thuc hanh | 6 | README_submission.md |
| Artefact | 6 | repo |
| **Nen** | **80** | |
| Golden 20/20 | +10 | reports/golden_benchmark.json |
| UI demo | +10 | src/demo_ui.py |
| **Tong** | **100** | |

---

## Dependencies giữa các phase

```
Phase 1 (smoke + seed)
    ↓
Phase 2 (quan sat STM) ← independent
    ↓
Phase 3 (TODO 1) ← cần seed xong
    ↓
Phase 4 (TODO 2) ← cần seed xong
    ↓
Phase 5 (TODO 3) ← cần seed xong
    ↓
Phase 6 (TODO 4 + full benchmark)
    ↓
Phase 7 (control plane) ← independent
    ↓
Phase 8 (privacy) ← cần benchmark xong
    ↓
Phase 9 (golden + UI)
    ↓
Phase 10 (submission)
```

---

## Checklist trước khi nộp

- [ ] `pytest -q` pass
- [ ] `benchmark.json` có 11 cases
- [ ] Hit rate >= 9/11 (80%)
- [ ] 4 hàm trong `memory_student.py` không còn `NotImplementedError`
- [ ] `README_submission.md` <= 400 từ
- [ ] 4 screenshot (long_term, episodic, semantic, privacy)
- [ ] Không commit secret (.env, ZEP_API_KEY, golden_eval.json)
