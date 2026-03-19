# OpenCTI ONYPHE Connector

| Status | Date | Comment |
|--------|------|---------|
| Community | -    | -       |

## Table of Contents

- [Introduction](#introduction)
- [Installation](#installation)
  - [Requirements](#requirements)
- [Configuration](#configuration)
  - [OpenCTI Configuration](#opencti-configuration)
  - [Base Connector Configuration](#base-connector-configuration)
  - [ONYPHE Configuration](#onyphe-configuration)
- [Deployment](#deployment)
  - [Docker Deployment](#docker-deployment)
  - [Multiple Instances](#multiple-instances)
  - [Manual Deployment](#manual-deployment)
- [Usage](#usage)
- [Behavior](#behavior)
  - [Data Flow](#data-flow)
  - [Observable Enrichment](#observable-enrichment)
  - [Indicator Enrichment](#indicator-enrichment)
  - [Generated STIX Objects](#generated-stix-objects)
- [Warnings](#warnings)
- [Debugging](#debugging)
- [Additional Information](#additional-information)

---

## Introduction

[ONYPHE](https://www.onyphe.io/) is a cyber defense search engine that collects open-source and cyber threat intelligence data by crawling the Internet. This connector enriches observables and indicators with comprehensive network intelligence.

The connector supports multiple [ONYPHE data categories](https://search.onyphe.io/docs/data-models/) through a single codebase. Each deployed instance is configured for a specific category, enabling distinct use cases on the same OpenCTI platform:

| Category | Use case | Key output |
|----------|----------|------------|
| `ctiscan` (default) | Threat intelligence enrichment | IP, hostname, certificate, fingerprint observables |
| `riskscan` | Attack surface management | IP, hostname, vulnerabilities (CVEs), risk labels |

---

## Installation

### Requirements

- OpenCTI Platform >= 6.0.0
- ONYPHE API key
- Network access to ONYPHE API

---

## Configuration

### OpenCTI Configuration

| Parameter | Docker envvar | Mandatory | Description |
|-----------|---------------|-----------|-------------|
| `opencti_url` | `OPENCTI_URL` | Yes | The URL of the OpenCTI platform |
| `opencti_token` | `OPENCTI_TOKEN` | Yes | The default admin token configured in the OpenCTI platform |

### Base Connector Configuration

| Parameter | Docker envvar | Mandatory | Description |
|-----------|---------------|-----------|-------------|
| `connector_id` | `CONNECTOR_ID` | Yes | A valid arbitrary `UUIDv4` — must be unique per instance |
| `connector_name` | `CONNECTOR_NAME` | Yes | Display name in OpenCTI — use distinct names per instance |
| `connector_scope` | `CONNECTOR_SCOPE` | Yes | See per-category scope below |
| `connector_auto` | `CONNECTOR_AUTO` | Yes | Enable/disable auto-enrichment |
| `connector_confidence_level` | `CONNECTOR_CONFIDENCE_LEVEL` | Yes | Default confidence level (0-100) |
| `connector_log_level` | `CONNECTOR_LOG_LEVEL` | Yes | Log level (`debug`, `info`, `warn`, `error`) |

### ONYPHE Configuration

| Parameter | Docker envvar | Mandatory | Description |
|-----------|---------------|-----------|-------------|
| `onyphe_api_key` | `ONYPHE_API_KEY` | Yes | ONYPHE API key |
| `onyphe_category` | `ONYPHE_CATEGORY` | No | Data category: `ctiscan` (default) or `riskscan` |
| `onyphe_base_url` | `ONYPHE_BASE_URL` | No | API base URL (default: `https://www.onyphe.io/api/v2/`) |
| `onyphe_max_tlp` | `ONYPHE_MAX_TLP` | No | Maximum TLP for enrichment (default: `TLP:AMBER`) |
| `onyphe_time_since` | `ONYPHE_TIME_SINCE` | No | Time window for data retrieval (default: `1w`) |
| `onyphe_default_score` | `ONYPHE_DEFAULT_SCORE` | No | Default score for created observables (default: `50`) |
| `onyphe_import_search_results` | `ONYPHE_IMPORT_SEARCH_RESULTS` | No | Import results as observables for indicator enrichment (default: `true`) |
| `onyphe_create_note` | `ONYPHE_CREATE_NOTE` | No | Attach enrichment summary as a Note (default: `false`) |
| `onyphe_import_full_data` | `ONYPHE_IMPORT_FULL_DATA` | No | Import full raw response text — can produce large data (default: `false`) |
| `onyphe_pivot_threshold` | `ONYPHE_PIVOT_THRESHOLD` | No | Skip observable enrichment if result count exceeds this (default: `10`) |
| `onyphe_pattern_type` | `ONYPHE_PATTERN_TYPE` | No | Vocabulary entry for ONYPHE indicator pattern type (default: `onyphe`) |

#### Connector scope by category

| Category | `CONNECTOR_SCOPE` |
|----------|-------------------|
| `ctiscan` | `IPv4-Addr,IPv6-Addr,Domain-Name,Hostname,x509-Certificate,Text,Indicator` |
| `riskscan` | `IPv4-Addr,IPv6-Addr,Domain-Name,Hostname,x509-Certificate,Indicator` |

---

## Deployment

### Docker Deployment

Build a Docker image using the provided `Dockerfile`.

**Single instance (ctiscan, default behaviour):**

```yaml
version: '3'
services:
  connector-onyphe:
    image: opencti/connector-onyphe:latest
    environment:
      - OPENCTI_URL=http://localhost
      - OPENCTI_TOKEN=ChangeMe
      - CONNECTOR_ID=ChangeMe
      - CONNECTOR_NAME=ONYPHE
      - CONNECTOR_SCOPE=IPv4-Addr,IPv6-Addr,Domain-Name,Hostname,x509-Certificate,Text,Indicator
      - CONNECTOR_AUTO=false
      - CONNECTOR_CONFIDENCE_LEVEL=50
      - CONNECTOR_LOG_LEVEL=error
      - ONYPHE_API_KEY=ChangeMe
      - ONYPHE_MAX_TLP=TLP:AMBER
      - ONYPHE_DEFAULT_SCORE=50
      - ONYPHE_IMPORT_SEARCH_RESULTS=true
      - ONYPHE_CREATE_NOTE=true
      - ONYPHE_IMPORT_FULL_DATA=false
      - ONYPHE_PIVOT_THRESHOLD=100
    restart: always
```

### Multiple Instances

Running multiple instances of the same Docker image with different `ONYPHE_CATEGORY` values lets you serve distinct use cases on the same OpenCTI platform. Each instance must have a unique `CONNECTOR_ID` and a distinct `CONNECTOR_NAME` so OpenCTI registers them separately.

**Example: CTI enrichment and ASM side by side:**

```yaml
version: '3'
services:

  # Threat intelligence enrichment — uses ONYPHE ctiscan category
  connector-onyphe-cti:
    image: opencti/connector-onyphe:latest
    environment:
      - OPENCTI_URL=http://localhost
      - OPENCTI_TOKEN=ChangeMe
      - CONNECTOR_ID=00000000-0000-0000-0000-000000000001   # unique UUIDv4
      - CONNECTOR_NAME=ONYPHE CTI
      - CONNECTOR_SCOPE=IPv4-Addr,IPv6-Addr,Domain-Name,Hostname,x509-Certificate,Text,Indicator
      - CONNECTOR_AUTO=false
      - CONNECTOR_CONFIDENCE_LEVEL=50
      - CONNECTOR_LOG_LEVEL=error
      - ONYPHE_API_KEY=ChangeMe
      - ONYPHE_CATEGORY=ctiscan
      - ONYPHE_MAX_TLP=TLP:AMBER
      - ONYPHE_DEFAULT_SCORE=50
      - ONYPHE_IMPORT_SEARCH_RESULTS=true
      - ONYPHE_PIVOT_THRESHOLD=100
    restart: always

  # Attack surface management — uses ONYPHE riskscan category
  connector-onyphe-asm:
    image: opencti/connector-onyphe:latest
    environment:
      - OPENCTI_URL=http://localhost
      - OPENCTI_TOKEN=ChangeMe
      - CONNECTOR_ID=00000000-0000-0000-0000-000000000002   # unique UUIDv4
      - CONNECTOR_NAME=ONYPHE ASM
      - CONNECTOR_SCOPE=IPv4-Addr,IPv6-Addr,Domain-Name,Hostname,x509-Certificate,Indicator
      - CONNECTOR_AUTO=false
      - CONNECTOR_CONFIDENCE_LEVEL=50
      - CONNECTOR_LOG_LEVEL=error
      - ONYPHE_API_KEY=ChangeMe
      - ONYPHE_CATEGORY=riskscan
      - ONYPHE_MAX_TLP=TLP:AMBER
      - ONYPHE_DEFAULT_SCORE=50
      - ONYPHE_IMPORT_SEARCH_RESULTS=true
      - ONYPHE_PIVOT_THRESHOLD=100
    restart: always
```

Each instance appears as a separate connector in the OpenCTI UI and can be given independent trigger filters, auto-enrichment settings, and confidence levels.

#### Indicator patterns and category selection

Indicators use `pattern_type: onyphe` and carry an OQL query as their pattern. The connector prepends `category:<ONYPHE_CATEGORY>` automatically if no `category:` clause is already present in the pattern. This means:

- An indicator with pattern `ip.dest:1.2.3.4` processed by the CTI instance becomes `category:ctiscan ip.dest:1.2.3.4`
- The same indicator processed by the ASM instance becomes `category:riskscan ip:1.2.3.4`
- An indicator that already includes `category:riskscan ip:1.2.3.4` is passed through unchanged by either instance

### Manual Deployment

1. Clone the repository
2. Copy `config.yml.sample` to `config.yml` and configure
3. Install dependencies: `pip install -r requirements.txt`
4. Run: `python src/main.py`

---

## Usage

The connector enriches:
1. **Observables**: IP addresses, domains, hostnames, certificates, text (fingerprints)
2. **Indicators**: OQL patterns with `pattern_type: onyphe`

Trigger enrichment:
- Manually via the OpenCTI UI
- Automatically if `CONNECTOR_AUTO=true` (see warnings)
- Via playbooks

---

## Behavior

### Data Flow

```mermaid
flowchart LR
    A[Observable/Indicator] --> B[ONYPHE Connector]
    B --> C{ONYPHE API}
    C --> D[Results]
    D --> E[IP Addresses]
    D --> F[Organizations]
    D --> G[Domains/Hostnames]
    D --> H[ASN]
    D --> I[Certificates]
    D --> J[Vulnerabilities]
    E --> K[OpenCTI]
    F --> K
    G --> K
    H --> K
    I --> K
    J --> K
```

### Observable Enrichment

The connector supports the following observable types as enrichment inputs. The STIX objects generated depend on the configured category.

#### Supported input types

| Observable type | ctiscan OQL field(s) | riskscan OQL field(s) |
|-----------------|----------------------|-----------------------|
| IPv4-Addr | `ip.dest:{value}` | `ip:{value}` |
| IPv6-Addr | `ip.dest:{value}` | `ip:{value}` |
| Domain-Name | `?dns.domain:{v} ?cert.domain:{v} ?extract.domain:{v}` | `domain:{value}` |
| Hostname | `?dns.hostname:{v} ?cert.hostname:{v}` | `hostname:{value}` |
| X509-Certificate | `cert.fingerprint.<algo>:{hash}` | `fingerprint.<algo>:{hash}` |
| Text | analytical pivot field (label-driven) | not supported |
| Indicator | OQL pattern (passed through) | OQL pattern (passed through) |

#### Generated STIX objects by category

| STIX object | ctiscan | riskscan |
|-------------|:-------:|:--------:|
| IPv4-Addr / IPv6-Addr | Yes | Yes |
| Domain-Name | Yes | Yes |
| Hostname | Yes | Yes |
| Autonomous-System | Yes | Yes |
| Organization (Identity) | Yes | Yes |
| X509-Certificate | Yes | Yes |
| Vulnerability | — | Yes |
| Text (fingerprint pivots) | Yes | — |
| Note (indicator summary) | Yes | Yes |
| Labels | Yes | Yes |
| External Reference | Yes | Yes |

Vulnerabilities generated by the riskscan category are linked to the enriched observable with a `has` relationship and carry an external reference to the CVE record.

### Indicator Enrichment

Indicators with `pattern_type: onyphe` are executed as OQL queries against the configured category. The connector returns:

- A **Note** containing a summary table of top values across key fields (IPs, organisations, countries, ports, protocols, CVEs, risk tags, etc.)
- Optionally, the full set of matching observables (when `ONYPHE_IMPORT_SEARCH_RESULTS=true`)

The summary fields vary by category:

| ctiscan summary fields | riskscan summary fields |
|------------------------|-------------------------|
| IP addresses | IP addresses |
| Organizations | Organizations |
| ASNs | Countries |
| Countries | Hostnames |
| Cert hostnames | Ports |
| Cert domains | Protocols |
| DNS hostnames | CVEs |
| TCP ports | Risk tags |
| Protocols | |
| Technologies | |

---

## Warnings

### Import Full Data

Setting `ONYPHE_IMPORT_FULL_DATA=true` imports the complete raw application response text into the enrichment description. This can produce very large objects. Start with `false`.

### Pivot Threshold

`ONYPHE_PIVOT_THRESHOLD` sets the maximum number of results before observable enrichment is skipped entirely for a given query. This guards against runaway imports on broad queries. The default is `10` — raise it deliberately for known high-cardinality targets.

### Auto Enrichment

Setting `CONNECTOR_AUTO=true` with broad scopes can trigger large numbers of API calls. Use Trigger Filters in the OpenCTI UI to limit which entities are processed automatically:

1. Navigate to: Data → Ingestion → Connectors → (connector name)
2. Add Trigger Filters to restrict which entities trigger enrichment

When running multiple instances, set `CONNECTOR_AUTO` independently per instance and apply appropriate filters to each.

---

## Debugging

Set `CONNECTOR_LOG_LEVEL=debug` to log:
- API request details and OQL queries
- Per-observable processing steps
- STIX object creation

---

## Additional Information

- [ONYPHE](https://www.onyphe.io/)
- [ONYPHE API Documentation](https://www.onyphe.io/documentation/api)
- [ONYPHE ctiscan data model](https://search.onyphe.io/docs/data-models/ctiscan)
- [ONYPHE riskscan tags](https://search.onyphe.io/docs/tags/riskscan)
- [ONYPHE vulnscan tags](https://search.onyphe.io/docs/tags/vulnscan)

### API Considerations

ONYPHE API has rate limits. The connector handles HTTP 429 responses with exponential back-off. To reduce API load:
- Use manual enrichment for high-value targets
- Set `ONYPHE_PIVOT_THRESHOLD` appropriately
- Avoid `CONNECTOR_AUTO=true` on broad scopes
