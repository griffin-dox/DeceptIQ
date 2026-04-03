# DeceptIQ — MVP Middleware Architecture Plan (24-Hour Sprint)
## Team Cyfer Trace — HoneyCloud with AI Brain

**Last Updated:** 30 March 2026  
**Status:** LOCKED — Ready for implementation

---

## 1. What We Decided (Summary of All Tradeoffs)

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Log source | Wazuh alerts.json (not raw honeypot logs) | Clean demo story + Wazuh stays in the pipeline |
| Database | PostgreSQL (Docker container) | Reliability, reproducibility, matches proposal |
| Middleware deployment | Single Python monolith on Wazuh VM | No extra Docker orchestration, fast to debug |
| Real-time mechanism | Polling (5-sec interval) | SSE/WebSocket adds complexity, polling is invisible in demo |
| ML models | XGBoost + Isolation Forest now, LSTM hot-pluggable | LSTM drops in when ready, no code changes needed |
| GenAI | Gemini Flash 2.0 via Vertex AI + Ollama fallback | Both already available on GCP |
| Deception delivery | Dynamic on Flask website only | Cowrie + OpenCanary pre-loaded with static bait |
| Docker automation | Start/stop honeypots as deception tactic | Adapts attack surface based on attacker behavior |
| Wazuh integration | Bidirectional — read alerts.json, write predictions back | DeceptIQ as a SIEM extension, not a replacement |
| Dashboard | Two dashboards — Wazuh (native alerts) + React (rich ML data) | Different audiences, complementary views |
| Red team script | Needs to be built for demo | Simulates attacks to generate live pipeline data |

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    HONEYPOT VM                          │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────┐    │
│  │ Cowrie   │  │ OpenCanary   │  │ Custom Flask   │    │
│  │ SSH      │  │ SQL/HoneyDB  │  │ Website        │    │
│  │ (static  │  │ (static      │  │ (DYNAMIC       │    │
│  │  bait)   │  │  bait)       │  │  deception)    │    │
│  └────┬─────┘  └──────┬───────┘  └───────┬────────┘    │
│       │               │                  │              │
│       └───────────┬───┴──────────────────┘              │
│                   │                                      │
│            Wazuh Agent                                   │
│         (collects all logs)                              │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│                    WAZUH VM                              │
│                                                          │
│  ┌────────────────────────────────┐                      │
│  │ Wazuh Server/Manager/Dashboard │◄─── Prediction log   │
│  │ alerts.json output             │     fed back in      │
│  └───────────────┬────────────────┘          ▲           │
│                  │                           │           │
│                  ▼                           │           │
│  ┌────────────────────────────────────────────────────┐  │
│  │           DeceptIQ MIDDLEWARE (Python)              │  │
│  │                                                     │  │
│  │  ┌──────────┐ ┌──────────┐ ┌────────────────────┐  │  │
│  │  │ Log      │→│ Feature  │→│ ML Engine          │  │  │
│  │  │ Ingestion│ │ Engineer │ │ XGBoost + IF + LSTM│  │  │
│  │  └──────────┘ └──────────┘ └─────────┬──────────┘  │  │
│  │                                      │              │  │
│  │                         ┌────────────┼────────────┐ │  │
│  │                         ▼            ▼            ▼ │  │
│  │                    ┌─────────┐ ┌──────────┐ ┌─────┐ │  │
│  │                    │ Threat  │ │Deception │ │Write│ │  │
│  │                    │ Intel   │ │ Engine   │ │Pred │─┘  │
│  │                    │AbuseIPDB│ │Gemini/   │ │Log  │    │
│  │                    │+ OTX   │ │Ollama    │ └─────┘    │
│  │                    └─────────┘ └──────────┘           │
│  │                                                       │
│  │  ┌──────────────────────────────────────────────┐     │
│  │  │ Flask REST API (serves React dashboard)      │     │
│  │  │ + Session Manager + STIX Export              │     │
│  │  └──────────────────────────────────────────────┘     │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  ┌──────────────┐                                        │
│  │ PostgreSQL   │  (Docker container)                    │
│  │ deceptiq DB  │                                        │
│  └──────────────┘                                        │
└──────────────────────────────────────────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │   React Dashboard     │
        │   (polls Flask API)   │
        └───────────────────────┘
```

---

## 3. Bidirectional Wazuh Integration (Key Innovation)

This is what makes DeceptIQ a SIEM extension, not a standalone tool.

### Direction 1: Wazuh → Middleware (reading alerts)

Wazuh stores all processed alerts at:
```
/var/ossec/logs/alerts/alerts.json
```

Each line is a JSON object. The middleware polls this file every 5 seconds for new lines. The raw honeypot data is preserved inside the Wazuh alert under the `data` or `full_log` field, so we get Wazuh's structured format wrapping the original honeypot data our models need.

### Direction 2: Middleware → Wazuh (writing predictions back)

The middleware writes ML predictions to a custom log file:
```
/var/log/deceptiq/predictions.json
```

Each line is a JSON object:
```json
{
  "timestamp": "2026-03-30T14:23:01Z",
  "source": "deceptiq",
  "alert_type": "ml_prediction",
  "source_ip": "45.33.12.87",
  "attack_type": "brute_force",
  "severity_score": 0.85,
  "model": "xgboost",
  "confidence": 0.94,
  "isolation_forest_anomaly": false,
  "lstm_stage": "initial_access",
  "abuseipdb_score": 87,
  "abuseipdb_reports": 142,
  "otx_pulse_count": 3,
  "session_id": "sess_45.33.12.87_2026-03-30T14:20:00Z",
  "deception_action": "served_fake_credentials",
  "deception_model": "gemini_flash"
}
```

### Wazuh Configuration (add to ossec.conf on the Wazuh VM agent)

```xml
<!-- Monitor DeceptIQ prediction logs -->
<localfile>
  <log_format>json</log_format>
  <location>/var/log/deceptiq/predictions.json</location>
  <label key="source">deceptiq</label>
</localfile>
```

### Custom Wazuh Decoder (save as /var/ossec/etc/decoders/deceptiq_decoder.xml)

```xml
<decoder name="deceptiq">
  <prematch>^{"timestamp".*"source":"deceptiq"</prematch>
  <plugin_decoder>JSON_Decoder</plugin_decoder>
</decoder>
```

### Custom Wazuh Rules (save as /var/ossec/etc/rules/deceptiq_rules.xml)

```xml
<group name="deceptiq,">

  <!-- Base rule for all DeceptIQ events -->
  <rule id="100100" level="3">
    <decoded_as>deceptiq</decoded_as>
    <description>DeceptIQ ML prediction event</description>
  </rule>

  <!-- High severity attack detected -->
  <rule id="100101" level="12">
    <if_sid>100100</if_sid>
    <field name="severity_score">^0\.[7-9]|^1\.0</field>
    <description>DeceptIQ: High severity attack detected - $(attack_type) from $(source_ip) [confidence: $(confidence)]</description>
  </rule>

  <!-- Medium severity -->
  <rule id="100102" level="8">
    <if_sid>100100</if_sid>
    <field name="severity_score">^0\.[4-6]</field>
    <description>DeceptIQ: Medium severity - $(attack_type) from $(source_ip)</description>
  </rule>

  <!-- Known malicious IP (AbuseIPDB score > 70) -->
  <rule id="100103" level="13">
    <if_sid>100100</if_sid>
    <field name="abuseipdb_score">^[7-9][0-9]$|^100$</field>
    <description>DeceptIQ: Known malicious IP $(source_ip) - AbuseIPDB score $(abuseipdb_score)</description>
  </rule>

  <!-- Deception triggered -->
  <rule id="100104" level="6">
    <if_sid>100100</if_sid>
    <field name="deception_action">\S+</field>
    <description>DeceptIQ: Deception action taken - $(deception_action) against $(source_ip)</description>
  </rule>

</group>
```

**Result:** ML predictions appear as native Wazuh alerts in the Wazuh dashboard with proper severity levels. SOC analysts see them alongside standard alerts without any workflow change.

---

## 4. PostgreSQL Setup

Single Docker command on the Wazuh VM:

```bash
docker run -d \
  --name deceptiq-db \
  --restart always \
  -e POSTGRES_DB=deceptiq \
  -e POSTGRES_USER=deceptiq_user \
  -e POSTGRES_PASSWORD=<CHANGE_THIS_PASSWORD> \
  -p 5432:5432 \
  -v deceptiq_pgdata:/var/lib/postgresql/data \
  postgres:16
```

### Database Schema

```sql
-- Run after PostgreSQL is up:
-- psql -h localhost -U deceptiq_user -d deceptiq -f schema.sql

-- 1. Raw events from Wazuh alerts
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ NOT NULL,
    source_ip VARCHAR(45),
    dest_ip VARCHAR(45),
    dest_port INTEGER,
    honeypot_type VARCHAR(50),        -- 'cowrie', 'opencanary', 'web'
    wazuh_rule_id INTEGER,
    wazuh_rule_level INTEGER,
    wazuh_rule_description TEXT,
    raw_log JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2. Attacker sessions
CREATE TABLE sessions (
    id SERIAL PRIMARY KEY,
    session_id VARCHAR(100) UNIQUE,
    source_ip VARCHAR(45) NOT NULL,
    start_time TIMESTAMPTZ,
    end_time TIMESTAMPTZ,
    honeypot_type VARCHAR(50),
    status VARCHAR(20) DEFAULT 'active',
    event_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. ML predictions
CREATE TABLE predictions (
    id SERIAL PRIMARY KEY,
    event_id INTEGER REFERENCES events(id),
    session_id VARCHAR(100),
    model_name VARCHAR(50),
    prediction VARCHAR(50),
    confidence FLOAT,
    attack_type VARCHAR(100),
    severity_score FLOAT,
    raw_output JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 4. Threat intel enrichment cache
CREATE TABLE threat_intel (
    id SERIAL PRIMARY KEY,
    source_ip VARCHAR(45) UNIQUE NOT NULL,
    abuseipdb_score INTEGER,
    abuseipdb_reports INTEGER,
    abuseipdb_country VARCHAR(10),
    abuseipdb_isp VARCHAR(255),
    otx_pulse_count INTEGER,
    otx_campaigns JSONB,
    otx_malware_families JSONB,
    last_checked TIMESTAMPTZ DEFAULT NOW(),
    raw_response JSONB
);

-- 5. Deception actions log
CREATE TABLE deception_actions (
    id SERIAL PRIMARY KEY,
    session_id VARCHAR(100),
    trigger_reason VARCHAR(100),
    action_type VARCHAR(50),
    content_served TEXT,
    attacker_response TEXT,
    genai_model VARCHAR(50),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 6. Attacker profiles (aggregated)
CREATE TABLE attacker_profiles (
    id SERIAL PRIMARY KEY,
    source_ip VARCHAR(45) UNIQUE NOT NULL,
    first_seen TIMESTAMPTZ,
    last_seen TIMESTAMPTZ,
    total_sessions INTEGER DEFAULT 0,
    total_events INTEGER DEFAULT 0,
    tools_detected JSONB,
    techniques_mitre JSONB,
    skill_level VARCHAR(20),
    behavior_pattern VARCHAR(100),
    threat_score FLOAT,
    abuseipdb_score INTEGER,
    otx_associations JSONB,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 7. STIX reports
CREATE TABLE stix_reports (
    id SERIAL PRIMARY KEY,
    session_id VARCHAR(100),
    source_ip VARCHAR(45),
    stix_bundle JSONB,
    generated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_events_source_ip ON events(source_ip);
CREATE INDEX idx_events_timestamp ON events(timestamp);
CREATE INDEX idx_sessions_source_ip ON sessions(source_ip);
CREATE INDEX idx_sessions_status ON sessions(status);
CREATE INDEX idx_predictions_session ON predictions(session_id);
CREATE INDEX idx_threat_intel_ip ON threat_intel(source_ip);
```

---

## 5. Middleware — File Structure

```
deceptiq-middleware/
├── app.py                       # Flask app + main polling loop
├── config.py                    # All configuration
├── requirements.txt             # Python dependencies
├── schema.sql                   # Database schema
│
├── services/
│   ├── __init__.py
│   ├── log_ingestion.py         # Read + parse Wazuh alerts.json
│   ├── feature_engineering.py   # Transform alerts → model features
│   ├── ml_engine.py             # XGBoost + IF + LSTM predictions
│   ├── threat_intel.py          # AbuseIPDB + AlienVault OTX
│   ├── deception_engine.py      # Gemini/Ollama content gen + Docker control
│   ├── session_manager.py       # Group events by attacker session
│   ├── stix_export.py           # Generate STIX 2.1 reports
│   └── prediction_writer.py     # Write predictions back for Wazuh
│
├── models/                      # Drop .pkl and .pt files here
│   ├── xgboost_model.pkl
│   ├── isolation_forest.pkl
│   ├── lstm_model.pt            # (when ready — hot-pluggable)
│   └── encoders/
│       ├── type_encoder.pkl     # Label encoders from training
│       ├── country_encoder.pkl
│       └── scaler.pkl           # StandardScaler if used
│
├── wazuh_config/                # Copy these to Wazuh directories
│   ├── deceptiq_decoder.xml
│   ├── deceptiq_rules.xml
│   └── ossec_localfile.conf     # Snippet to add to ossec.conf
│
└── scripts/
    └── red_team_sim.py          # Attack simulation for demo
```

---

## 6. Service Details

### 6.1 Log Ingestion (log_ingestion.py)

Polls `/var/ossec/logs/alerts/alerts.json` every 5 seconds. Tracks file position to only read new lines.

**Key logic:**
- Parse each JSON line from Wazuh alert
- Identify honeypot source from agent name, rule ID, or log content
- Extract source IP, destination port, timestamp, protocol
- Store raw event in PostgreSQL `events` table
- Return parsed event for the ML pipeline

**Honeypot identification rules:**
```
'cowrie' in log content OR rule relates to SSH  → honeypot_type = 'cowrie'
'opencanary' in log content OR MySQL/SQL refs   → honeypot_type = 'opencanary'
HTTP/Flask/web references                       → honeypot_type = 'web'
```

### 6.2 Feature Engineering (feature_engineering.py)

Transforms Wazuh alerts into feature vectors matching what models were trained on.

**Critical requirement:** You MUST export label encoders and scalers from your training notebooks:
```python
# In your training notebook, after training:
import joblib
joblib.dump(label_encoder_type, 'encoders/type_encoder.pkl')
joblib.dump(label_encoder_country, 'encoders/country_encoder.pkl')
joblib.dump(scaler, 'encoders/scaler.pkl')
```

**Feature mapping (Wazuh alert → model input):**
```
Wazuh Field                     → Model Feature
───────────────────────────────────────────────
data.srcip                      → src (encoded)
data.srcport / srcPort          → src_port
data.dstport / dstPort          → dest_port
timestamp                       → datetime (encoded)
GeoIP(data.srcip).latitude      → latitude
GeoIP(data.srcip).longitude     → longitude
GeoIP(data.srcip).country       → country (encoded)
data.protocol / connection type → type (encoded)
```

**For LSTM (when ready):** Collects sequences of events per session, transforms into fixed-length padded arrays of action vectors.

### 6.3 ML Engine (ml_engine.py)

Loads models and runs prediction pipeline:

```
Event arrives
    │
    ├── XGBoost (signature): "Is this a known attack?" (~5-10ms)
    │   Returns: prediction (0/1), confidence, attack_type
    │
    ├── Isolation Forest (anomaly): "Is this unusual?" (~2-5ms)
    │   Returns: is_anomaly (bool), anomaly_score
    │
    └── LSTM (behavioral, if available): "What stage?" (~50-100ms)
        Requires: 3+ events in session
        Returns: kill_chain_stage, threat_score
    │
    ▼
Combined Output:
    - attack_type (from XGBoost or LSTM)
    - severity_score (weighted: XGB 0.4, IF 0.3, LSTM 0.3)
    - deception_trigger (true if severity > 0.6)
    - model outputs (stored individually)
```

**LSTM hot-plug:** The engine checks for `lstm_model.pt` on startup. If not found, runs XGBoost + IF only. When you drop the file in and restart, LSTM activates automatically.

### 6.4 Threat Intel (threat_intel.py)

Runs for every new source IP. Cached in PostgreSQL to avoid rate limits.

**AbuseIPDB** (free tier: 1,000 checks/day):
- Endpoint: `GET https://api.abuseipdb.com/api/v2/check`
- Returns: confidence score (0-100), total reports, country, ISP

**AlienVault OTX** (10,000 requests/hour):
- Endpoint: `GET https://otx.alienvault.com/api/v1/indicators/IPv4/{ip}/general`
- Returns: pulse count, campaign names, malware family associations

**Cache TTL:** 1 hour. If IP was checked within the last hour, use cached result.

### 6.5 Deception Engine (deception_engine.py)

Two responsibilities:

**A) Content Generation (for Flask website)**
- Triggered when ML engine sets `deception_trigger: true`
- Calls Gemini Flash 2.0 via Vertex AI with attack context
- Generates: fake credentials, fake error messages, fake partial data access
- Falls back to Ollama if Vertex AI fails
- Exposes endpoint: `GET /api/deception/content?session_id=X&context=Y`
- Flask website calls this endpoint before rendering pages to attacker

**B) Docker Honeypot Control (attack surface adaptation)**
- Can start/stop additional honeypot containers based on attacker behavior
- Example: attacker probing web → spin up additional fake API endpoints
- Example: attacker found SSH → expose a new "database" container
- Uses Docker SDK for Python (`docker` library) to manage containers
- This makes the attack surface dynamic without modifying existing honeypots

### 6.6 Session Manager (session_manager.py)

Groups events by attacker IP + time proximity.

**Rules:**
- Same source IP + events within 30 minutes = same session
- Gap > 30 minutes = new session
- Session ID format: `sess_{source_ip}_{start_timestamp}`
- Session expires after 30 minutes of no activity
- On session end → trigger STIX report generation

### 6.7 Prediction Writer (prediction_writer.py)

Writes ML prediction results to `/var/log/deceptiq/predictions.json` (one JSON object per line). Wazuh agent monitors this file and creates native alerts via custom decoder + rules.

### 6.8 STIX Export (stix_export.py)

Auto-generates STIX 2.1 bundle on session end. Uses `stix2` Python library.

**Bundle contents:**
- Indicator (attacker IP + observed patterns)
- Attack Pattern (MITRE ATT&CK techniques)
- Observed Data (event timeline)
- Threat Actor (profile from ML + threat intel)
- Relationships linking all objects

---

## 7. Flask API Routes (for React Dashboard)

```
# Real-time data
GET  /api/events/recent            # Last N events (dashboard polls this)
GET  /api/sessions/active          # Currently active attacker sessions
GET  /api/dashboard/stats          # Summary: total attacks, active sessions, etc.

# Attacker data
GET  /api/attackers                # All attacker profiles
GET  /api/attackers/<ip>           # Single attacker full profile
GET  /api/attackers/<ip>/timeline  # Full event timeline for an attacker

# ML predictions
GET  /api/predictions/recent       # Recent ML predictions with model details
GET  /api/predictions/session/<id> # All predictions for a specific session

# Threat intelligence
GET  /api/threatintel/<ip>         # AbuseIPDB + OTX data for an IP
GET  /api/threatintel/map          # All attacker IPs with geo coordinates (for map)

# Deception
GET  /api/deception/content        # Get dynamic fake content (called by Flask website)
GET  /api/deception/actions        # Recent deception actions taken
GET  /api/deception/session/<id>   # Deception actions for a session

# Docker control
POST /api/honeypots/start          # Start a new honeypot container
POST /api/honeypots/stop           # Stop a honeypot container
GET  /api/honeypots/status         # List running honeypot containers

# STIX export
GET  /api/stix/session/<id>        # Get STIX report for a completed session
POST /api/stix/generate/<id>       # Manually trigger STIX generation

# System
GET  /api/health                   # Middleware health + model status
```

**Dashboard polling:** React dashboard calls `/api/events/recent` and `/api/sessions/active` every 5 seconds. Response includes a `last_event_id` for pagination so it only fetches new data.

---

## 8. Deception Delivery — Per Honeypot Strategy

### Custom Flask Website (DYNAMIC — primary deception target)
- Website calls `GET /api/deception/content` from middleware before rendering
- Middleware returns contextual fake content based on session and attack stage
- Pages dynamically serve fake client lists, security reports, credentials, error messages
- Gemini generates content tailored to what the attacker has done so far

### Cowrie SSH (STATIC — pre-loaded before demo)
Pre-populate fake filesystem with:
- `/home/admin/.ssh/id_rsa` — fake SSH private key
- `/home/admin/.env` — fake DB credentials, API keys
- `/opt/client-portal/config.yml` — fake client configuration
- `/var/backups/client_db_dump.sql` — fake database backup
- `/etc/crontab` — shows "automated security scans" for believability

### OpenCanary SQL (STATIC — pre-loaded before demo)
Pre-load fake database tables:
- `clients` — fake client company names, contacts, contract values
- `vulnerability_assessments` — fake security findings per client
- `siem_credentials` — fake SIEM platform logins (honeytokens)
- `incident_reports` — fake security incident history
- `employee_accounts` — fake internal staff credentials

### Docker Automation (DYNAMIC — adaptive attack surface)
- ML engine detects attacker behavior pattern
- Middleware spins up/down Docker containers to present new targets
- Example: attacker scanning ports → expose a new "Redis" container
- Example: attacker found web portal → expose a "staging API" container
- All controlled via Docker SDK, triggered by deception engine

---

## 9. Complete Event Flow (End to End)

```
1.  Attacker SSH brute-forces Cowrie
2.  Cowrie logs to its JSON log file
3.  Wazuh Agent reads Cowrie log
4.  Wazuh Agent sends to Wazuh Manager
5.  Wazuh Manager applies rules, writes to alerts.json
6.  ════════ MIDDLEWARE KICKS IN ════════
7.  Log Ingestion polls alerts.json, finds new alert
8.  Session Manager: assigns to session (new or existing)
9.  Feature Engineering: transforms to model input vectors
10. ML Engine runs pipeline:
    ├── XGBoost: "SSH brute force" (confidence: 0.94)
    ├── Isolation Forest: "Not anomalous" (known pattern)
    └── LSTM: "Kill-chain: Initial Access" (if 3+ events)
11. Combined: severity=0.85, deception_trigger=TRUE
12. Parallel actions:
    ├── Threat Intel: AbuseIPDB=87, OTX="Mirai botnet"
    ├── Deception Engine: Gemini generates fake SSH creds
    │   Flask website now serves these if attacker visits
    └── Docker: optionally spin up new container as lure
13. Everything stored in PostgreSQL
14. Prediction Writer → /var/log/deceptiq/predictions.json
15. Wazuh Agent picks up prediction log
16. Custom decoder + rules → Native Wazuh alert created
17. Wazuh Dashboard: "DeceptIQ: Brute force from 45.x.x.x"
18. React Dashboard: polls API, shows rich attacker profile
19. Session ends (30 min timeout) → STIX report auto-generated
```

---

## 10. Configuration (config.py)

```python
import os

class Config:
    # Database
    DATABASE_URL = os.getenv(
        'DATABASE_URL',
        'postgresql://deceptiq_user:password@localhost:5432/deceptiq'
    )

    # Wazuh
    WAZUH_ALERTS_PATH = os.getenv(
        'WAZUH_ALERTS_PATH',
        '/var/ossec/logs/alerts/alerts.json'
    )

    # Prediction output (Wazuh reads this back)
    PREDICTION_LOG_PATH = os.getenv(
        'PREDICTION_LOG_PATH',
        '/var/log/deceptiq/predictions.json'
    )

    # Threat Intel API Keys
    ABUSEIPDB_API_KEY = os.getenv('ABUSEIPDB_API_KEY', '')
    OTX_API_KEY = os.getenv('OTX_API_KEY', '')

    # GenAI
    GCP_PROJECT_ID = os.getenv('GCP_PROJECT_ID', '')
    VERTEX_AI_LOCATION = os.getenv('VERTEX_AI_LOCATION', 'us-central1')
    OLLAMA_URL = os.getenv('OLLAMA_URL', 'http://localhost:11434')
    OLLAMA_MODEL = os.getenv('OLLAMA_MODEL', 'llama3')

    # ML Models (drop files in models/ directory)
    XGBOOST_MODEL_PATH = 'models/xgboost_model.pkl'
    ISOLATION_FOREST_PATH = 'models/isolation_forest.pkl'
    LSTM_MODEL_PATH = 'models/lstm_model.pt'
    ENCODERS_DIR = 'models/encoders/'

    # GeoIP
    GEOIP_DB_PATH = os.getenv('GEOIP_DB_PATH', 'models/GeoLite2-City.mmdb')

    # Session Management
    SESSION_TIMEOUT_MINUTES = 30
    POLL_INTERVAL_SECONDS = 5

    # Severity Weights
    XGBOOST_WEIGHT = 0.4
    ISOLATION_FOREST_WEIGHT = 0.3
    LSTM_WEIGHT = 0.3
    DECEPTION_TRIGGER_THRESHOLD = 0.6

    # Flask API
    API_HOST = '0.0.0.0'
    API_PORT = 5000
    CORS_ORIGINS = ['*']  # Restrict in production
```

---

## 11. Python Dependencies (requirements.txt)

```
# Core
flask==3.0.3
flask-cors==4.0.1
gunicorn==22.0.0

# Database
psycopg2-binary==2.9.9

# ML
scikit-learn==1.5.1
xgboost==2.0.3
torch==2.3.1
numpy==1.26.4
pandas==2.2.2
joblib==1.4.2

# GeoIP
geoip2==4.8.0

# Threat Intel
requests==2.32.3

# GenAI
google-cloud-aiplatform==1.60.0

# STIX
stix2==3.0.1

# Docker control
docker==7.1.0

# Utilities
python-dotenv==1.0.1
```

---

## 12. Implementation Order (Step by Step)

### Phase 1: Foundation (Hours 1-4)
1. ☐ Start PostgreSQL Docker container, run schema.sql
2. ☐ Create directory structure and config.py
3. ☐ Build log_ingestion.py — read and parse Wazuh alerts.json
4. ☐ Build session_manager.py — group events by IP
5. ☐ Build prediction_writer.py — write to prediction log
6. ☐ **TEST:** Trigger a Cowrie login → verify event appears in PostgreSQL

### Phase 2: ML Integration (Hours 4-8)
7. ☐ Export model .pkl files + encoders from training notebooks
8. ☐ Build feature_engineering.py — map Wazuh fields to model features
9. ☐ Build ml_engine.py — load models, run prediction pipeline
10. ☐ **TEST:** Trigger attack → verify prediction in DB + prediction log

### Phase 3: Intelligence + Deception (Hours 8-14)
11. ☐ Build threat_intel.py — AbuseIPDB + OTX with caching
12. ☐ Build deception_engine.py — Gemini content gen + Docker control
13. ☐ Connect Flask website to deception content endpoint
14. ☐ **TEST:** Full pipeline — attack → prediction → enrichment → deception served

### Phase 4: Wazuh Integration (Hours 14-16)
15. ☐ Install Wazuh decoder + rules XML files
16. ☐ Add localfile config for prediction log
17. ☐ Restart Wazuh Manager
18. ☐ **TEST:** ML alerts appear in Wazuh dashboard as native alerts

### Phase 5: API + Dashboard (Hours 16-20)
19. ☐ Build Flask REST API with all routes
20. ☐ Connect React dashboard to API endpoints
21. ☐ Build STIX export on session end
22. ☐ **TEST:** React dashboard shows live data

### Phase 6: Demo Prep (Hours 20-24)
23. ☐ Build red_team_sim.py attack simulation script
24. ☐ Pre-load Cowrie fake filesystem
25. ☐ Pre-load OpenCanary fake database tables
26. ☐ Drop LSTM model in (if ready)
27. ☐ Full end-to-end demo rehearsal

---

## 13. Critical Notes

### Encoder Export (DO THIS FIRST)
The ML pipeline will NOT work until you export the label encoders from your training notebooks. The middleware must use the exact same encoding as training. Run this in your notebook:
```python
import joblib
joblib.dump(label_encoder_type, 'type_encoder.pkl')
joblib.dump(label_encoder_country, 'country_encoder.pkl')
# Export ALL encoders and scalers you used during training
```

### GeoIP Database
Download MaxMind GeoLite2-City database (free with registration) and place at `models/GeoLite2-City.mmdb`. Needed for latitude/longitude features.

### Rate Limits
- AbuseIPDB free: 1,000 checks/day → cache aggressively
- AlienVault OTX: 10,000/hour → cache with 1-hour TTL
- Gemini via Vertex AI: check your quota in GCP console

### Wazuh Log Access
The middleware runs on the Wazuh VM, so it can read `/var/ossec/logs/alerts/alerts.json` directly. No Docker volume mounting needed — just ensure the Python process has read permissions.

### Prediction Log Directory
Create before first run:
```bash
sudo mkdir -p /var/log/deceptiq
sudo chown $USER:$USER /var/log/deceptiq
```

---

## 14. Demo Pitch Angle

**"DeceptIQ is not a standalone tool — it's a SIEM extension that makes Wazuh (and any SIEM) smarter."**

Key points for judges:
- Wazuh collects logs from honeypots (operational monitoring)
- DeceptIQ adds ML-powered threat classification + GenAI deception
- Results flow back INTO Wazuh as native alerts (zero workflow change for SOC)
- React dashboard provides deep-dive threat intelligence view
- Every attack makes the system smarter (continuous learning loop)
- Production-ready: GCP-native, STIX export, AbuseIPDB + OTX integration
- Modular: any organization can bolt this onto their existing Wazuh deployment
