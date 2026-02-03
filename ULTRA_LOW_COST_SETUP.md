# Ultra-Low-Cost Budget App Deployment ($10-15/month)

## 🎯 Goal: Deploy for Under $15/month

This guide focuses on **absolute minimum cost** while maintaining functionality.

---

## 💰 Cost Breakdown (Target: $10-15/month)

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| Cloud SQL | db-f1-micro (shared), 10GB SSD | $7.00 |
| Cloud Run | 512MB RAM, scale-to-zero | $0-3.00 |
| Artifact Registry | <1GB storage | $0.10 |
| Cloud Build | Free tier (120 min/mo) | $0.00 |
| Secret Manager | 3 secrets, 6 access/mo | $0.00 |
| Networking | No VPC connector | $0.00 |
| **TOTAL** | | **$7-10/month** |

---

## 🚀 Quick Setup (30 minutes)

### Step 1: Minimal GCP Setup

```bash
export PROJECT_ID="budget-app-prod"
export REGION="us-central1"

# Create project
gcloud projects create $PROJECT_ID --name="Budget App"
gcloud config set project $PROJECT_ID

# Enable only essential APIs
gcloud services enable \
  run.googleapis.com \
  sqladmin.googleapis.com \
  cloudbuild.googleapis.com \
  secretmanager.googleapis.com \
  artifactregistry.googleapis.com
```

### Step 2: Ultra-Cheap Database

```bash
# Generate password
export DB_PASSWORD="$(openssl rand -base64 32)"
echo "Save this: $DB_PASSWORD"

# Create SMALLEST instance WITH PUBLIC IP (avoids VPC connector cost)
gcloud sql instances create budget-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=$REGION \
  --assign-ip \
  --authorized-networks=0.0.0.0/0 \
  --root-password="$DB_PASSWORD" \
  --backup-start-time=03:00 \
  --retained-backups-count=1 \
  --no-enable-point-in-time-recovery

# Create database
gcloud sql databases create budgetapp --instance=budget-db
```

**⚠️ Security Note:** Public IP with 0.0.0.0/0 is not ideal for production. For better security, only allow Cloud Run IP ranges or use Cloud SQL Auth Proxy.

### Step 3: Store Secrets

```bash
# Store password
echo -n "$DB_PASSWORD" | gcloud secrets create db-password \
  --data-file=- --replication-policy="automatic"

# Create NextAuth secret
echo -n "$(openssl rand -base64 32)" | gcloud secrets create nextauth-secret \
  --data-file=- --replication-policy="automatic"

# Get Cloud SQL connection details
export PUBLIC_IP=$(gcloud sql instances describe budget-db --format='value(ipAddresses[0].ipAddress)')
export DB_URL="postgresql://postgres:${DB_PASSWORD}@${PUBLIC_IP}:5432/budgetapp"

# Store DB URL
echo -n "$DB_URL" | gcloud secrets create db-url \
  --data-file=- --replication-policy="automatic"
```

### Step 4: Create Artifact Registry

```bash
gcloud artifacts repositories create budget-app \
  --repository-format=docker \
  --location=$REGION \
  --description="Budget App"
```

### Step 5: Service Account (Minimal Permissions)

```bash
# Create service account
gcloud iam service-accounts create gitlab-cicd \
  --display-name="GitLab CI/CD"

export SA_EMAIL="gitlab-cicd@${PROJECT_ID}.iam.gserviceaccount.com"

# Minimal required roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/run.developer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/cloudsql.client"

# Create key for GitLab
gcloud iam service-accounts keys create gitlab-key.json \
  --iam-account=$SA_EMAIL

# Base64 encode
cat gitlab-key.json | base64 -w 0 > gitlab-key-base64.txt

echo "Copy this to GitLab variable GCP_SERVICE_KEY:"
cat gitlab-key-base64.txt
```

### Step 6: Simplified GitLab CI/CD

Create `.gitlab-ci.yml` in your repository:

```yaml
variables:
  GCP_PROJECT_ID: "budget-app-prod"
  GCP_REGION: "us-central1"
  IMAGE_NAME: "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/budget-app/app"

stages:
  - build
  - deploy

build:
  stage: build
  image: google/cloud-sdk:alpine
  services:
    - docker:24-dind
  only:
    - main
  before_script:
    - echo $GCP_SERVICE_KEY | base64 -d > gcp-key.json
    - gcloud auth activate-service-account --key-file gcp-key.json
    - gcloud config set project $GCP_PROJECT_ID
    - gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev --quiet
  script:
    - docker build -t ${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA} -t ${IMAGE_NAME}:latest .
    - docker push ${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA}
    - docker push ${IMAGE_NAME}:latest
  after_script:
    - rm -f gcp-key.json

deploy:
  stage: deploy
  image: google/cloud-sdk:alpine
  only:
    - main
  before_script:
    - echo $GCP_SERVICE_KEY | base64 -d > gcp-key.json
    - gcloud auth activate-service-account --key-file gcp-key.json
    - gcloud config set project $GCP_PROJECT_ID
  script:
    - |
      gcloud run deploy budget-app \
        --image ${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA} \
        --platform managed \
        --region ${GCP_REGION} \
        --allow-unauthenticated \
        --set-secrets "DATABASE_URL=db-url:latest,NEXTAUTH_SECRET=nextauth-secret:latest" \
        --min-instances 0 \
        --max-instances 3 \
        --memory 512Mi \
        --cpu 1 \
        --timeout 300 \
        --concurrency 80 \
        --cpu-throttling \
        --port 8080
  after_script:
    - rm -f gcp-key.json
```

### Step 7: GitLab Variables

Add these in GitLab: **Settings → CI/CD → Variables**

- `GCP_SERVICE_KEY`: (base64 key from above) - Protected + Masked
- `GCP_PROJECT_ID`: `budget-app-prod`
- `GCP_REGION`: `us-central1`

### Step 8: Deploy!

```bash
# Add files to your repo
cd /path/to/budget-simple-app

# Create Dockerfile (if not exists)
cat > Dockerfile <<'EOF'
FROM node:18-alpine AS base

FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
ENV NEXT_TELEMETRY_DISABLED 1
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder /app/prisma ./prisma
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
USER nextjs
EXPOSE 8080
ENV PORT 8080
ENV HOSTNAME "0.0.0.0"
CMD ["node", "server.js"]
EOF

# Update next.config.js
cat > next.config.js <<'EOF'
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
  experimental: {
    serverComponentsExternalPackages: ['@prisma/client', 'prisma']
  },
  compress: true
}

module.exports = nextConfig
EOF

# Commit and push
git add .gitlab-ci.yml Dockerfile next.config.js
git commit -m "Add CI/CD pipeline"
git push origin main
```

---

## 📊 Monitor Costs

```bash
# Set up budget alert
gcloud billing budgets create \
  --billing-account=BILLING_ACCOUNT_ID \
  --display-name="Budget App Alert" \
  --budget-amount=20 \
  --threshold-rule=percent=50 \
  --threshold-rule=percent=90 \
  --threshold-rule=percent=100

# Check current costs
gcloud billing accounts list
```

---

## 🔧 Further Cost Reduction

### Option 1: Use Cloud Run Free Tier

Cloud Run free tier includes:
- 2 million requests/month
- 360,000 GB-seconds/month
- 180,000 vCPU-seconds/month
- 1 GB network egress/month

With proper configuration, you might pay $0 for Cloud Run!

### Option 2: Reduce Database Backups

```bash
# Keep only 1 backup (minimum)
gcloud sql instances patch budget-db \
  --retained-backups-count=1
```

### Option 3: Schedule Database Downtime

For truly minimal apps, stop the database when not in use:

```bash
# Stop database
gcloud sql instances patch budget-db --activation-policy=NEVER

# Start when needed
gcloud sql instances patch budget-db --activation-policy=ALWAYS
```

**Cost:** $0 when stopped (you only pay for storage)

### Option 4: Use Cloud Functions for Infrequent Access

If your app has very few users, consider Cloud Functions instead of Cloud Run:

```bash
# Deploy as Cloud Function (not covered in this guide)
# Even lower cost for sporadic traffic
```

---

## ⚠️ Limitations of This Setup

1. **Security:** Database has public IP (mitigate with Cloud SQL Auth Proxy)
2. **Performance:** Smallest instance, can be slow under load
3. **Scalability:** Max 3 instances (increase if needed)
4. **Backups:** Only 1 backup (consider increasing for production)
5. **No VPC:** Direct internet connection (less secure)

---

## 🎯 Upgrade Path

When you have more users/budget:

```bash
# Upgrade database
gcloud sql instances patch budget-db \
  --tier=db-g1-small

# Increase Cloud Run instances
gcloud run services update budget-app \
  --max-instances=10 \
  --region=$REGION

# Add VPC connector for security
# (See main guide)
```

---

## 🎉 Result

You now have:

- ✅ Fully automated CI/CD from GitLab
- ✅ Auto-scaling Cloud Run deployment
- ✅ PostgreSQL database with backups
- ✅ Secure secret management
- ✅ **Total cost: $7-15/month**

Perfect for side projects, MVPs, and low-traffic applications!
