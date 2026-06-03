# CHOICES.md — Three Key Design Decisions

---

## Decision 1: Detection Model — YOLOv8n

### Options Considered

| Model | Speed (CPU, 1080p) | Person recall | ReID support | Notes |
|-------|-------------------|---------------|--------------|-------|
| YOLOv8n | ~12 FPS | Good (COCO mAP50 ~0.52) | Via Ultralytics | Easy setup, well-maintained |
| YOLOv8s | ~6 FPS | Better (+5% mAP) | Via Ultralytics | 2x slower on CPU |
| RT-DETR | ~3 FPS CPU | Excellent | External | Transformer-based, slow on CPU |
| MediaPipe Pose | ~20 FPS | Person only (no bbox quality) | None | No bounding box confidence for tracker |
| YOLO-NAS | ~8 FPS | Best in class | Via SuperGradients | Complex setup, less ecosystem support |

### What AI Suggested

When I asked an LLM to rank these for a retail CCTV use case (1080p, 15fps input, CPU-only server in a container), it recommended:

> "YOLOv8s would give you a better accuracy/speed balance for crowded retail scenes. The nano model may struggle with partially occluded people at distance. If you need real-time performance on CPU, use frame skipping (every 3rd frame at 15fps = 5 effective fps) to keep latency acceptable."

The AI also suggested RT-DETR for best accuracy but flagged its CPU performance as a problem.

### What I Chose and Why

**YOLOv8n with frame_skip=3** (5 effective FPS).

My reasoning:
1. At 5 effective FPS, entry/exit events can still be reliably detected — people take 2–5 seconds to cross a threshold.
2. YOLOv8n runs at ~40ms/frame on CPU at 1080p after torch optimisation. At frame_skip=3, this is comfortably under the 200ms budget for real-time processing.
3. The AI's suggestion of YOLOv8s is correct for production deployment with a GPU. Without a GPU in the submission container, the nano model is the pragmatic choice.
4. Partial occlusion handling: I addressed this through the two-pass ByteTracker (low-confidence detections still tracked in pass 2) rather than relying on model accuracy alone.

**What I would change for production**: Switch to YOLOv8s on a T4 GPU instance. The accuracy gain on partially occluded people in crowded billing queues is worth the compute cost.

---

## Decision 2: Event Schema Design

### Options Considered

**Option A** — Flat schema, no metadata field:
All fields at the top level. Simpler to query but schema changes (adding queue_depth) require table migrations.

**Option B** — Fully nested, everything in metadata:
More flexible but harder to index and query. SQL `json_extract()` calls are slower than column access.

**Option C** — Hybrid (chosen): Fixed typed columns for queryable fields, `metadata` for event-type-specific extras:
```json
{
  "event_id": "...", "store_id": "...", "visitor_id": "...",
  "event_type": "...", "timestamp": "...",
  "zone_id": "...", "dwell_ms": 0, "is_staff": false, "confidence": 0.91,
  "metadata": {"queue_depth": null, "sku_zone": "...", "session_seq": 1}
}
```

### What AI Suggested

I asked an LLM to evaluate the three options. It recommended Option C (hybrid), noting:

> "Put all fields that appear in WHERE clauses or aggregations as top-level columns with database indexes. Fields that are event-type-specific and unlikely to be filtered on (queue_depth, sku_zone) belong in metadata. This mirrors how Kafka schema registries + columnar stores like Parquet are typically designed."

It also suggested adding a `session_id` field to directly link events without joining through visitor_id + timestamp. 

### What I Chose and Why

**Option C (hybrid)**, but I **disagreed with the session_id suggestion**.

My reasoning against explicit session_id:
1. The pipeline doesn't have a reliable session boundary when processing video — sessions only become clearly defined when an EXIT event fires. Using visitor_id as the session key is simpler and avoids the complexity of session token management across camera restarts.
2. Session tracking is maintained server-side in the `sessions` table, keyed on `store_id:visitor_id:entry_ts`. This achieves the same query performance without adding a new field to every event.

The AI was right about the hybrid approach. The `metadata` JSON blob for sku_zone and session_seq is a deliberate choice — these fields are informational, not frequently filtered, and can evolve without schema migrations.

**One thing I would add for production**: A `frame_offset_ms` field at the top level to support video timestamp debugging without parsing the ISO timestamp.

---

## Decision 3: API Storage Engine — SQLite vs PostgreSQL

### Options Considered

| Storage | Pros | Cons | Setup complexity |
|---------|------|------|-----------------|
| SQLite (WAL) | Zero config, `docker compose up` = 1 container, ACID-compliant | Single writer, no horizontal scaling, ~10GB practical limit | None |
| PostgreSQL | Concurrent writes, full SQL, proven at scale | Requires separate container, connection pooling | Medium |
| PostgreSQL + TimescaleDB | Auto-partitioning on time columns, fast range queries, compression | TimescaleDB license (Apache 2 for core, TSL for enterprise features) | High |
| Redis + SQLite | Redis for real-time counters, SQLite for event log | Two datastores to manage, consistency issues | Medium |

### What AI Suggested

The AI recommended PostgreSQL + TimescaleDB without hesitation:

> "For a time-series event store where you'll be doing rolling 7-day aggregations and range queries by store_id + timestamp, TimescaleDB's automatic hypertable partitioning and continuous aggregates will give you 10–100x faster queries at scale compared to a standard SQLite B-tree index on timestamp."

### What I Chose and Why

**SQLite with WAL mode** for the submission, with PostgreSQL as the documented upgrade path.

**Rationale**:

1. **Acceptance gate requirement**: `docker compose up` must start everything. Adding PostgreSQL means a multi-service compose file, a health check dependency chain, and init scripts. SQLite keeps the acceptance gate simple and deterministic.

2. **Data volume**: At 5 stores × 3 cameras × 20 minutes @ 5 effective FPS ≈ 90,000 frames. With typical detection rates (0-8 people/frame), this produces ~50,000-200,000 events — comfortably within SQLite's sweet spot.

3. **WAL mode specifics**: SQLite WAL mode allows concurrent readers (all GET endpoints) to run without blocking the single writer (POST /events/ingest). This matches the read-heavy, batch-write access pattern of this system.

4. **Upgrade path**: The `get_connection()` context manager in `database.py` is the only abstraction boundary that would need to change to switch to asyncpg + PostgreSQL. The rest of the application uses standard SQL that runs on both engines.

**What would make me change this decision**: If the system were to receive live streaming events from 40 stores simultaneously at 15fps (40 × 3 cameras × 15fps = 1,800 detection cycles per second), the single-writer limitation of SQLite would become a bottleneck. At that scale, switching to PostgreSQL with connection pooling (pgBouncer) and TimescaleDB hypertables is the right call.
