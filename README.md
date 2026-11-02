# TA-sycope - Splunk Technology Add-on for Sycope NDR

![Splunk](https://img.shields.io/badge/Splunk-Technology%20Add--on-000000?logo=splunk&logoColor=white)
![Sycope](https://img.shields.io/badge/Sycope-NDR-1F9D8A)
![Language](https://img.shields.io/badge/Language-SPL%20%2F%20XML-2F6FED)

## Overview

TA-sycope is a Splunk Technology Add-on that enables ingestion, parsing, and visualization of security alerts from **Sycope** (Network Detection and Response). Sycope analyzes NetFlow/sFlow/IPFIX data in real-time and generates security alerts with MITRE ATT&CK framework mapping.
<img width="1470" height="1551" alt="image" src="https://github.com/user-attachments/assets/eba42846-77cd-4a7f-8df7-a8f5763ed09" />


### Supported Sycope Versions

- Sycope 3.x and later

### Key Features

- **Dual input support**: CEF/Syslog or JSON/HEC (one at a time)
- **CIM compliance**: Maps to Splunk Common Information Model (Intrusion Detection, Network Traffic)
- **MITRE ATT&CK integration**: 10 tactics, 30 techniques, subtechnique mapping
- **96 alert templates**: Complete reference lookup with severity, score, and MITRE mappings
- **6 built-in dashboards**: Overview, MITRE matrix, Top Attackers, Geo Distribution, Trends, Alert Detail
- **Pivot to Sycope**: one click from an alert in Splunk opens the Sycope console filtered to that IP and time window
- **Unified field namespace**: Both formats normalize to common field names for consistent searching

---

## Installation

1. Copy the `TA-sycope` directory to your Splunk apps directory:
   ```
   cp -r TA-sycope/ $SPLUNK_HOME/etc/apps/
   ```

2. Restart Splunk:
   ```
   $SPLUNK_HOME/bin/splunk restart
   ```

3. The app will appear in the Splunk Web menu as "Sycope Security Alerts".

---

## Configuration

Choose **one** of the following input methods depending on how Sycope is configured to send alerts.

### Option A: CEF/Syslog Input

Sycope sends security alerts as CEF-formatted syslog messages.

#### Enable the built-in listener

Edit `$SPLUNK_HOME/etc/apps/TA-sycope/local/inputs.conf`:

```ini
[udp://514]
disabled = false
sourcetype = sycope:cef
connection_host = dns
index = main
```

#### Configure Sycope

In Sycope:
1. Navigate to **Integrations > External Destinations**
2. Add a new **Syslog** destination
3. Set the syslog server IP to your Splunk instance
4. Set the port to **514** (UDP)
5. Set the format to **CEF**

#### Use an existing syslog infrastructure

If you already have a syslog collector (syslog-ng, rsyslog, Splunk Universal Forwarder):

1. Configure it to forward Sycope events to Splunk
2. Set the sourcetype to `sycope:cef` on the forwarder or in `props.conf`

### Option B: JSON/HEC Input (recommended)

Sycope sends security alerts as JSON via Splunk HTTP Event Collector. This format provides richer data including `mitreSubtechnique`, `score`, `ruleType`, country codes, and session counts.

#### Configure HEC in Splunk

1. Go to **Settings > Data Inputs > HTTP Event Collector**
2. Click **Global Settings** and ensure HEC is **Enabled**
3. Click **New Token**:
   - **Name**: `sycope`
   - **Source type**: Select > `sycope:json`
   - **Index**: `main` (or your preferred security index)
4. Save and **copy the generated token**

#### Configure Sycope

In Sycope:
1. Navigate to **Integrations > External Destinations**
2. Add or edit a **REST client** destination:

| Setting | Value |
|---------|-------|
| **Name** | `Splunk_HEC` |
| **Protocol** | HTTPS |
| **Host** | `<your_splunk_ip>` |
| **Port** | `8088` |
| **Method** | POST |
| **Path** | `/services/collector/raw` |

3. **Query params** tab — add:

| Key | Value |
|-----|-------|
| `sourcetype` | `sycope:json` |

4. **Headers** tab — add:

| Key | Value |
|-----|-------|
| `Authorization` | `Splunk <your_hec_token>` |

5. **Authentication** tab — set to **Disabled** (authorization is handled via the custom header above)

6. **Options** tab — turn **SSL Verify** off if Splunk HEC uses a self-signed certificate

7. **Custom Payload** tab — leave it disabled; TA-sycope parses the default Sycope JSON

8. Add the destination as an action on each alert rule that should reach Splunk (action type third-party system, external system `REST`, your destination as the instance). A destination alone sends nothing.

> **Important**: Use `/services/collector/raw` (not `/services/collector/event`). Sycope sends raw JSON without the `{"event":...}` wrapper that the `/event` endpoint requires. The `Bearer token` authentication option does not work with Splunk HEC — use a custom `Authorization` header with the `Splunk` prefix instead.

#### Verify HEC manually

```bash
curl -k https://<splunk_host>:8088/services/collector/raw?sourcetype=sycope:json \
  -H "Authorization: Splunk <your_hec_token>" \
  -d '{"name":"Test Alert","mitreTactic":"Discovery","severity":"High","score":5}'
```

Expected response: `{"text":"Success","code":0}`

### Custom Index

If you use a custom index (recommended for production), update the macro:

1. Go to **Settings > Advanced Search > Search Macros**
2. Find `sycope_index` (owned by TA-sycope)
3. Update the definition to include your index, e.g.:
   ```
   index=sycope (sourcetype="sycope:cef" OR sourcetype="sycope:json")
   ```

Or create `$SPLUNK_HOME/etc/apps/TA-sycope/local/macros.conf`:
```ini
[sycope_index]
definition = index=sycope (sourcetype="sycope:cef" OR sourcetype="sycope:json")
iseval = 0
```

---

## Dashboards

### 1. Security Alert Overview
- Total alerts, critical/high count, unique source/destination IPs
- Severity pie chart and timeline
- Top 10 alert types and MITRE techniques
- Recent alerts table with drill-down

### 2. MITRE ATT&CK Mapping
- MITRE ATT&CK matrix view with observed techniques per tactic
- Tactic and technique coverage metrics
- Severity distribution by tactic
- Full alert rule reference table (all 96 Sycope templates)
- Technique activity timeline

### 3. Top Attackers & Targets
- Top 20 source IPs (attackers) and destination IPs (targets)
- Source and destination IP detail tables
- Top communication pairs (source -> destination)
- Source IP activity over time

### 4. Geographic Distribution
- World choropleth map of alert activity by country
- Top countries by alert count
- Country detail table with unique IPs and alert types
- City-level breakdown (when iplocation data available)

### 5. Alert Trend Analysis
- Alert volume over time by alert name
- Day of week and hour of day analysis
- Severity trend over time
- Alert type first-seen and activity analysis
- Average alerts per hour by type

### 6. Alert Detail
- Filterable by: Alert ID, Alert Name, Source IP, Destination IP, Severity, MITRE ID
- Summary statistics
- Filtered alerts timeline
- Full alert detail table with all extracted fields
- Raw event viewer for selected alert
- Field extraction table for individual alerts

---

## Pivot to Sycope Console

Every Sycope event in Splunk gets workflow actions that open the Sycope console on the **Alerts** view, filtered
to the alert's IP and a +/-15 minute window around it:

| Action | Sycope filter |
|--------|---------------|
| Sycope: investigate source IP | alerts where the source IP is client or server |
| Sycope: investigate destination IP | alerts where the destination IP is client or server |
| Sycope: investigate conversation | alerts between the source and the destination IP |

They are in the event's **Event Actions** menu, and the IP ones also in the menu of the `src_ip` / `dest_ip`
field.

### Setup

Set your Sycope address in the links (replace `SYCOPE_HOST`). Keep the change in `local/` so an app upgrade
does not overwrite it:

```bash
cd $SPLUNK_HOME/etc/apps/TA-sycope
mkdir -p local
sed 's/SYCOPE_HOST/sycope.example.com/g' default/workflow_actions.conf > local/workflow_actions.conf
```

Then restart Splunk (or reload `/debug/refresh`). The analyst needs a Sycope login; the link opens in a new tab.

---

## Field Reference

### Unified Fields (available for both sourcetypes)

| Field | Description | Example |
|-------|-------------|---------|
| `alert_name` | Alert rule name | Brute Force Attack |
| `alert_id` | Unique alert instance ID | 6711de99-e073839a-28612f9c |
| `alert_severity` | Alert severity level | High |
| `alert_rule_id` | Alert rule identifier | Credential Access_008 |
| `alert_definition_id` | Alert definition ID | 65da262b-27dd8da4-d94acaf7 |
| `alert_threshold_level` | Threshold level | Minor |
| `alert_tags` | Semicolon-separated tags | 60143f92-...;60143f92-... |
| `severity` | CIM severity (lowercase) | high |
| `rule_type` | Alert rule type (uppercase) | SECURITY, PERFORMANCE, VISIBILITY |
| `src_ip` | Source IP address | 10.240.2.49 |
| `dest_ip` | Destination IP address | 84.205.187.65 |
| `dest_port` | Destination port | 53 |
| `mitre_tactic` | MITRE ATT&CK tactic | Credential Access |
| `mitre_technique` | MITRE ATT&CK technique | Brute Force |
| `mitre_technique_id` | MITRE technique ID | T1110 |
| `mitre_subtechnique` | MITRE subtechnique (from lookup or JSON) | T1595.001: Scanning IP Blocks |
| `mitre_tactic_id` | MITRE tactic ID (from lookup) | TA0006 |
| `transport` | Protocol name | tcp, udp, icmp |
| `signature` | CIM alert signature | Brute Force Attack |
| `vendor_product` | CIM vendor product | Sycope NDR |

### CEF-specific Fields

| Field | Description |
|-------|-------------|
| `cef_vendor` | CEF vendor (Sycope) |
| `cef_product` | CEF product, as sent by Sycope (`NPM`) |
| `cef_device_version` | Sycope version |
| `cef_severity` | CEF numeric severity |
| `bytes_in` | Inbound bytes |
| `bytes_out` | Outbound bytes |
| `packets` | Packet/session count |
| `sum_bytes` | Total bytes |

### JSON-specific Fields

| Field | Description |
|-------|-------------|
| `alert_score` | Numeric alert score (1-10) |
| `session_count` | Aggregated session count |
| `unique_client_ips` | Distinct client IP count |
| `dest_country` | Destination country code |
| `src_country` | Source country code |
| `mitre_subtechnique` | MITRE subtechnique (native field) |

### Lookup-enriched Fields

| Field | Source Lookup | Description |
|-------|-------------|-------------|
| `mitre_tactic_id` | `sycope_mitre_techniques` | MITRE tactic ID (e.g., TA0006) |
| `mitre_subtechnique` | `sycope_alert_reference` | Subtechnique for CEF events |
| `default_severity` | `sycope_alert_reference` | Default severity from template |
| `default_score` | `sycope_alert_reference` | Default risk score (1-10) |
| `severity_normalized` | `sycope_severity_lookup` | Normalized severity |
| `severity_weight` | `sycope_severity_lookup` | Numeric weight for sorting |
| `protocol_name` | `sycope_protocol_lookup` | Protocol name from number |

---

## CIM Compliance

### Intrusion Detection Data Model

TA-sycope maps to the `IDS_Attacks` data model:

| CIM Field | Mapping |
|-----------|---------|
| `action` | "allowed" (Sycope is passive) |
| `category` | Security |
| `dest` | Destination IP |
| `dest_port` | Destination port |
| `ids_type` | "network" |
| `severity` | Alert severity (lowercase) |
| `signature` | Alert name |
| `signature_id` | Rule ID |
| `src` | Source IP |
| `transport` | Protocol (tcp/udp/icmp) |
| `vendor_product` | "Sycope NDR" |

### Network Traffic Data Model (CEF only)

| CIM Field | Mapping |
|-----------|---------|
| `bytes_in` | Inbound bytes |
| `bytes_out` | Outbound bytes |
| `bytes` | Total bytes |
| `packets` | Count |

### Event Types and Tags

| Event Type | Tags | CIM Model |
|-----------|------|-----------|
| `sycope_ids_alert` | ids, attack | Intrusion Detection |
| `sycope_network_traffic` | network, communicate | Network Traffic |
| `sycope_security_alert` | — | Security rule alerts |
| `sycope_performance_alert` | — | Performance rule alerts |
| `sycope_visibility_alert` | — | Visibility rule alerts |

---

## Troubleshooting

### No data appearing

1. Verify data is arriving:
   ```
   index=_internal sourcetype=splunkd "sycope"
   ```

2. Check sourcetype assignment:
   ```
   | eventcount summarize=false index=* sourcetype=sycope:*
   ```

3. For syslog: verify the port is listening:
   ```
   | rest /services/data/inputs/udp | search title=514
   ```

4. For HEC: verify the token is active:
   ```
   | rest /services/data/inputs/http | search name=sycope*
   ```

5. For HEC: check for authorization errors:
   ```
   index=_internal sourcetype=splunkd "HttpInputDataHandler" ERROR | head 10
   ```

### HEC returns errors

| Error | Cause | Fix |
|-------|-------|-----|
| `401 Invalid authorization` | Wrong token or missing `Splunk` prefix | Verify header: `Authorization: Splunk <token>` |
| `400 No data` | Using `/services/collector/event` | Switch to `/services/collector/raw` |
| `404 Not Found` | Wrong URL path | Use `/services/collector/raw` |

### Fields not extracting

1. Verify the sourcetype is set correctly:
   ```
   index=* (sourcetype="sycope:cef" OR sourcetype="sycope:json") | head 5
   ```

2. Check field extraction for CEF:
   ```
   sourcetype="sycope:cef" | table alert_name, severity, src, dst, mitre_tactic
   ```

3. Check field extraction for JSON:
   ```
   sourcetype="sycope:json" | table name, severity, mitre_tactic, serverIp.addressString
   ```

### Dashboards show no data

1. Verify the `sycope_index` macro matches your configuration:
   ```
   | makeresults | eval search="`sycope_index`" | table search
   ```

2. Update the macro if using a custom index:
   - **Settings > Advanced Search > Search Macros > sycope_index**

---

## Release Notes

### Version 1.2.0
- Naming: Sycope NPM renamed to Sycope / Sycope NDR in the app label, dashboards and docs
- `vendor_product` is now `Sycope NDR` (was `Sycope NPM`); `dvc` is `sycope` for JSON events

### Version 1.1.0
- Pivot to the Sycope console from Splunk events (workflow actions on source IP, destination IP, conversation)
- Aggregated alerts (JSON): `clientIps` / `serverIps` lists are split into multivalue `src_ip` / `dest_ip`
- Sycope `Info` severity is normalized to the CIM value `informational`

### Version 1.0.0
- Initial release
- Support for CEF/Syslog and JSON/HEC input formats
- CIM compliance (Intrusion Detection, Network Traffic)
- MITRE ATT&CK field mapping (10 tactics, 30 techniques)
- 96 alert template reference lookup with severity and MITRE mappings
- 6 built-in dashboards
- Severity, protocol, and MITRE lookup tables

---

## Support

For issues and feature requests, contact Sycope support or open an issue in the project repository.
