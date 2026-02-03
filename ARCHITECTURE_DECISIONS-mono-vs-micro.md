# Budget App Architecture Decisions

## Project: fatsql-dbre
## Date: January 2026

---

## Decision 1: VPC Strategy

### Chosen Approach: Shared VPC with Subnet Isolation

```
fatsql-dbre Project Architecture
================================

VPC: fatsql-vpc (10.0.0.0/16)
│
├── Subnet: budget-app (10.1.0.0/24) - us-central1
│   ├── Cloud Run services via VPC Connector
│   ├── Budget app resources only
│   └── Firewall: budget-app-rules
│
├── Subnet: dbre-tools (10.2.0.0/24) - us-central1  
│   ├── Existing DBRE tools
│   └── Firewall: dbre-tools-rules
│
├── Subnet: shared-services (10.3.0.0/24) - us-central1
│   ├── Monitoring
│   ├── Logging aggregation
│   └── Shared utilities
│
└── Cloud SQL Private Service Connection
    └── Automatic IP range (10.100.0.0/20)
```

### Firewall Rules

```yaml
budget-app-egress:
  - Allow: budget-app subnet → Cloud SQL
  - Allow: budget-app subnet → Internet (for external APIs)
  - Deny: budget-app subnet → dbre-tools subnet

budget-app-ingress:
  - Allow: Load Balancer → Cloud Run (budget app)
  - Deny: dbre-tools → budget app (unless explicitly needed)

dbre-tools-isolation:
  - Allow: dbre-tools → Internet
  - Allow: dbre-tools → Cloud SQL (if needed)
  - Deny: dbre-tools → budget-app (default)
```

### Cost Analysis

**Shared VPC:**
- VPC Connector: $52/mo (shared across apps)
- Cloud NAT: $45/mo (shared)
- Firewall Rules: Free
- **Total: ~$97/mo** (amortized across all apps)

**Separate VPC (alternative):**
- VPC Connector: $52/mo per app
- Cloud NAT: $45/mo per app
- **Total: ~$97/mo per app**

### Recommendation

✅ **Use Shared VPC** because:
1. You're in the same GCP project
2. Cost efficiency (share infrastructure)
3. Proper isolation via subnets + firewall rules
4. Easier to add more apps later
5. Shared services (monitoring, logging)

---

## Decision 2: Application Architecture

We'll provide **TWO deployment options**:

### Option A: Monolithic (Recommended for MVP)

```
┌─────────────────────────────────────────┐
│     Single Next.js Application          │
│  ┌────────────────────────────────────┐ │
│  │     Frontend (React)               │ │
│  │  - Server Components               │ │
│  │  - Client Components               │ │
│  │  - Static Pages                    │ │
│  └────────────────────────────────────┘ │
│  ┌────────────────────────────────────┐ │
│  │     Backend (API Routes)           │ │
│  │  - Authentication                  │ │
│  │  - Transactions API                │ │
│  │  - Budgets API                     │ │
│  │  - Reports API                     │ │
│  │  - User Management                 │ │
│  └────────────────────────────────────┘ │
│  ┌────────────────────────────────────┐ │
│  │     Business Logic                 │ │
│  │  - Budget Calculations             │ │
│  │  - Savings Analysis                │ │
│  │  - Data Validation                 │ │
│  └────────────────────────────────────┘ │
│  ┌────────────────────────────────────┐ │
│  │     Data Layer (Prisma)            │ │
│  │  - ORM                             │ │
│  │  - Migrations                      │ │
│  │  - Query Builder                   │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
              │
              ↓
      ┌───────────────┐
      │  Cloud SQL    │
      │  PostgreSQL   │
      └───────────────┘

Deployment: Single Cloud Run service
Cost: $7-15/mo (minimal) or $60-80/mo (with VPC)
Terraform: Yes (full IaC)
```

**When to use Monolithic:**
- ✅ MVP / Early stage
- ✅ Small team (1-5 developers)
- ✅ < 1,000 users
- ✅ Need fast iteration
- ✅ Limited budget
- ✅ Simple deployment

---

### Option B: Microservices (Scalable)

```
┌──────────────────────────────────────────────────────┐
│                    Load Balancer                      │
│                   (Cloud Load Balancing)              │
└────┬──────────────────┬────────────────┬─────────────┘
     │                  │                │
     ↓                  ↓                ↓
┌─────────┐      ┌──────────────┐  ┌──────────────┐
│Frontend │      │  API Gateway │  │  Web Socket  │
│ Service │      │   Service    │  │   Service    │
│         │      │              │  │              │
│Next.js  │      │ Cloud Run    │  │  Cloud Run   │
│Static   │      │              │  │              │
│Cloud    │      └──────┬───────┘  └──────────────┘
│Storage  │             │
└─────────┘             │
                        ↓
          ┌─────────────┴─────────────┐
          │                           │
     ┌────▼──────┐              ┌─────▼────────┐
     │   Auth    │              │ Transaction  │
     │  Service  │              │   Service    │
     │           │              │              │
     │Cloud Run  │              │  Cloud Run   │
     └───────────┘              └──────────────┘
          │                           │
          ↓                           ↓
     ┌────────────┐              ┌──────────────┐
     │  Budget    │              │   Reports    │
     │  Service   │              │   Service    │
     │            │              │              │
     │ Cloud Run  │              │  Cloud Run   │
     └─────┬──────┘              └──────┬───────┘
           │                            │
           └────────────┬───────────────┘
                        ↓
              ┌──────────────────┐
              │   Event Bus      │
              │   (Pub/Sub)      │
              └──────────────────┘
                        │
                        ↓
              ┌──────────────────┐
              │   Cloud SQL      │
              │   PostgreSQL     │
              │   (Shared DB)    │
              └──────────────────┘
                        │
                        ↓
              ┌──────────────────┐
              │   Redis Cache    │
              │   (Memorystore)  │
              └──────────────────┘
```

**Service Breakdown:**

1. **Frontend Service** ($0.50/mo)
   - Next.js static export
   - Cloud Storage + CDN
   - SEO optimized

2. **API Gateway Service** ($5-10/mo)
   - Request routing
   - Rate limiting
   - Authentication check

3. **Auth Service** ($5-10/mo)
   - User registration
   - Login/logout
   - Session management
   - JWT tokens

4. **Transaction Service** ($5-10/mo)
   - CRUD transactions
   - Category management
   - Transaction search

5. **Budget Service** ($5-10/mo)
   - Budget creation
   - Budget tracking
   - Budget vs actual

6. **Reports Service** ($5-10/mo)
   - Generate reports
   - Analytics
   - Data export

**Shared Infrastructure:**
- Cloud SQL: $25/mo (db-g1-small)
- Redis: $30/mo (Memorystore basic)
- Pub/Sub: $1-5/mo
- VPC Connector: $52/mo
- Load Balancer: $18/mo

**Total Cost: $145-180/mo**

**When to use Microservices:**
- ✅ Production / Growth stage
- ✅ Team > 5 developers
- ✅ > 1,000 users
- ✅ Need independent scaling
- ✅ Multiple teams
- ✅ Complex business logic

---

## Decision 3: Infrastructure as Code

### Chosen Approach: Terraform for Everything

**Directory Structure:**
```
terraform/
├── modules/
│   ├── vpc/
│   ├── cloudsql/
│   ├── cloudrun/
│   ├── secrets/
│   └── monitoring/
│
├── environments/
│   ├── dev/
│   ├── staging/
│   └── production/
│
├── monolithic/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars
│
└── microservices/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── terraform.tfvars
```

### Benefits of Terraform:
1. Version controlled infrastructure
2. Repeatable deployments
3. Easy to destroy and recreate
4. Environment parity (dev/staging/prod)
5. Documentation as code

---

## Decision 4: Database Strategy

### Approach: Shared Database (Initially)

Both monolithic and microservices will use:
- **Single PostgreSQL instance**
- **Schema-per-service** (for microservices)
- **Connection pooling** via Prisma

```sql
Database: budgetapp
├── Schema: public (monolithic)
│   └── All tables
│
├── Schema: auth (microservices)
│   ├── users
│   ├── sessions
│   └── accounts
│
├── Schema: transactions (microservices)
│   ├── transactions
│   └── categories
│
├── Schema: budgets (microservices)
│   ├── budgets
│   └── budget_items
│
└── Schema: reports (microservices)
    └── report_cache
```

### Migration Path:
1. **Phase 1**: Monolithic (single schema)
2. **Phase 2**: Split schemas in same DB
3. **Phase 3**: Separate databases per service (if needed)

---

## Decision 5: Deployment Strategy

### CI/CD Pipeline (GitLab)

**Monolithic:**
```yaml
Stages:
  1. Test (lint, type-check, unit tests)
  2. Build (Docker image)
  3. Deploy Dev (auto)
  4. Deploy Staging (auto on main)
  5. Deploy Production (manual approval)
```

**Microservices:**
```yaml
Stages:
  1. Test (per service)
  2. Build (parallel builds)
  3. Deploy Dev (auto per service)
  4. Integration Tests
  5. Deploy Staging (auto)
  6. Deploy Production (manual, rolling)
```

---

## Summary of Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **VPC Strategy** | Shared VPC with subnets | Cost efficient, proper isolation |
| **Network** | Private IP for Cloud SQL | Security, no internet exposure |
| **Architecture** | Both monolithic & microservices | Start simple, scale when needed |
| **IaC Tool** | Terraform | Industry standard, repeatable |
| **Database** | Shared PostgreSQL | Cost efficient, easy to split later |
| **CI/CD** | GitLab CI/CD | Already using GitLab |
| **Monitoring** | Cloud Monitoring | Native GCP integration |

---

## Implementation Plan

### Phase 1: Foundation (Week 1)
- [ ] Set up shared VPC with Terraform
- [ ] Create subnets and firewall rules
- [ ] Set up Cloud SQL with private IP
- [ ] Configure VPC Connector
- [ ] Test network connectivity

### Phase 2: Monolithic Deployment (Week 2)
- [ ] Deploy monolithic app
- [ ] Set up CI/CD pipeline
- [ ] Configure monitoring
- [ ] Load testing
- [ ] Documentation

### Phase 3: Microservices Preparation (Week 3-4)
- [ ] Design service boundaries
- [ ] Create service Terraform modules
- [ ] Set up service mesh (optional)
- [ ] Deploy microservices
- [ ] Integration testing

### Phase 4: Production (Week 5)
- [ ] Security audit
- [ ] Performance optimization
- [ ] Disaster recovery plan
- [ ] Go live!

---

## Cost Comparison

| Setup | Monthly Cost | Best For |
|-------|-------------|----------|
| **Monolithic (minimal)** | $7-15 | Development, testing |
| **Monolithic (production)** | $60-80 | MVP, <1000 users |
| **Microservices** | $145-180 | Production, >1000 users |

---

## Recommendation

**For Budget App Launch:**

1. **Start with Monolithic** architecture
   - Deploy on shared VPC
   - Use private IP for Cloud SQL
   - Cost: ~$60-80/mo

2. **Prepare Microservices** infrastructure with Terraform
   - Keep it ready
   - Don't deploy yet
   - Switch when needed

3. **Migration Trigger Points:**
   - Team size > 5 developers
   - User base > 1,000 active users
   - Need for independent service scaling
   - Complex feature requirements

---

This approach gives you:
✅ Production-ready infrastructure  
✅ Cost efficiency to start  
✅ Clear path to scale  
✅ Infrastructure as code  
✅ Proper isolation and security  

