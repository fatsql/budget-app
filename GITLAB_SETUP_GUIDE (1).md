# GitLab CI/CD Setup Guide for Budget App - GCP Deployment

## 🎯 Overview

This guide will help you set up automated CI/CD deployment from GitLab to Google Cloud Platform (GCP) with **cost-optimized** settings.

**Estimated Monthly Cost:** $10-25 for low traffic (under 10,000 users)

---

## 📋 Prerequisites

- GitLab account with your repository at https://gitlab.com/fatsql-group/budget-simple-app
- Google Cloud Platform account with billing enabled
- Basic understanding of Docker and CI/CD concepts

---

## 🚀 Step-by-Step Setup

### Phase 1: GCP Project Setup (15 minutes)

#### 1.1 Create GCP Project

```bash
# Set your project ID (must be globally unique)
export PROJECT_ID="budget-app-prod"
export REGION="us-central1"

# Create project
gcloud projects create $PROJECT_ID --name="Budget App"

# Set as default project
gcloud config set project $PROJECT_ID

# Enable billing (do this via console: https://console.cloud.google.com/billing)
```

#### 1.2 Enable Required APIs

```bash
# Enable all necessary APIs
gcloud services enable \
  run.googleapis.com \
  sql-component.googleapis.com \
  sqladmin.googleapis.com \
  cloudbuild.googleapis.com \
  secretmanager.googleapis.com \
  compute.googleapis.com \
  vpcaccess.googleapis.com \
  artifactregistry.googleapis.com
```

**⏱️ Time:** ~2-3 minutes

#### 1.3 Create Artifact Registry Repository

```bash
# Create Docker repository (cheaper than Container Registry)
gcloud artifacts repositories create budget-app \
  --repository-format=docker \
  --location=$REGION \
  --description="Budget App Docker Images"

# Verify
gcloud artifacts repositories list
```

---

### Phase 2: Database Setup (25 minutes)

#### 2.1 Create Cloud SQL Instance (Cost-Optimized)

```bash
# Generate secure password
export DB_PASSWORD="$(openssl rand -base64 32)"
export INSTANCE_NAME="budget-db"
export DB_NAME="budgetapp"

# Save password securely
echo "DB_PASSWORD=$DB_PASSWORD" >> ~/budget-secrets.txt
chmod 600 ~/budget-secrets.txt

# Create smallest possible instance
gcloud sql instances create $INSTANCE_NAME \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=$REGION \
  --network=default \
  --no-assign-ip \
  --root-password="$DB_PASSWORD" \
  --backup-start-time=03:00 \
  --maintenance-window-day=SUN \
  --maintenance-window-hour=3 \
  --enable-bin-log=false \
  --retained-backups-count=7
```

**💰 Cost:** ~$7/month for db-f1-micro

#### 2.2 Create Database

```bash
# Create application database
gcloud sql databases create $DB_NAME \
  --instance=$INSTANCE_NAME
```

#### 2.3 Store Secrets in Secret Manager

```bash
# Store database password
echo -n "$DB_PASSWORD" | gcloud secrets create db-password \
  --data-file=- \
  --replication-policy="automatic"

# Generate NextAuth secret
NEXTAUTH_SECRET="$(openssl rand -base64 32)"
echo -n "$NEXTAUTH_SECRET" | gcloud secrets create nextauth-secret \
  --data-file=- \
  --replication-policy="automatic"

# Create database URL secret
DB_URL="postgresql://postgres:${DB_PASSWORD}@localhost:5432/${DB_NAME}?host=/cloudsql/${PROJECT_ID}:${REGION}:${INSTANCE_NAME}"
echo -n "$DB_URL" | gcloud secrets create db-url \
  --data-file=- \
  --replication-policy="automatic"

# Verify secrets
gcloud secrets list
```

---

### Phase 3: Networking Setup (10 minutes)

#### 3.1 Create VPC Connector

```bash
# Create VPC connector for Cloud Run to access Cloud SQL
gcloud compute networks vpc-access connectors create budget-connector \
  --network=default \
  --region=$REGION \
  --range=10.8.0.0/28

# Verify (wait for READY state)
gcloud compute networks vpc-access connectors describe budget-connector \
  --region=$REGION
```

**⏱️ Time:** 2-3 minutes to provision

---

### Phase 4: Service Account Setup (10 minutes)

#### 4.1 Create Service Account for GitLab CI/CD

```bash
# Create service account
gcloud iam service-accounts create gitlab-cicd \
  --display-name="GitLab CI/CD" \
  --description="Service account for GitLab CI/CD pipeline"

export SA_EMAIL="gitlab-cicd@${PROJECT_ID}.iam.gserviceaccount.com"

# Grant necessary permissions
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/cloudsql.client"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

# Allow service account to access secrets
for secret in db-password nextauth-secret db-url; do
  gcloud secrets add-iam-policy-binding $secret \
    --member="serviceAccount:${SA_EMAIL}" \
    --role="roles/secretmanager.secretAccessor"
done
```

#### 4.2 Create and Download Service Account Key

```bash
# Create key
gcloud iam service-accounts keys create ~/gitlab-cicd-key.json \
  --iam-account=$SA_EMAIL

# Base64 encode for GitLab (copy this output)
cat ~/gitlab-cicd-key.json | base64 -w 0 > ~/gitlab-cicd-key-base64.txt

echo "================================================"
echo "⚠️  IMPORTANT: Copy the base64 key below to GitLab"
echo "================================================"
cat ~/gitlab-cicd-key-base64.txt
echo ""
echo "================================================"

# Secure the files
chmod 600 ~/gitlab-cicd-key.json ~/gitlab-cicd-key-base64.txt
```

---

### Phase 5: GitLab Configuration (10 minutes)

#### 5.1 Add GitLab CI/CD Variables

1. Go to your GitLab repository: https://gitlab.com/fatsql-group/budget-simple-app
2. Navigate to: **Settings → CI/CD → Variables**
3. Add these variables (click "Expand" then "Add variable"):

| Key | Value | Protected | Masked |
|-----|-------|-----------|--------|
| `GCP_SERVICE_KEY` | (paste base64 key from above) | ✅ | ✅ |
| `GCP_PROJECT_ID` | `budget-app-prod` | ✅ | ❌ |
| `GCP_REGION` | `us-central1` | ✅ | ❌ |

**🔒 Security Notes:**
- Mark `GCP_SERVICE_KEY` as **Protected** and **Masked**
- Protected variables only work on protected branches (main)
- Never commit service account keys to your repository

#### 5.2 Add .gitlab-ci.yml to Repository

```bash
# Navigate to your local repo
cd /path/to/your/budget-simple-app

# Copy the .gitlab-ci.yml file
cp /home/claude/.gitlab-ci.yml .gitlab-ci.yml

# Commit and push
git add .gitlab-ci.yml
git commit -m "Add GitLab CI/CD pipeline for GCP deployment"
git push origin main
```

---

### Phase 6: Required Project Files (15 minutes)

#### 6.1 Create Dockerfile (if not exists)

Create `Dockerfile` in your project root:

```dockerfile
# Multi-stage build for optimal size
FROM node:18-alpine AS base

# Dependencies stage
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

COPY package.json package-lock.json* ./
RUN npm ci

# Builder stage
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Generate Prisma Client
RUN npx prisma generate

ENV NEXT_TELEMETRY_DISABLED 1
RUN npm run build

# Runner stage (production)
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
```

#### 6.2 Create .dockerignore

```
node_modules
npm-debug.log*
.next
.git
.gitignore
.env*
README.md
.vscode
.idea
*.swp
.DS_Store
Thumbs.db
coverage
.nyc_output
dist
build
*.log
```

#### 6.3 Update next.config.js

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
  
  experimental: {
    serverComponentsExternalPackages: ['@prisma/client', 'prisma']
  },
  
  compress: true,
  
  images: {
    domains: ['storage.googleapis.com'],
  },
  
  // Environment variables available to the browser
  env: {
    NEXT_PUBLIC_APP_URL: process.env.NEXTAUTH_URL || 'http://localhost:3000',
  }
}

module.exports = nextConfig
```

---

### Phase 7: Initial Deployment (20 minutes)

#### 7.1 Manual First Deployment (Recommended)

Before running CI/CD, let's do a manual deployment to ensure everything works:

```bash
# Build and push image manually
export IMAGE_NAME="${REGION}-docker.pkg.dev/${PROJECT_ID}/budget-app/app"

# Authenticate Docker
gcloud auth configure-docker ${REGION}-docker.pkg.dev

# Build
docker build -t ${IMAGE_NAME}:manual .

# Push
docker push ${IMAGE_NAME}:manual

# Deploy to Cloud Run
export CONNECTION_NAME="${PROJECT_ID}:${REGION}:${INSTANCE_NAME}"

gcloud run deploy budget-app \
  --image ${IMAGE_NAME}:manual \
  --platform managed \
  --region $REGION \
  --allow-unauthenticated \
  --set-env-vars "NODE_ENV=production" \
  --set-secrets "DATABASE_URL=db-url:latest,NEXTAUTH_SECRET=nextauth-secret:latest" \
  --add-cloudsql-instances $CONNECTION_NAME \
  --vpc-connector budget-connector \
  --vpc-egress private-ranges-only \
  --min-instances 0 \
  --max-instances 5 \
  --memory 512Mi \
  --cpu 1 \
  --timeout 300 \
  --concurrency 80 \
  --cpu-throttling \
  --port 8080

# Get service URL
export SERVICE_URL=$(gcloud run services describe budget-app --region $REGION --format='value(status.url)')
echo "Your app is live at: $SERVICE_URL"
```

#### 7.2 Run Database Migrations

```bash
# Install Cloud SQL Proxy locally
wget https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.2/cloud-sql-proxy.linux.amd64 -O cloud-sql-proxy
chmod +x cloud-sql-proxy

# Start proxy in background
./cloud-sql-proxy $CONNECTION_NAME &

# Set DATABASE_URL
export DATABASE_URL="postgresql://postgres:${DB_PASSWORD}@127.0.0.1:5432/${DB_NAME}"

# Run migrations
npm install
npx prisma migrate deploy

# Kill proxy
pkill cloud-sql-proxy
```

#### 7.3 Test Deployment

```bash
# Health check
curl $SERVICE_URL

# Check logs
gcloud run services logs tail budget-app --region $REGION
```

---

### Phase 8: Trigger CI/CD Pipeline

Now that manual deployment works, let's test the CI/CD pipeline:

```bash
# Make a small change
echo "# CI/CD test" >> README.md

# Commit and push
git add README.md
git commit -m "Test CI/CD pipeline"
git push origin main
```

**Watch the pipeline:**
1. Go to https://gitlab.com/fatsql-group/budget-simple-app/-/pipelines
2. You should see a running pipeline with stages: test → build → deploy → post-deploy

---

## 💰 Cost Optimization Tips

### Current Configuration Costs (Monthly)

| Service | Configuration | Cost |
|---------|--------------|------|
| Cloud SQL | db-f1-micro, 10GB | ~$7 |
| Cloud Run | 512MB, 0-5 instances, scale to zero | $0-10 |
| Artifact Registry | Storage | $0.10/GB |
| VPC Connector | Serverless VPC Access | $0.071/hour ≈ $52/mo |
| Secret Manager | 3 secrets | Free tier |
| **Total** | | **~$60-70/month** |

### Ways to Reduce Costs Further

#### Option 1: Remove VPC Connector (-$52/month)

**Trade-off:** Cloud SQL must have public IP

```bash
# Enable public IP on Cloud SQL
gcloud sql instances patch $INSTANCE_NAME \
  --assign-ip

# Get public IP
gcloud sql instances describe $INSTANCE_NAME \
  --format='value(ipAddresses[0].ipAddress)'

# Update deployment (no VPC connector needed)
gcloud run deploy budget-app \
  --image ${IMAGE_NAME}:latest \
  --add-cloudsql-instances $CONNECTION_NAME \
  --remove-vpc-connector
```

**New total: ~$10-20/month**

#### Option 2: Use Cloud Run Jobs for Migrations

Instead of running migrations in CI/CD, create a Cloud Run Job:

```bash
gcloud run jobs create budget-app-migrate \
  --image ${IMAGE_NAME}:latest \
  --region $REGION \
  --task-timeout 10m \
  --set-env-vars "NODE_ENV=production" \
  --set-secrets "DATABASE_URL=db-url:latest" \
  --add-cloudsql-instances $CONNECTION_NAME

# Run manually when needed
gcloud run jobs execute budget-app-migrate --region $REGION
```

#### Option 3: Use Shared Core SQL Instance

```bash
# Update to shared-core (cheaper but less performant)
gcloud sql instances patch $INSTANCE_NAME \
  --tier=db-g1-small
```

**Cost: ~$9.50/month** (vs $7 for f1-micro)

---

## 🔐 Security Best Practices

### 1. Restrict Service Account Permissions

The service account has broad permissions. Once everything works, narrow them down:

```bash
# Remove admin role
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/run.admin"

# Add specific roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/run.developer"
```

### 2. Enable Cloud Run Authentication (Production)

```bash
# Require authentication
gcloud run services update budget-app \
  --region $REGION \
  --no-allow-unauthenticated

# Add IAM binding for specific users
gcloud run services add-iam-policy-binding budget-app \
  --region $REGION \
  --member="user:admin@yourdomain.com" \
  --role="roles/run.invoker"
```

### 3. Rotate Service Account Keys Regularly

```bash
# List keys
gcloud iam service-accounts keys list \
  --iam-account=$SA_EMAIL

# Delete old keys
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account=$SA_EMAIL
```

### 4. Enable Cloud Armor (DDoS Protection)

```bash
# Create security policy
gcloud compute security-policies create budget-app-policy

# Add rate limiting rule
gcloud compute security-policies rules create 1000 \
  --security-policy budget-app-policy \
  --expression "true" \
  --action "rate-based-ban" \
  --rate-limit-threshold-count 100 \
  --rate-limit-threshold-interval-sec 60
```

---

## 📊 Monitoring Setup

### 1. Create Uptime Check

```bash
gcloud monitoring uptime-check-configs create budget-app-uptime \
  --display-name="Budget App Uptime" \
  --http-check-path="/" \
  --monitored-resource-type="uptime_url" \
  --monitored-resource="host=$(echo $SERVICE_URL | sed 's|https://||')"
```

### 2. Set Up Alerting

```bash
# Create email notification channel
gcloud alpha monitoring channels create \
  --display-name="Email Alerts" \
  --type=email \
  --channel-labels=email_address=your@email.com

# Get channel ID
export CHANNEL_ID=$(gcloud alpha monitoring channels list \
  --filter="displayName:'Email Alerts'" \
  --format="value(name)")

# Create error rate alert
cat > alert-policy.yaml <<EOF
displayName: "High Error Rate"
conditions:
  - displayName: "Error rate > 5%"
    conditionThreshold:
      filter: 'resource.type="cloud_run_revision" AND metric.type="run.googleapis.com/request_count" AND metric.labels.response_code_class!="2xx"'
      comparison: COMPARISON_GT
      thresholdValue: 0.05
      duration: 60s
notificationChannels:
  - ${CHANNEL_ID}
EOF

gcloud alpha monitoring policies create --policy-from-file=alert-policy.yaml
```

---

## 🐛 Troubleshooting

### Pipeline Fails at Build Stage

**Issue:** Docker build fails

**Solution:**
```bash
# Check build logs in GitLab
# Common issues:
# 1. Missing dependencies in package.json
# 2. Dockerfile syntax errors
# 3. Build timeout (increase in gitlab-ci.yml)

# Test build locally
docker build -t test .
```

### Pipeline Fails at Deploy Stage

**Issue:** Authentication error

**Solution:**
```bash
# Verify service account key
cat ~/gitlab-cicd-key.json

# Re-create base64 encoded key
cat ~/gitlab-cicd-key.json | base64 -w 0

# Update GitLab variable GCP_SERVICE_KEY
```

### Cloud Run Can't Connect to Database

**Issue:** VPC connector timeout

**Solution:**
```bash
# Check VPC connector status
gcloud compute networks vpc-access connectors describe budget-connector \
  --region=$REGION

# Verify Cloud SQL instance
gcloud sql instances describe $INSTANCE_NAME

# Check service account permissions
gcloud projects get-iam-policy $PROJECT_ID
```

### Migrations Fail

**Issue:** Prisma can't connect to database

**Solution:**
```bash
# Test connection locally with Cloud SQL Proxy
./cloud-sql-proxy $CONNECTION_NAME &

# Verify DATABASE_URL secret
gcloud secrets versions access latest --secret=db-url

# Check if database exists
gcloud sql databases list --instance=$INSTANCE_NAME
```

---

## 📝 Next Steps

### 1. Custom Domain Setup

```bash
# Map custom domain
gcloud run domain-mappings create \
  --service budget-app \
  --domain app.yourdomain.com \
  --region $REGION

# Add DNS records (from command output)
```

### 2. Enable CDN

```bash
# Create load balancer with CDN
# (Requires additional setup - see GCP docs)
```

### 3. Set Up Staging Environment

Uncomment the `deploy:staging` job in `.gitlab-ci.yml` and create a `develop` branch:

```bash
git checkout -b develop
git push origin develop
```

### 4. Add More Tests

Update the `test` stage in `.gitlab-ci.yml`:

```yaml
test:
  script:
    - npm ci
    - npm run lint
    - npm run test
    - npm run test:e2e
```

---

## 📚 Additional Resources

- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Cloud SQL Documentation](https://cloud.google.com/sql/docs)
- [Next.js Docker Documentation](https://nextjs.org/docs/deployment)
- [Prisma Cloud SQL Guide](https://www.prisma.io/docs/guides/deployment/deployment-guides/deploying-to-google-cloud-platform)

---

## ✅ Checklist

Before going to production:

- [ ] All environment variables set in GitLab
- [ ] Service account key is secure and masked
- [ ] Database migrations run successfully
- [ ] Manual deployment works
- [ ] CI/CD pipeline passes all stages
- [ ] Health checks passing
- [ ] Monitoring and alerts configured
- [ ] Custom domain configured (optional)
- [ ] Backup strategy in place
- [ ] Security review completed
- [ ] Cost alerts configured

---

## 🎉 You're Done!

Your budget app should now automatically deploy to GCP every time you push to the `main` branch!

**Total Setup Time:** ~2-3 hours  
**Monthly Cost:** $10-70 depending on configuration  
**Deployment Time:** ~5-10 minutes per push

Happy deploying! 🚀
