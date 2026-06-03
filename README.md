# Store Intelligence API — Apex Retail

End-to-end retail analytics from CCTV footage to live metrics API.

## Quick Start (5 commands)

```bash
# 1. Clone and enter the project
git clone <your-repo-url> store-intelligence
cd store-intelligence

# 2. Generate sample data (POS transactions + sample events)
python data/generate_sample_data.py

# 3. Start the API + Dashboard
docker compose up --build

# 4. Verify the API is healthy
curl http://localhost:8000/health

# 5. Open the live dashboard
open http://localhost:3000
# Or visit http://localhost:3000 in your browser
```

The API is available at **http://localhost:8000**  
The live dashboard is at **http://localhost:3000**  
Interactive API docs: **http://localhost:8000/docs**

---

## Running the Detection Pipeline

### With real CCTV clips

Place your video clips in `data/clips/` following the naming convention:
```
data/clips/STORE_BLR_002_CAM_ENTRY_01.mp4
data/clips/STORE_BLR_002_CAM_FLOOR_01.mp4
data/clips/STORE_BLR_002_CAM_BILLING_01.mp4
```

Install pipeline dependencies:
```bash
pip install -r requirements-pipeline.txt
```

Run the detection pipeline:
```bash
# Process all stores and feed events directly to the running API
API_URL=http://localhost:8000 bash pipeline/run.sh

# Process a single store
python -m pipeline.detect \
  --video-dir data/clips \
  --store-id STORE_BLR_002 \
  --api-url http://localhost:8000 \
  --output data/events_output.jsonl
```

### Without clips (mock mode)

The pipeline runs in mock mode when video files are not found — it generates plausible synthetic detections for testing the full pipeline:

```bash
python -m pipeline.detect \
  --video-dir data/clips \
  --api-url http://localhost:8000 \
  --output data/events_output.jsonl
```

### Replay pre-processed events

If you have a `events_output.jsonl` file from a previous pipeline run:
```bash
# Batch ingest all events
python -c "
import json, requests
events = [json.loads(l) for l in open('data/events_output.jsonl') if l.strip()]
# Send in batches of 500
for i in range(0, len(events), 500):
    batch = events[i:i+500]
    r = requests.post('http://localhost:8000/events/ingest', json={'events': batch})
    print(f'Batch {i//500+1}: {r.json()[\"accepted\"]} accepted')
"
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/events/ingest` | Ingest batch of up to 500 events (idempotent) |
| GET | `/stores/{id}/metrics` | Unique visitors, conversion rate, dwell, queue |
| GET | `/stores/{id}/funnel` | Entry → Zone → Billing → Purchase funnel |
| GET | `/stores/{id}/heatmap` | Zone visit frequency, normalised 0–100 |
| GET | `/stores/{id}/anomalies` | Active anomalies with severity + suggested action |
| GET | `/health` | Service status, per-store feed lag |

**Store IDs**: `STORE_BLR_001`, `STORE_BLR_002`, `STORE_MUM_001`, `STORE_DEL_001`, `STORE_HYD_001`

### Example requests

```bash
# Get metrics for a store
curl http://localhost:8000/stores/STORE_BLR_002/metrics | python -m json.tool

# Check for anomalies
curl http://localhost:8000/stores/STORE_BLR_002/anomalies | python -m json.tool

# Ingest a test event
curl -X POST http://localhost:8000/events/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "events": [{
      "event_id": "550e8400-e29b-41d4-a716-446655440000",
      "store_id": "STORE_BLR_002",
      "camera_id": "CAM_ENTRY_01",
      "visitor_id": "VIS_test01",
      "event_type": "ENTRY",
      "timestamp": "2026-03-03T14:22:10Z",
      "zone_id": "ENTRY_THRESHOLD",
      "dwell_ms": 0,
      "is_staff": false,
      "confidence": 0.91,
      "metadata": {"queue_depth": null, "sku_zone": null, "session_seq": 1}
    }]
  }'
```

---

## Running Tests

```bash
# Install dependencies
pip install -r requirements.txt

# Run all tests with coverage
pytest tests/ --cov=app --cov=pipeline --cov-report=term-missing -v

# Run a specific test file
pytest tests/test_metrics.py -v

# Run the assertion suite against a live API
python tests/assertions.py http://localhost:8000
```

---

## Project Structure

```
store-intelligence/
├── pipeline/
│   ├── detect.py        # Main detection + tracking script
│   ├── tracker.py       # ByteTracker + Re-ID logic
│   ├── emit.py          # Event schema + emission
│   └── run.sh           # One command to process all clips
├── app/
│   ├── main.py          # FastAPI entrypoint
│   ├── models.py        # Pydantic event schema
│   ├── database.py      # SQLite setup + connection management
│   ├── ingestion.py     # Ingest, validate, deduplicate
│   ├── metrics.py       # Real-time metric computation
│   ├── funnel.py        # Funnel + session logic
│   ├── heatmap.py       # Zone heatmap computation
│   ├── anomalies.py     # Anomaly detection
│   ├── health.py        # Health endpoint
│   └── logging_config.py # Structured JSON logging
├── tests/
│   ├── test_pipeline.py # Detection + tracker tests (with AI prompt block)
│   ├── test_metrics.py  # API endpoint tests (with AI prompt block)
│   ├── test_anomalies.py # Anomaly detection tests (with AI prompt block)
│   └── assertions.py    # 10 acceptance assertions
├── dashboard/
│   └── index.html       # Live web dashboard
├── data/
│   ├── store_layout.json         # Zone definitions for 5 stores
│   ├── generate_sample_data.py   # Generate POS + sample events
│   ├── pos_transactions.csv      # (generated)
│   └── sample_events.jsonl       # (generated)
├── docs/
│   ├── DESIGN.md        # Architecture + AI-assisted decisions
│   └── CHOICES.md       # 3 key decisions with full reasoning
├── docker-compose.yml
├── Dockerfile
├── nginx.conf
├── requirements.txt
└── README.md
```

---

## Architecture Summary

- **Detection**: YOLOv8n person detection + ByteTrack-inspired IoU tracker + trajectory Re-ID
- **Events**: JSONL stream → batch POST to `/events/ingest`
- **Storage**: SQLite with WAL mode (single writer, concurrent readers)
- **API**: FastAPI with Pydantic v2 validation, structured JSON logging, graceful degradation
- **Dashboard**: Vanilla JS SPA polling the API every 10s, served by nginx

See `docs/DESIGN.md` for full architecture detail and `docs/CHOICES.md` for decision rationale.

---

## Live Dashboard

After `docker compose up`, visit **http://localhost:3000** to see:
- Real-time KPI cards (visitors, conversion rate, avg dwell, queue depth)
- Conversion funnel (Entry → Zone → Billing → Purchase)
- Zone heatmap (visit frequency normalised 0–100)
- Active anomalies list (queue spike, conversion drop, dead zones)

The dashboard auto-refreshes every 10 seconds. Use the store selector to switch between stores.
