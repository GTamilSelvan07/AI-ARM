# AI-ARM Technical Product Requirements Document (PRD)

## Document Info
| Field | Value |
|-------|-------|
| Product | AI-ARM Platform |
| Version | 1.0 (MVP) |
| Author | AI-ARM Founding Team |
| Last Updated | February 2025 |
| Status | Draft |

---

## 1. Executive Summary

### 1.1 Product Overview
AI-ARM is a vendor-agnostic control plane for managing enterprise AI agents. It provides unified visibility, context portability, quality monitoring, cost optimization, and governance across agents from multiple vendors (OpenAI, Anthropic, Google, Microsoft, custom).

### 1.2 Problem Statement
Enterprises deploying AI agents face:
- **Fragmentation**: Agents scattered across 5+ vendors with no unified view
- **Context Loss**: Switching vendors means losing months of accumulated intelligence
- **Cost Blindness**: No aggregated view of spend across platforms
- **Quality Drift**: Agents degrade over time without monitoring
- **Governance Gaps**: No audit trail or policy enforcement

### 1.3 Solution
A unified platform that:
1. Connects to all major agent platforms via APIs/connectors
2. Provides single-pane-of-glass visibility
3. Captures and ports context across agents
4. Monitors quality and detects drift
5. Tracks and optimizes costs
6. Enforces governance policies

---

## 2. Product Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           AI-ARM PLATFORM                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │                     PRESENTATION LAYER                          │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │     │
│  │  │  Web Console │  │   REST API   │  │  Webhooks    │         │     │
│  │  │  (Next.js)   │  │  (GraphQL)   │  │  (Events)    │         │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘         │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                    │                                     │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │                     APPLICATION LAYER                           │     │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │     │
│  │  │ Agent   │ │ Context │ │ Quality │ │  Cost   │ │Governance│  │     │
│  │  │ Service │ │ Service │ │ Service │ │ Service │ │ Service  │  │     │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘  │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                    │                                     │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │                      DATA LAYER                                 │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │     │
│  │  │PostgreSQL│  │  Redis   │  │ Pinecone │  │TimescaleDB│       │     │
│  │  │ (Core)   │  │ (Cache)  │  │ (Vectors)│  │ (Metrics) │       │     │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                    │                                     │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │                   CONNECTOR LAYER                               │     │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐       │     │
│  │  │ OpenAI │ │Anthropic│ │ Google │ │ Azure  │ │ Custom │       │     │
│  │  │Connector│ │Connector│ │Connector│ │Connector│ │ SDK   │       │     │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘       │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            ┌───────────┐   ┌───────────┐   ┌───────────┐
            │  OpenAI   │   │ Anthropic │   │  Custom   │
            │  Platform │   │  Platform │   │  Agents   │
            └───────────┘   └───────────┘   └───────────┘
```

### 2.2 Technology Stack

#### Frontend
| Component | Technology | Rationale |
|-----------|------------|-----------|
| Framework | Next.js 14 (App Router) | SSR, performance, ecosystem |
| UI Library | Tailwind CSS + shadcn/ui | Rapid development, consistent design |
| State Management | Zustand + React Query | Simple global state + server state |
| Charts | Tremor + Recharts | Dashboard-ready components |
| Real-time | Socket.io | Live updates for monitoring |

#### Backend
| Component | Technology | Rationale |
|-----------|------------|-----------|
| Runtime | Node.js 20 LTS | Team expertise, ecosystem |
| Framework | Fastify + tRPC | Performance, type safety |
| API | GraphQL (Pothos) + REST | Flexible queries + simple endpoints |
| Queue | BullMQ + Redis | Background jobs, event processing |
| Scheduler | node-cron | Periodic health checks |

#### Data Storage
| Component | Technology | Rationale |
|-----------|------------|-----------|
| Primary DB | PostgreSQL 15 (Supabase) | Relational data, JSONB flexibility |
| Cache | Redis (Upstash) | Session, rate limiting, queue |
| Vector DB | Pinecone | Context embeddings, similarity search |
| Time-series | TimescaleDB | Metrics, usage data |
| Object Storage | S3 / Cloudflare R2 | Logs, exports |

#### Infrastructure
| Component | Technology | Rationale |
|-----------|------------|-----------|
| Hosting | Vercel (FE) + Railway (BE) | Simplicity, scaling |
| Container | Docker | Consistent environments |
| CI/CD | GitHub Actions | Integration with repo |
| Monitoring | Datadog + Sentry | APM, error tracking |
| Secrets | Doppler | Secure secret management |

---

## 3. Core Services Specification

### 3.1 Agent Service

#### Purpose
Manage agent inventory, connections, and health monitoring across all vendors.

#### Data Model
```typescript
interface Agent {
  id: string;                    // UUID
  organizationId: string;        // Tenant ID
  name: string;                  // Display name
  vendor: AgentVendor;           // OPENAI | ANTHROPIC | GOOGLE | AZURE | CUSTOM
  vendorAgentId: string;         // ID in vendor's system
  type: AgentType;               // ASSISTANT | CHAT | COMPLETION | CUSTOM
  status: AgentStatus;           // ACTIVE | PAUSED | ERROR | UNKNOWN
  config: AgentConfig;           // Vendor-specific configuration
  metadata: Record<string, any>; // Custom metadata
  createdAt: DateTime;
  updatedAt: DateTime;
  lastHealthCheck: DateTime;
  healthScore: number;           // 0-100
}

interface AgentConfig {
  model: string;                 // gpt-4, claude-3, etc.
  systemPrompt?: string;
  temperature?: number;
  maxTokens?: number;
  tools?: AgentTool[];
  customConfig?: Record<string, any>;
}

interface AgentHealthCheck {
  id: string;
  agentId: string;
  timestamp: DateTime;
  status: HealthStatus;          // HEALTHY | DEGRADED | DOWN | UNKNOWN
  latencyMs: number;
  errorRate: number;             // 0-1
  details: HealthDetails;
}
```

#### API Endpoints
```
GET    /api/v1/agents                    # List all agents
POST   /api/v1/agents                    # Register new agent
GET    /api/v1/agents/:id                # Get agent details
PATCH  /api/v1/agents/:id                # Update agent
DELETE /api/v1/agents/:id                # Remove agent
POST   /api/v1/agents/:id/health-check   # Trigger health check
GET    /api/v1/agents/:id/health         # Get health history
POST   /api/v1/agents/discover           # Auto-discover agents from vendor
```

#### Connector Interface
```typescript
interface AgentConnector {
  vendor: AgentVendor;

  // Discovery
  discoverAgents(credentials: VendorCredentials): Promise<Agent[]>;

  // Health
  checkHealth(agent: Agent): Promise<AgentHealthCheck>;

  // Usage
  getUsage(agent: Agent, period: DateRange): Promise<UsageData>;

  // Interactions
  getInteractions(agent: Agent, options: QueryOptions): Promise<Interaction[]>;

  // Control
  pauseAgent(agent: Agent): Promise<void>;
  resumeAgent(agent: Agent): Promise<void>;
}
```

---

### 3.2 Context Service (Context Vault)

#### Purpose
Capture, store, and port agent context and memory across vendors.

#### Data Model
```typescript
interface Context {
  id: string;
  organizationId: string;
  agentId: string;
  type: ContextType;             // CONVERSATION | PREFERENCE | FACT | INSTRUCTION
  content: string;               // The actual context
  embedding: number[];           // Vector embedding for similarity
  source: ContextSource;         // How it was captured
  confidence: number;            // 0-1 extraction confidence
  validFrom: DateTime;
  validUntil?: DateTime;         // For time-limited context
  metadata: Record<string, any>;
  createdAt: DateTime;
}

interface ContextSource {
  type: 'INTERACTION' | 'MANUAL' | 'IMPORT' | 'INFERENCE';
  interactionId?: string;
  importedFrom?: string;
}

interface Preference {
  id: string;
  organizationId: string;
  userId?: string;               // User-specific preferences
  category: string;              // communication_style, format, etc.
  key: string;
  value: string;
  examples: string[];            // Supporting examples
  confidence: number;
  lastObserved: DateTime;
}

interface ContextExport {
  id: string;
  organizationId: string;
  sourceAgentId: string;
  format: ExportFormat;          // ARM_NATIVE | OPENAI | ANTHROPIC | JSON
  contexts: Context[];
  preferences: Preference[];
  exportedAt: DateTime;
  checksum: string;
}
```

#### API Endpoints
```
GET    /api/v1/context                        # List contexts
POST   /api/v1/context                        # Add context manually
GET    /api/v1/context/:id                    # Get context
DELETE /api/v1/context/:id                    # Remove context

GET    /api/v1/context/search                 # Semantic search
POST   /api/v1/context/extract                # Extract from interaction

POST   /api/v1/context/export                 # Export for migration
POST   /api/v1/context/import                 # Import from export

GET    /api/v1/preferences                    # List preferences
POST   /api/v1/preferences/inject/:agentId    # Inject into agent
```

#### Context Extraction Pipeline
```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Interaction │───▶│  Extraction  │───▶│  Embedding   │───▶│   Storage    │
│   Received   │    │    (LLM)     │    │  Generation  │    │  (Pinecone)  │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                           │
                    ┌──────▼──────┐
                    │  Extracted  │
                    │  - Facts    │
                    │  - Prefs    │
                    │  - Intent   │
                    └─────────────┘
```

---

### 3.3 Quality Service

#### Purpose
Monitor agent quality, detect drift, hallucinations, and degradation.

#### Data Model
```typescript
interface QualityMetric {
  id: string;
  agentId: string;
  timestamp: DateTime;
  metricType: QualityMetricType;
  value: number;                  // Normalized 0-1
  rawValue: any;                  // Original measurement
  sampleSize: number;
  details: Record<string, any>;
}

enum QualityMetricType {
  HALLUCINATION_RATE = 'hallucination_rate',
  RELEVANCE_SCORE = 'relevance_score',
  CONSISTENCY_SCORE = 'consistency_score',
  LATENCY_P50 = 'latency_p50',
  LATENCY_P99 = 'latency_p99',
  ERROR_RATE = 'error_rate',
  USER_SATISFACTION = 'user_satisfaction',
  TASK_COMPLETION = 'task_completion'
}

interface QualityAlert {
  id: string;
  agentId: string;
  alertType: AlertType;
  severity: Severity;            // INFO | WARNING | CRITICAL
  metric: QualityMetricType;
  threshold: number;
  actualValue: number;
  message: string;
  status: AlertStatus;           // OPEN | ACKNOWLEDGED | RESOLVED
  createdAt: DateTime;
  resolvedAt?: DateTime;
}

interface DriftEvent {
  id: string;
  agentId: string;
  detectedAt: DateTime;
  driftType: DriftType;          // ANSWER | BEHAVIOR | PERFORMANCE
  baselineValue: number;
  currentValue: number;
  deviation: number;             // Standard deviations from baseline
  samples: string[];             // Example interactions showing drift
}
```

#### API Endpoints
```
GET    /api/v1/quality/:agentId/metrics       # Get quality metrics
GET    /api/v1/quality/:agentId/alerts        # Get alerts
POST   /api/v1/quality/:agentId/alerts/:id/ack # Acknowledge alert
GET    /api/v1/quality/:agentId/drift         # Get drift events
POST   /api/v1/quality/:agentId/evaluate      # Trigger quality evaluation
GET    /api/v1/quality/:agentId/report        # Generate quality report
```

#### Hallucination Detection
```typescript
interface HallucinationCheck {
  interactionId: string;
  response: string;
  groundingSources: string[];     // RAG sources, if any

  // Detection results
  hallucinationScore: number;     // 0-1
  flaggedClaims: FlaggedClaim[];
  verdict: 'CLEAN' | 'SUSPICIOUS' | 'HALLUCINATION';
}

interface FlaggedClaim {
  claim: string;
  startIndex: number;
  endIndex: number;
  reason: string;
  confidence: number;
  suggestedCorrection?: string;
}
```

---

### 3.4 Cost Service

#### Purpose
Track, analyze, and optimize costs across all agent platforms.

#### Data Model
```typescript
interface UsageRecord {
  id: string;
  agentId: string;
  timestamp: DateTime;
  periodType: PeriodType;         // HOUR | DAY | MONTH

  // Token metrics
  inputTokens: number;
  outputTokens: number;
  totalTokens: number;

  // Cost metrics
  inputCost: Decimal;
  outputCost: Decimal;
  totalCost: Decimal;
  currency: string;               // USD

  // Volume metrics
  requestCount: number;
  errorCount: number;

  // Breakdown
  modelBreakdown: ModelUsage[];
}

interface ModelUsage {
  model: string;
  tokens: number;
  cost: Decimal;
  requests: number;
}

interface CostAlert {
  id: string;
  organizationId: string;
  agentId?: string;               // null = org-wide
  alertType: CostAlertType;       // THRESHOLD | ANOMALY | FORECAST
  threshold?: Decimal;
  forecastPeriod?: string;        // e.g., "7d"
  triggeredAt: DateTime;
  actualCost: Decimal;
  message: string;
}

interface CostOptimization {
  id: string;
  agentId: string;
  optimizationType: OptimizationType;
  currentCost: Decimal;
  projectedSavings: Decimal;
  recommendation: string;
  implementation: string;         // How to implement
  effort: 'LOW' | 'MEDIUM' | 'HIGH';
  impact: 'LOW' | 'MEDIUM' | 'HIGH';
}
```

#### API Endpoints
```
GET    /api/v1/costs/usage                    # Get usage data
GET    /api/v1/costs/summary                  # Cost summary by period
GET    /api/v1/costs/breakdown                # Breakdown by agent/model
GET    /api/v1/costs/forecast                 # Cost forecast
GET    /api/v1/costs/alerts                   # Cost alerts
POST   /api/v1/costs/alerts                   # Create cost alert
GET    /api/v1/costs/optimizations            # Get optimization suggestions
POST   /api/v1/costs/budget                   # Set budget limits
```

---

### 3.5 Governance Service

#### Purpose
Enforce policies, maintain audit trails, and ensure compliance.

#### Data Model
```typescript
interface AuditLog {
  id: string;
  organizationId: string;
  timestamp: DateTime;

  // Actor
  actorType: ActorType;           // USER | AGENT | SYSTEM
  actorId: string;
  actorName: string;

  // Action
  action: AuditAction;
  resourceType: string;
  resourceId: string;

  // Details
  before?: Record<string, any>;
  after?: Record<string, any>;
  metadata: Record<string, any>;

  // Request context
  ipAddress?: string;
  userAgent?: string;
  requestId: string;
}

interface Policy {
  id: string;
  organizationId: string;
  name: string;
  description: string;
  type: PolicyType;
  rules: PolicyRule[];
  enforcement: 'AUDIT' | 'WARN' | 'BLOCK';
  enabled: boolean;
  createdAt: DateTime;
  updatedAt: DateTime;
}

interface PolicyRule {
  id: string;
  condition: PolicyCondition;
  action: PolicyAction;
}

interface PolicyCondition {
  field: string;                  // e.g., "request.content", "agent.type"
  operator: Operator;             // CONTAINS | EQUALS | MATCHES | GT | LT
  value: any;
}

interface PolicyViolation {
  id: string;
  policyId: string;
  agentId: string;
  interactionId?: string;
  timestamp: DateTime;
  rule: PolicyRule;
  matchedContent: string;
  severity: Severity;
  actionTaken: string;
  reviewed: boolean;
}
```

#### API Endpoints
```
GET    /api/v1/governance/audit               # Query audit logs
GET    /api/v1/governance/audit/export        # Export audit logs

GET    /api/v1/governance/policies            # List policies
POST   /api/v1/governance/policies            # Create policy
PATCH  /api/v1/governance/policies/:id        # Update policy
DELETE /api/v1/governance/policies/:id        # Delete policy

GET    /api/v1/governance/violations          # List violations
POST   /api/v1/governance/violations/:id/review # Review violation

GET    /api/v1/governance/report              # Compliance report
```

---

## 4. MVP Feature Specification

### 4.1 MVP Scope (v1.0)

#### Must Have (P0)
| Feature | Description | Acceptance Criteria |
|---------|-------------|---------------------|
| Agent Inventory | List all agents across vendors | View agents from OpenAI + Anthropic |
| Agent Connection | Connect to vendor APIs | OAuth/API key setup, test connection |
| Health Dashboard | See agent status | Up/down status, last check time |
| Basic Metrics | View key metrics | Token usage, request count, error rate |
| Cost Tracking | See costs by agent | Daily/monthly cost, breakdown by agent |
| Alerting | Get notified of issues | Email alerts for errors, cost spikes |
| User Auth | Secure access | Login, basic RBAC (Admin/Member) |

#### Should Have (P1)
| Feature | Description | Acceptance Criteria |
|---------|-------------|---------------------|
| Context Capture | Extract context from interactions | Auto-extract facts and preferences |
| Context Search | Find relevant context | Semantic search across context |
| Quality Score | Basic quality metrics | Latency, error rate tracking |
| Cost Alerts | Budget threshold alerts | Alert when spend exceeds threshold |
| Basic Export | Export data | CSV export of agents, costs |

#### Nice to Have (P2)
| Feature | Description | Acceptance Criteria |
|---------|-------------|---------------------|
| Context Migration | Port context between agents | Export/import context format |
| Drift Detection | Detect quality changes | Alert on significant drift |
| Google Connector | Support Google AI | Connect to Vertex AI agents |
| API Access | Programmatic access | REST API with auth |

### 4.2 User Stories (MVP)

#### US-001: Connect Agent Platform
```
As an Admin
I want to connect my OpenAI organization to AI-ARM
So that I can see all my agents in one place

Acceptance Criteria:
- Can enter OpenAI API key securely
- System validates the API key
- All assistants are discovered and listed
- Connection status is shown
- Can disconnect if needed
```

#### US-002: View Agent Inventory
```
As a User
I want to see all agents across all connected platforms
So that I have a single view of my agent landscape

Acceptance Criteria:
- Dashboard shows all agents in a list/grid
- Can filter by vendor, status, type
- Can search by agent name
- Shows agent count per vendor
- Shows health status indicator
```

#### US-003: Monitor Agent Health
```
As a User
I want to see if my agents are healthy
So that I can identify and fix issues quickly

Acceptance Criteria:
- Each agent shows health status (Healthy/Degraded/Down)
- Health check runs every 5 minutes
- Can see health history (last 24h)
- Can trigger manual health check
- Shows last error if any
```

#### US-004: Track Costs
```
As an Admin
I want to see how much each agent costs
So that I can manage my AI spending

Acceptance Criteria:
- Dashboard shows total cost (today/week/month)
- Can see cost breakdown by agent
- Can see cost breakdown by model
- Can see cost trend over time
- Shows token usage alongside cost
```

#### US-005: Receive Alerts
```
As a User
I want to be notified when something goes wrong
So that I can respond quickly to issues

Acceptance Criteria:
- Can configure alert rules (error rate, cost threshold)
- Receives email when alert triggers
- Can see alert history in dashboard
- Can acknowledge/resolve alerts
- Can configure alert recipients
```

---

## 5. API Specification

### 5.1 Authentication

```
# All API requests require authentication

# Option 1: API Key (for programmatic access)
Authorization: Bearer arm_sk_live_xxxxx

# Option 2: Session Cookie (for web console)
Cookie: arm_session=xxxxx
```

### 5.2 Core API Endpoints (MVP)

```yaml
# Health
GET /api/health                    # System health check

# Auth
POST /api/auth/login               # Login with email/password
POST /api/auth/logout              # Logout
GET  /api/auth/me                  # Get current user

# Organizations
GET  /api/v1/organizations/:id     # Get organization
PATCH /api/v1/organizations/:id    # Update organization

# Connections (Vendor credentials)
GET  /api/v1/connections           # List connections
POST /api/v1/connections           # Add connection
DELETE /api/v1/connections/:id     # Remove connection
POST /api/v1/connections/:id/test  # Test connection

# Agents
GET  /api/v1/agents                # List agents
GET  /api/v1/agents/:id            # Get agent
POST /api/v1/agents/:id/health     # Check health

# Metrics
GET  /api/v1/metrics/usage         # Get usage metrics
GET  /api/v1/metrics/costs         # Get cost metrics
GET  /api/v1/metrics/health        # Get health metrics

# Alerts
GET  /api/v1/alerts                # List alerts
POST /api/v1/alerts                # Create alert rule
PATCH /api/v1/alerts/:id           # Update alert
DELETE /api/v1/alerts/:id          # Delete alert
POST /api/v1/alerts/:id/ack        # Acknowledge alert
```

### 5.3 Response Format

```json
// Success response
{
  "success": true,
  "data": { ... },
  "meta": {
    "page": 1,
    "perPage": 20,
    "total": 100
  }
}

// Error response
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid API key format",
    "details": { ... }
  }
}
```

---

## 6. Security Requirements

### 6.1 Authentication & Authorization
- [ ] OAuth 2.0 / OIDC for SSO (Google, Microsoft)
- [ ] Email/password with strong password requirements
- [ ] MFA support (TOTP)
- [ ] Role-based access control (Admin, Member, Viewer)
- [ ] API key authentication for programmatic access
- [ ] Session management with secure cookies

### 6.2 Data Protection
- [ ] All data encrypted at rest (AES-256)
- [ ] All data encrypted in transit (TLS 1.3)
- [ ] Vendor API keys encrypted with per-org keys
- [ ] PII handling according to GDPR/CCPA
- [ ] Data residency options (US, EU)

### 6.3 Infrastructure Security
- [ ] VPC isolation
- [ ] WAF protection
- [ ] DDoS mitigation
- [ ] Regular penetration testing
- [ ] Vulnerability scanning in CI/CD

### 6.4 Compliance Targets
- [ ] SOC 2 Type 1 (MVP+3 months)
- [ ] SOC 2 Type 2 (MVP+12 months)
- [ ] GDPR compliance (MVP)
- [ ] HIPAA readiness (Year 2)

---

## 7. Performance Requirements

### 7.1 Latency
| Operation | Target | Max |
|-----------|--------|-----|
| Dashboard load | <2s | <5s |
| Agent list | <500ms | <2s |
| Health check | <5s | <30s |
| Search | <200ms | <1s |
| API calls | <100ms | <500ms |

### 7.2 Throughput
| Metric | MVP Target | Scale Target |
|--------|------------|--------------|
| Concurrent users | 100 | 10,000 |
| Agents managed | 10,000 | 1,000,000 |
| API requests/sec | 100 | 10,000 |
| Events processed/sec | 1,000 | 100,000 |

### 7.3 Availability
- Target: 99.9% uptime (8.7h downtime/year)
- Maintenance windows: Sundays 2-4am UTC
- RTO: 4 hours
- RPO: 1 hour

---

## 8. Database Schema (Core Tables)

```sql
-- Organizations
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255),
  name VARCHAR(255),
  role VARCHAR(50) DEFAULT 'member',
  organization_id UUID REFERENCES organizations(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Vendor Connections
CREATE TABLE connections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id),
  vendor VARCHAR(50) NOT NULL,
  credentials_encrypted BYTEA NOT NULL,
  status VARCHAR(50) DEFAULT 'pending',
  last_sync_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Agents
CREATE TABLE agents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id),
  connection_id UUID REFERENCES connections(id),
  vendor VARCHAR(50) NOT NULL,
  vendor_agent_id VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  type VARCHAR(50),
  status VARCHAR(50) DEFAULT 'unknown',
  config JSONB DEFAULT '{}',
  metadata JSONB DEFAULT '{}',
  health_score INTEGER DEFAULT 0,
  last_health_check TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(organization_id, vendor, vendor_agent_id)
);

-- Health Checks
CREATE TABLE health_checks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID REFERENCES agents(id),
  status VARCHAR(50) NOT NULL,
  latency_ms INTEGER,
  error_rate DECIMAL(5,4),
  details JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Usage Records (partitioned by time)
CREATE TABLE usage_records (
  id UUID DEFAULT gen_random_uuid(),
  agent_id UUID NOT NULL,
  period_start TIMESTAMPTZ NOT NULL,
  period_type VARCHAR(20) NOT NULL,
  input_tokens BIGINT DEFAULT 0,
  output_tokens BIGINT DEFAULT 0,
  total_cost DECIMAL(12,6) DEFAULT 0,
  request_count INTEGER DEFAULT 0,
  error_count INTEGER DEFAULT 0,
  model_breakdown JSONB DEFAULT '[]',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (id, period_start)
) PARTITION BY RANGE (period_start);

-- Alerts
CREATE TABLE alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id),
  agent_id UUID REFERENCES agents(id),
  alert_type VARCHAR(50) NOT NULL,
  severity VARCHAR(20) NOT NULL,
  condition JSONB NOT NULL,
  message TEXT,
  status VARCHAR(50) DEFAULT 'active',
  triggered_at TIMESTAMPTZ,
  acknowledged_at TIMESTAMPTZ,
  resolved_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Context (for Context Vault)
CREATE TABLE contexts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id),
  agent_id UUID REFERENCES agents(id),
  type VARCHAR(50) NOT NULL,
  content TEXT NOT NULL,
  embedding vector(1536),
  confidence DECIMAL(3,2),
  source JSONB DEFAULT '{}',
  valid_from TIMESTAMPTZ DEFAULT NOW(),
  valid_until TIMESTAMPTZ,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Audit Log
CREATE TABLE audit_logs (
  id UUID DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  actor_type VARCHAR(50) NOT NULL,
  actor_id VARCHAR(255) NOT NULL,
  action VARCHAR(100) NOT NULL,
  resource_type VARCHAR(100),
  resource_id VARCHAR(255),
  details JSONB DEFAULT '{}',
  ip_address INET,
  PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

-- Indexes
CREATE INDEX idx_agents_org ON agents(organization_id);
CREATE INDEX idx_agents_vendor ON agents(vendor);
CREATE INDEX idx_agents_status ON agents(status);
CREATE INDEX idx_health_agent ON health_checks(agent_id);
CREATE INDEX idx_health_time ON health_checks(created_at);
CREATE INDEX idx_usage_agent ON usage_records(agent_id);
CREATE INDEX idx_alerts_org ON alerts(organization_id);
CREATE INDEX idx_alerts_status ON alerts(status);
CREATE INDEX idx_contexts_embedding ON contexts USING ivfflat (embedding vector_cosine_ops);
CREATE INDEX idx_audit_org_time ON audit_logs(organization_id, timestamp);
```

---

## 9. Development Milestones

### Milestone 1: Foundation (Week 1-2)
- [ ] Project setup (repo, CI/CD, environments)
- [ ] Database schema and migrations
- [ ] Basic auth (email/password)
- [ ] Organization/user management
- [ ] API scaffolding

### Milestone 2: Connectors (Week 3-4)
- [ ] OpenAI connector (discover, health, usage)
- [ ] Anthropic connector (discover, health, usage)
- [ ] Connection management UI
- [ ] Agent sync job

### Milestone 3: Dashboard (Week 5-6)
- [ ] Agent inventory view
- [ ] Health status display
- [ ] Basic metrics charts
- [ ] Real-time status updates

### Milestone 4: Costs & Alerts (Week 7-8)
- [ ] Cost tracking implementation
- [ ] Cost dashboard
- [ ] Alert rules engine
- [ ] Email notifications

### Milestone 5: Polish & Testing (Week 9-10)
- [ ] UI/UX polish
- [ ] Error handling
- [ ] Integration tests
- [ ] Performance optimization

### Milestone 6: Beta Launch (Week 11-12)
- [ ] Security audit
- [ ] Documentation
- [ ] Design partner onboarding
- [ ] Feedback collection

---

## 10. Success Metrics

### Technical Metrics
| Metric | Target |
|--------|--------|
| Test coverage | >80% |
| Build time | <5 min |
| Deploy time | <10 min |
| API error rate | <0.1% |
| P99 latency | <500ms |

### Product Metrics (MVP)
| Metric | Week 4 | Week 8 | Week 12 |
|--------|--------|--------|---------|
| Agents connected | 100 | 500 | 2,000 |
| Daily active users | 10 | 30 | 50 |
| Health checks/day | 1K | 5K | 20K |
| Alerts sent | 50 | 200 | 500 |

---

## Appendix A: Vendor API Reference

### OpenAI APIs Used
- `GET /v1/assistants` - List assistants
- `GET /v1/threads` - List threads
- `GET /v1/usage` - Usage data
- `GET /v1/organization/usage` - Org usage

### Anthropic APIs Used
- Admin API (beta) for organization management
- Usage API for token consumption
- Messages API for health checks

### Future Integrations
- Google Vertex AI Agent Builder
- Azure OpenAI Service
- LangSmith (LangChain observability)
- Custom agents via webhook/SDK

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| Agent | An AI assistant/bot deployed on a vendor platform |
| Connector | Module that integrates with a specific vendor API |
| Context | Captured memory, preferences, facts from interactions |
| Drift | Gradual degradation in agent quality over time |
| Health Check | Periodic verification of agent availability and performance |
| Vendor | Platform providing AI agents (OpenAI, Anthropic, etc.) |

---

**Document Version**: 1.0
**Last Updated**: February 2025
**Next Review**: After MVP launch
