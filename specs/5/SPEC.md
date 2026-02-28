# SPEC: Test Gate

**Issue:** smereddy/engineering#5
**Source:** Idea PR #3
**Status:** Draft
**Date:** 2026-02-27

---

## 1. Problem Statement

CI/CD pipelines across the organization lack quality gates between stages. Teams merge pull requests without automated enforcement of test, coverage, or quality thresholds. The result is broken builds propagating downstream, wasted developer time debugging regressions, and inconsistent quality standards across repositories.

**Current pain points:**
- No automated checks blocking progression between pipeline stages
- Broken builds reach downstream environments (staging, production)
- Regressions are detected late, increasing mean time to resolution
- No visibility into which gates pass or fail across the organization
- Teams apply quality standards inconsistently

## 2. Proposed Solution

Build a configurable quality gate system that integrates with existing GitHub Actions workflows. Each repository defines gate rules in a YAML configuration file. Gates run automated checks between pipeline stages and block progression when thresholds are not met. Gate results are cached to avoid redundant test runs, and failures trigger Slack notifications to team leads.

### 2.1 Core Capabilities

1. **Gate Execution Engine**: Runs configured checks (test suites, coverage analysis, linting, custom scripts) between pipeline stages
2. **Threshold Enforcement**: Blocks pipeline progression when results fall below configured thresholds (e.g., coverage < 80%)
3. **Result Caching**: Caches previous test results keyed by content hash to skip redundant runs when code has not changed
4. **Notification System**: Sends Slack notifications to team leads on gate failures with context (which gate, which threshold, diff from passing)
5. **Gate Dashboard**: Web UI showing pass/fail rates, trends, and gate health across repositories
6. **Gate Status API**: REST endpoint for external tools to query current gate status for any repository/branch

### 2.2 How It Works

```
PR Merge → GitHub Actions Trigger → Gate Config Loaded → Cache Check
    ↓                                                        ↓
    ↓                                              [Cache Hit: skip]
    ↓                                              [Cache Miss: run]
    ↓                                                        ↓
Gate Checks Execute → Results Evaluated Against Thresholds
    ↓                           ↓                    ↓
 [PASS]                      [FAIL]             [ERROR]
    ↓                           ↓                    ↓
 Next Stage              Block Pipeline        Alert + Retry
    ↓                    Notify via Slack      Notify via Slack
 Cache Results           Cache Results
    ↓                           ↓
 Update Dashboard        Update Dashboard
```

## 3. Functional Requirements

### 3.1 Gate Configuration (YAML)

Each repository contains a `.test-gate.yml` file at the root defining gate rules.

```yaml
# .test-gate.yml
version: 1
gates:
  - name: unit-tests
    stage: build
    command: npm test
    thresholds:
      coverage: 80
      pass_rate: 100
    timeout: 300  # seconds
    cache:
      enabled: true
      key: unit-tests-{{ hash "src/**" }}

  - name: integration-tests
    stage: deploy-staging
    command: npm run test:integration
    thresholds:
      pass_rate: 100
    timeout: 600
    cache:
      enabled: false

  - name: lint-check
    stage: build
    command: npm run lint
    thresholds:
      pass_rate: 100
    timeout: 120
    cache:
      enabled: true
      key: lint-{{ hash "src/**" "*.config.*" }}

notifications:
  slack:
    channel: "#ci-alerts"
    on_failure: true
    on_recovery: true
    mention: "@team-lead"

settings:
  fail_fast: true          # Stop on first gate failure in a stage
  parallel_gates: true     # Run gates within a stage in parallel
  retry_on_error: 1        # Retry count for errored (not failed) gates
```

### 3.2 Gate Execution

| Requirement | Description |
|---|---|
| **FR-1** | Gates execute as GitHub Actions steps within existing workflows |
| **FR-2** | Gates within the same stage run in parallel by default (configurable) |
| **FR-3** | Gates block pipeline progression to the next stage on failure |
| **FR-4** | Gate timeout is configurable per gate; defaults to 300 seconds |
| **FR-5** | Gates that error (infrastructure failure, timeout) are retried up to the configured retry count |
| **FR-6** | `fail_fast` mode stops remaining gates in a stage when one fails |

### 3.3 Result Caching

| Requirement | Description |
|---|---|
| **FR-7** | Gate results are cached using a content-addressable key derived from file hashes |
| **FR-8** | Cache keys support glob patterns to hash relevant source files |
| **FR-9** | On cache hit, the gate is skipped and the cached result (pass/fail + metrics) is used |
| **FR-10** | Cache entries expire after a configurable TTL (default: 24 hours) |
| **FR-11** | Cache can be explicitly disabled per gate |
| **FR-12** | Cache storage uses GitHub Actions cache or an external store (S3) |

### 3.4 Notifications

| Requirement | Description |
|---|---|
| **FR-13** | Gate failures trigger Slack notifications to the configured channel |
| **FR-14** | Notifications include: gate name, repository, branch, failure reason, threshold vs actual value |
| **FR-15** | Recovery notifications are sent when a previously failing gate passes |
| **FR-16** | Notification channels and mention targets are configurable per repository |

### 3.5 Dashboard

| Requirement | Description |
|---|---|
| **FR-17** | Web dashboard displays gate pass/fail rates per repository |
| **FR-18** | Dashboard shows trend data (pass rate over time) for each gate |
| **FR-19** | Dashboard provides drill-down into individual gate runs with logs and metrics |
| **FR-20** | Dashboard supports filtering by repository, gate name, time range, and status |

### 3.6 API

| Requirement | Description |
|---|---|
| **FR-21** | REST API endpoint: `GET /api/gates/{repo}/{branch}/status` returns current gate status |
| **FR-22** | REST API endpoint: `GET /api/gates/{repo}/history` returns gate run history |
| **FR-23** | REST API endpoint: `GET /api/gates/{repo}/metrics` returns aggregate metrics |
| **FR-24** | API responses include gate name, status, timestamp, threshold values, and actual values |
| **FR-25** | API supports authentication via API key or GitHub token |

## 4. Non-Functional Requirements

| Requirement | Description |
|---|---|
| **NFR-1** | Gate execution overhead must be < 30 seconds (excluding test runtime) |
| **NFR-2** | Cache lookup latency must be < 2 seconds |
| **NFR-3** | Dashboard must load within 3 seconds for up to 100 repositories |
| **NFR-4** | API response time must be < 500ms at p95 |
| **NFR-5** | System must support at least 500 gate runs per hour across all repositories |
| **NFR-6** | Gate results must be stored for at least 90 days for trend analysis |
| **NFR-7** | No secrets or credentials stored in gate configuration files |

## 5. Technical Architecture

### 5.1 Components

```
┌─────────────────────────────────────────────────────────┐
│                    GitHub Actions                        │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────┐ │
│  │ Workflow  │→ │ Gate Action  │→ │ Gate Runner       │ │
│  │ Trigger   │  │ (Composite)  │  │ (executes checks) │ │
│  └──────────┘  └──────┬───────┘  └────────┬──────────┘ │
│                       │                    │             │
│              ┌────────▼────────┐  ┌───────▼──────────┐ │
│              │ Config Parser   │  │ Cache Manager     │ │
│              │ (.test-gate.yml)│  │ (GH Cache / S3)   │ │
│              └─────────────────┘  └──────────────────┘ │
└────────────────────────┬────────────────────────────────┘
                         │ Results
                         ▼
              ┌─────────────────────┐
              │   Results Store     │
              │   (PostgreSQL)      │
              └──────────┬──────────┘
                    ┌────┴────┐
                    ▼         ▼
          ┌──────────┐  ┌──────────┐
          │ Dashboard│  │ REST API │
          │ (Web UI) │  │          │
          └──────────┘  └──────────┘
                    ┌────┘
                    ▼
          ┌──────────────┐
          │ Slack Notify │
          └──────────────┘
```

### 5.2 Technology Choices

| Component | Technology | Rationale |
|---|---|---|
| Gate Action | GitHub Composite Action | Native integration with existing workflows; no infrastructure to manage |
| Config Parser | TypeScript / Node.js | Runs natively in GitHub Actions runners |
| Cache Manager | GitHub Actions Cache + S3 fallback | GitHub Cache for speed; S3 for persistence beyond workflow scope |
| Results Store | PostgreSQL | Structured data, good for time-series queries, existing team expertise |
| Dashboard | Next.js | Consistent with existing frontend stack |
| REST API | Node.js (Express or Fastify) | Lightweight, same language as gate action |
| Notifications | Slack Incoming Webhooks | Simple integration, widely adopted |

### 5.3 GitHub Actions Integration

The gate system is distributed as a reusable GitHub Action. Repositories integrate it by adding a step to their workflow:

```yaml
# .github/workflows/ci.yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Quality Gates
        uses: smereddy/test-gate-action@v1
        with:
          stage: build
          config: .test-gate.yml
          api-url: ${{ secrets.GATE_API_URL }}
          api-key: ${{ secrets.GATE_API_KEY }}
          slack-webhook: ${{ secrets.SLACK_WEBHOOK }}
```

### 5.4 Caching Strategy

Cache keys are computed from file content hashes:

1. Gate config specifies glob patterns for cache key computation
2. At runtime, the gate action hashes the matched files using SHA-256
3. The composite key is: `{gate-name}-{content-hash}`
4. Cache is checked before execution; on hit, stored result is returned
5. On miss, the gate executes normally and results are cached
6. TTL-based expiry prevents stale results (default 24h, configurable)

## 6. User Stories

| ID | Story | Acceptance Criteria |
|---|---|---|
| **US-1** | As a developer, I want quality gates to run automatically on my PR so that I know if my changes meet quality standards before merging | Gates execute on PR events; results appear as GitHub status checks |
| **US-2** | As a team lead, I want to define coverage and test thresholds per repository so that each team can set appropriate quality bars | YAML config supports per-gate thresholds; config is validated on load |
| **US-3** | As a team lead, I want Slack notifications on gate failures so that I can respond quickly to quality regressions | Slack messages sent within 60 seconds of gate failure; includes actionable context |
| **US-4** | As a platform engineer, I want cached gate results so that pipelines are fast when code has not changed | Cache hit skips execution; pipeline time reduced proportionally |
| **US-5** | As a VP of Engineering, I want a dashboard showing gate health across repositories so that I can track quality trends | Dashboard shows pass/fail rates, trends, and allows filtering |
| **US-6** | As a CI/CD tool, I want an API to query gate status so that I can integrate gate results into deployment decisions | API returns structured JSON with gate status, thresholds, and metrics |

## 7. Success Metrics

| Metric | Target | Measurement |
|---|---|---|
| Broken-build incident reduction | 50% reduction from baseline | Count of broken builds in downstream environments (before vs after) |
| Mean time to detect regressions | < 10 minutes | Time from commit to gate failure notification |
| Team adoption rate | 95% of teams within Q2 | Percentage of active repositories with `.test-gate.yml` configured |
| Cache hit rate | > 60% of gate runs | Cached runs / total runs |
| Pipeline overhead | < 30 seconds per gate (excluding test time) | Gate action startup + teardown time |
| Developer satisfaction | > 4.0/5.0 in survey | Quarterly developer experience survey |

## 8. Rollout Plan

### Phase 1: Core Gate Engine (Weeks 1-3)
- Gate configuration parser and validator
- Gate execution engine (run commands, evaluate thresholds)
- GitHub Actions composite action packaging
- Basic pass/fail reporting as GitHub status checks

### Phase 2: Caching and Notifications (Weeks 4-5)
- Result caching with content-addressable keys
- Cache TTL and invalidation
- Slack notification integration
- Recovery notifications

### Phase 3: Dashboard and API (Weeks 6-8)
- Results storage (PostgreSQL schema and ingestion)
- REST API endpoints for gate status, history, and metrics
- Dashboard UI with filtering and trend visualization
- API authentication

### Phase 4: Adoption and Iteration (Weeks 9-12)
- Onboard pilot teams (3-5 repositories)
- Collect feedback and iterate on configuration UX
- Documentation and onboarding guides
- Broader rollout to remaining teams

## 9. Open Questions

| # | Question | Impact | Default Assumption |
|---|---|---|---|
| 1 | Where should the results store and API be hosted? | Infrastructure cost and maintenance | Existing cloud infrastructure (AWS) |
| 2 | Should gates run on PR creation, PR update, or both? | Pipeline execution frequency and cost | Both: run on `pull_request` events (opened, synchronize) |
| 3 | Should the dashboard be a standalone app or embedded in an existing tool? | Development effort and discoverability | Standalone Next.js app, linked from GitHub status checks |
| 4 | What is the cache storage budget and retention policy? | Storage costs | GitHub Actions Cache (10GB per repo limit); S3 for overflow |
| 5 | Should gate configuration support inheritance (org-level defaults)? | Configuration complexity vs consistency | Not in v1; repositories define complete configs |
| 6 | How should the system handle flaky tests (intermittent failures)? | Developer trust in the gate system | Configurable retry count for errored gates; flaky test detection deferred to v2 |

## 10. Out of Scope (v1)

- Automatic test quarantine / flaky test management
- Gate configuration inheritance across org/team hierarchies
- Self-service gate rule creation via UI (config-as-code only)
- Integration with non-GitHub CI systems (Jenkins, GitLab CI)
- Automatic threshold tuning based on historical data
- Cost allocation / chargeback for gate compute time

---

*Generated from idea PR #3 for smereddy/engineering#5.*
