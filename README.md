# PTC Windchill — Dynatrace Gen3 Monitoring Starter Set

Complete Monitoring as Code (Monaco v2) project for PTC Windchill, targeting the Dynatrace Gen3 platform. Covers JMX extensions, segments (Gen3 replacement for management zones), auto-tagging, request attributes, calculated metrics, custom services, RUM application, synthetic monitors, and 12 dashboards.

---

## Contents

```
gen3/
├── manifest.yaml                          # Monaco project manifest
├── extensions/                            # Extensions 2.0 (JMX)
│   ├── custom.ptc.windchill.active-users/
│   ├── custom.ptc.windchill.caches/
│   └── custom.ptc.windchill.queues/
└── ptc-windchill/                         # Monaco project folder
    ├── segments/                          # Gen3 Segment (replaces Management Zones)
    ├── auto-tags/                         # builtin:tags.auto-tagging  (incl. broad "PTC" scope tag)
    ├── request-attributes/                # builtin:request-attribute
    ├── calculated-metrics/                # builtin:calculated.metrics.service
    ├── custom-services/                   # builtin:service.customservices
    ├── app/                               # builtin:rum.web.app
    ├── app-detection/                     # builtin:rum.web.app-detection
    ├── synthetic/                         # builtin:synthetic.browser.monitor
    └── dashboards/                        # Gen3 Documents API dashboards (12 total)
```

---

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| [Monaco v2](https://github.com/Dynatrace/dynatrace-configuration-as-code) | ≥ 2.x | `brew install dynatrace/tap/monaco` or download from GitHub Releases |
| [Dynatrace CLI (`dtctl`)](https://docs.dynatrace.com/docs/manage/configuration-as-code/dynatrace-cli) | latest | Required to build and deploy Extensions 2.0 |
| Dynatrace Gen3 tenant | – | SaaS with Platform capabilities enabled |
| Platform token (OAuth) | – | Created in Account Management — see below |

### Platform Token Scopes

> **Gen3 note:** Classic API tokens (`dt0c01.*`, created in *Settings → Access tokens*) are deprecated in Dynatrace Gen3. Authentication now uses **Platform Tokens** issued via OAuth 2.0 client credentials. Tokens are created in the **Dynatrace Account Management** portal, not inside an individual environment.

Navigate to **Account Management → Identity & access management → OAuth clients**, create a new OAuth client, and grant the following scopes:

| Scope | Used for |
|-------|---------|
| `settings:read`, `settings:write` | Monaco deploy (Settings API 2.0) |
| `entities:read` | Entity ID lookup |
| `document:read`, `document:write` | Dashboard deploy |
| `extensions:read`, `extensions:write` | Extension 2.0 deploy |
| `syntheticmonitor:read`, `syntheticmonitor:write` | Synthetic monitor deploy |
| `segments:read`, `segments:write` | Segment deploy |

After creating the OAuth client you will receive a **Client ID** and **Client Secret** — keep these safe. The token endpoint URL is `https://sso.dynatrace.com/sso/oauth2/token` for SaaS.

---

## Step 1: Fill In Placeholders

Before deploying, replace every placeholder in the files below. A full placeholder reference is at the end of this document.

### Synthetic Monitors (`gen3/ptc-windchill/synthetic/`)

Open each of the three JSON files and replace:

| Placeholder | Replace with |
|-------------|-------------|
| `WINDCHILL_BASE_URL` | Your Windchill base URL, e.g. `https://windchill.example.com` |
| `WINDCHILL_DOCUMENT_URL_PLACEHOLDER` | Full URL of a specific document for the file-server download check |
| `REPLICA_HOSTNAME_PLACEHOLDER` | Hostname of the file server replica (used in the download check) |
| `LOCATION_ENTITY_ID` | A valid Dynatrace synthetic location entity ID (find with DQL below) |

**Find a synthetic location entity ID:**
```dql
fetch dt.entity.synthetic_location
| fields entity.name, id
| sort entity.name
```

All synthetic monitors are deployed with `enabled: false`. Enable each one manually in the Dynatrace UI after verifying the URLs are correct.

---

## Step 2: Set Environment Variables

Export these in your shell before running any Monaco or dtctl command:

```bash
export DT_ENVIRONMENT_URL="https://<your-tenant-id>.apps.dynatrace.com"  # no trailing slash
export DT_OAUTH_CLIENT_ID="dt0s02.xxxxxxxxxxxx"                           # OAuth Client ID
export DT_OAUTH_CLIENT_SECRET="dt0s02.xxxxxxxxxxxx.yyyyyyyy"              # OAuth Client Secret
export DT_OAUTH_TOKEN_URL="https://sso.dynatrace.com/sso/oauth2/token"
```

Create a `gen3/environments.yaml` file (not committed — add it to `.gitignore`) to wire Monaco to OAuth:

```yaml
# gen3/environments.yaml  — DO NOT COMMIT (contains secrets references)
environments:
  production:
    url:
      type: environment
      value: DT_ENVIRONMENT_URL
    auth:
      oAuth:
        clientId:
          type: environment
          value: DT_OAUTH_CLIENT_ID
        clientSecret:
          type: environment
          value: DT_OAUTH_CLIENT_SECRET
        tokenEndpoint:
          type: environment
          value: DT_OAUTH_TOKEN_URL
```

---

## Step 3: Deploy Extensions 2.0 (JMX)

Extensions 2.0 are built and signed with `dtctl` — Monaco does not manage them directly.

```bash
# Authenticate dtctl (one-time setup) using OAuth credentials
dtctl auth login \
  --url "$DT_ENVIRONMENT_URL" \
  --client-id "$DT_OAUTH_CLIENT_ID" \
  --client-secret "$DT_OAUTH_CLIENT_SECRET"

# Build and deploy each extension
for ext in gen3/extensions/custom.ptc.windchill.active-users \
            gen3/extensions/custom.ptc.windchill.caches \
            gen3/extensions/custom.ptc.windchill.queues; do
    dtctl extensions build "$ext"
    dtctl extensions upload "$ext/dist/$(ls $ext/dist | head -1)"
done
```

After uploading, activate each extension and configure the monitoring configuration (select the JMX endpoint host/process) in **Settings → Extensions 2.0**.

---

## Step 4: First Monaco Deploy

Run a dry-run first to validate:

```bash
cd gen3
monaco deploy manifest.yaml --environment production --dry-run
```

If the dry-run passes, deploy everything **except** the app-detection rule (which needs the APPLICATION entity ID from the previous step):

```bash
monaco deploy manifest.yaml --environment production
```

> **Note:** The app-detection config (`app-detection/config.yaml`) uses a placeholder scope `APPLICATION_ENTITY_ID_PLACEHOLDER`. Monaco will fail on that config until you complete Step 5. You can deploy with `--project ptc-windchill` and temporarily exclude the app-detection directory, or simply allow the error for that one config and continue.

---

## Step 5: Update App Detection Rule (Two-Step Deploy)

The application detection rule must be scoped to the specific APPLICATION entity created in Step 4.

**1. Find the entity ID:**

```dql
fetch dt.entity.application
| filter entity.name == "PTC Windchill"
| fields id, entity.name
```

**2. Update the scope in `gen3/ptc-windchill/app-detection/config.yaml`:**

```yaml
# Replace APPLICATION_ENTITY_ID_PLACEHOLDER with the actual ID
scope: "APPLICATION-XXXXXXXXXXXXXXXX"
```

**3. Redeploy:**

```bash
monaco deploy manifest.yaml --environment production
```

---

## Step 6: Verify the Deployment

Run these DQL queries in a Dynatrace Notebook to confirm everything landed:

```dql
// Segment exists and "PTC" tag is being applied (run after Monaco deploy)
fetch dt.entity.service
| filter in(tags, "PTC")
| fields entity.name, tags
| limit 10
```

```dql
// Web app exists
fetch dt.entity.application
| filter entity.name == "PTC Windchill"
| fields entity.name, id, healthState
```

```dql
// JMX metrics are ingesting (run after ~5 min of extension activity)
timeseries avg(`ext:ptc.windchill.active_users.count`)
| limit 5
```

```dql
// Dashboards were created
fetch documents
| filter type == "dashboard" and matchesPhrase(name, "PTC Windchill")
| fields name, id
```

---

## Metric Key Reference

### Extension 2.0 Metrics (JMX)

| Metric Key | Description | Dimension |
|------------|-------------|-----------|
| `ext:ptc.windchill.active_users.count` | Active user count | — |
| `ext:ptc.windchill.active_users.total_count` | Total active user count | — |
| `ext:ptc.windchill.caches.size` | Cache size | `cache_name` |
| `ext:ptc.windchill.caches.entry_count` | Cache entry count | `cache_name` |
| `ext:ptc.windchill.queues.total_entries` | Queue total entries | `queue_name` |
| `ext:ptc.windchill.queues.waiting_ready_entries` | Queue waiting/ready entries | `queue_name` |
| `ext:ptc.windchill.queues.max_exec_time_seconds` | Max entry execution time (s) | `queue_name` |

### Calculated Service Metrics

| Metric Key | Description | Dimension |
|------------|-------------|-----------|
| `calc:service.ptc.exception_count` | Exception count by class | `Exception:Class` |
| `calc:service.ptc.action_name_by_user` | Response time by action + user | `{RequestAttribute:ActionName} By {RequestAttribute:GetRemoteUser}` |

---

## Dashboard Reference

| File | Dashboard Name |
|------|---------------|
| `ptc-operations.json` | PTC Windchill - Operations |
| `m108-app-overview.json` | M108 - Windchill Application Overview |
| `d100-caches.json` | D100 - Cache Dashboard |
| `d202-workspace-operations.json` | D202 - Workspace Operations |
| `m101-ms-status.json` | M101 - Method Server Status |
| `m103-sm-status.json` | M103 - Server Manager Status |
| `m105-process-queues.json` | M105 - Process Queues |
| `m106-publish-queues.json` | M106 - Publish Queues |
| `m107-schedule-queues.json` | M107 - Schedule Queues |
| `m109-host-health.json` | M109 - HOST Health |
| `m200-jvm-health.json` | M200 - JVM Health |
| `m201-host-monitoring.json` | M201 - HOST Windchill Monitoring |

---

## Auto-Tags Applied

| Tag | Filter | Propagation |
|-----|--------|-------------|
| `PTC` | All PTC Windchill processes (by Java main class / command args) + web app named "PTC Windchill" | Processes → Services, Processes → Hosts |
| `MethodServer` | Java main class `wt.method.MethodServerMain` | — |
| `ServerManagerMain` | Java main class `wt.manager.ServerManagerMain` | — |
| `DirectoryServer` | Java main class `com.ptc.windchill.uwgm.proessproxy.DirectoryServerMain` | — |
| `BackgroundMethodServer` | Command-line args containing `BackgroundMethodServer` | — |

> **Note on Segments:** Management Zones are deprecated in Dynatrace Gen3 and replaced by **Segments** (`segments/ptc-segment.json`). The Segment uses `tag("PTC")` entity selector syntax to group all PTC Windchill entities. Monaco v2 deploys the segment via the `segment` config type. Dashboard DQL queries use `entitySelector("type(X),tag(\"PTC\")")` instead of the deprecated `mzName()` predicate.

---

## Custom Services Captured

Checkin, Checkout, Copy, Create, CreateDocument, Delete, Edit, FolderNavigation, Move, Paste, Rename, SetPreference, WCLoginBean

All defined on Java class entry points under `com.ptc.windchill.*` and `wt.httpgw.*`.

---

## Troubleshooting

**Monaco deploy fails on `segment` config type:**
Monaco v2.x added `segment` as a config type. If you see "unknown config type: segment", upgrade Monaco to the latest 2.x release (`monaco --version`). As a workaround, create the Segment manually via **Dynatrace UI → Segments** (or the Segments API) using the criteria defined in `segments/ptc-segment.json`.

**Monaco deploy fails with "400 Bad Request" on app-detection:**
The scope must be a valid `APPLICATION-XXXXXXXXXXXXXXXX` entity ID. Complete Step 5 first.

**No JMX metrics appearing:**
- Confirm the OneAgent version on the Windchill host supports Extensions 2.0 (≥ 1.257)
- Verify the extension is activated in the Extensions 2.0 UI and a monitoring configuration is assigned to the correct process

**Dashboards not visible:**
- Confirm your OAuth client has the `document:read` and `document:write` scopes
- Check Monaco output for `document` config errors

**Synthetic monitors failing:**
All monitors are deployed disabled (`enabled: false`). Enable them in the Dynatrace UI only after replacing the URL placeholders and confirming connectivity.
