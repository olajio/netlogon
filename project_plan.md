## Project Plan: Netlogon Debug Log Ingestion & Alerting

### Phase 1: Discovery & Sample Analysis

Request a sample domain controller from requester and review sample log files from `C:\Windows\debug\netlogon.log` to help me understand the log format as I'll need this in writing the parsing rules maybe using ingest pipeline or filebeat dissect processors.

**Clarify with requester:**
- How many domain controllers will this cover? Got a 
- What specific events needed to be alerted on: No alert requests for now
- Expected log volume per DC? This is not clear for now, I will device a way to determine this

### Phase 2: Log Format Analysis

I have analysed the logs and now working on the ingest pipeline

### Phase 3: Filebeat Configuration

Create a Filebeat input configuration

### Phase 4: Parsing Pipeline

Ingest Pipeline (recommended)** — Create an Elasticsearch ingest pipeline with grok or dissect processors. This keeps parsing logic centralized and easier to update without redeploying Filebeat.


### Phase 5: Validation & Testing

Deploy to a single DC first:
- Verify logs are ingested and parsed correctly
- Check for parsing failures in the ingest pipeline
- Determine log volume

### Phase 7: Rollout filebeat input
- First rollout to filebeat to other DC's
