# GitLab CI/CD + GCP Troubleshooting Guide

## 🔍 Quick Diagnostics

### Check Pipeline Status

```bash
# View recent pipeline status
# Go to: https://gitlab.com/fatsql-group/budget-simple-app/-/pipelines

# Or use GitLab CLI (if installed)
glab ci status
```

### Check Cloud Run Service

```bash
export PROJECT_ID="budget-app-prod"
export REGION="us-central1"

# Get service status
gcloud run services describe budget-app \
  --region=$REGION \
  --format='value(status.conditions[0].message)'

# Get service URL
gcloud run services describe budget-app \
  --region=$REGION \
  --format='value(status.url)'
```

### Check Recent Logs

```bash
# Cloud Run logs (last 50 lines)
gcloud run services logs tail budget-app \
  --region=$REGION \
  --limit=50

# Filter by error
gcloud logging read "resource.type=cloud_run_revision AND severity>=ERROR" \
  --limit=20 \
  --format=json
```

---

## ❌ Common Issues & Solutions

### 1. Pipeline Fails: "ERROR: Error fetching remote repo"

**Symptom:**
```
ERROR: Error fetching remote repo origin
fatal: unable to access 'https://gitlab.com/...': Could not resolve host
```

**Cause:** Network connectivity issue in GitLab Runner

**Solution:**
```yaml
# Add retry in .gitlab-ci.yml
build:
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
```

---

### 2. Build Fails: "permission denied while trying to connect to Docker daemon"

**Symptom:**
```
ERROR: permission denied while trying to connect to the Docker daemon socket
```

**Cause:** Docker-in-Docker service not properly configured

**Solution:**
```yaml
# Ensure in .gitlab-ci.yml:
build:
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
```

---

### 3. Deploy Fails: "ERROR: (gcloud.run.deploy) INVALID_ARGUMENT: could not resolve source"

**Symptom:**
```
ERROR: (gcloud.run.deploy) INVALID_ARGUMENT: 
The Container Registry image 'gcr.io/...' does not exist
```

**Cause:** Image not pushed to registry or wrong image name

**Solution:**
```bash
# Verify image exists in Artifact Registry
gcloud artifacts docker images list \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/budget-app

# Check image name in deploy command matches build
echo "IMAGE_NAME=${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/budget-app/app"
```

---

### 4. Deploy Fails: "ERROR: (gcloud.run.deploy) User does not have permission"

**Symptom:**
```
ERROR: (gcloud.run.deploy) PERMISSION_DENIED: 
User does not have permission to access service budget-app
```

**Cause:** Service account missing required roles

**Solution:**
```bash
export PROJECT_ID="budget-app-prod"
export SA_EMAIL="gitlab-cicd@${PROJECT_ID}.iam.gserviceaccount.com"

# Add missing roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/iam.serviceAccountUser"

# Verify
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:${SA_EMAIL}"
```

---

### 5. Cloud Run Error: "Container failed to start"

**Symptom:**
```
Cloud Run error: Container failed to start. 
Failed to start and then listen on the port defined by the PORT environment variable
```

**Cause:** App not listening on PORT 8080 or crash on startup

**Solutions:**

#### Check Dockerfile PORT

```dockerfile
# Ensure these lines in Dockerfile:
EXPOSE 8080
ENV PORT 8080
CMD ["node", "server.js"]
```

#### Verify Next.js Config

```javascript
// next.config.js
const nextConfig = {
  output: 'standalone',  // REQUIRED for Docker
  // ...
}
```

#### Check Logs

```bash
# Get detailed startup logs
gcloud run services logs tail budget-app \
  --region=$REGION \
  --limit=100

# Look for:
# - "listening on port 8080" or similar
# - Error messages
# - Stack traces
```

---

### 6. Database Connection Error

**Symptom:**
```
Error: P1001: Can't reach database server at `localhost:5432`
```

**Cause:** DATABASE_URL incorrect or Cloud SQL connection issue

**Solutions:**

#### Check Secret Value

```bash
# View current DATABASE_URL secret
gcloud secrets versions access latest --secret=db-url

# Should look like:
# postgresql://postgres:PASSWORD@PUBLIC_IP:5432/budgetapp
# OR (with Cloud SQL Auth Proxy):
# postgresql://postgres:PASSWORD@localhost:5432/budgetapp?host=/cloudsql/PROJECT:REGION:INSTANCE
```

#### Verify Cloud SQL Instance

```bash
# Check instance status
gcloud sql instances describe budget-db

# Should show: state: RUNNABLE

# Check database exists
gcloud sql databases list --instance=budget-db
```

#### Test Connection Locally

```bash
# Install Cloud SQL Proxy
wget https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.2/cloud-sql-proxy.linux.amd64 -O cloud-sql-proxy
chmod +x cloud-sql-proxy

# Get connection name
export CONNECTION_NAME=$(gcloud sql instances describe budget-db --format='value(connectionName)')

# Start proxy
./cloud-sql-proxy $CONNECTION_NAME &

# Test with psql
export DB_PASSWORD=$(gcloud secrets versions access latest --secret=db-password)
psql "postgresql://postgres:${DB_PASSWORD}@127.0.0.1:5432/budgetapp"

# Should connect successfully
```

---

### 7. Secrets Not Accessible

**Symptom:**
```
ERROR: (gcloud.secretmanager.secrets.versions.access) PERMISSION_DENIED
```

**Cause:** Service account missing Secret Manager access

**Solution:**
```bash
export PROJECT_ID="budget-app-prod"
export SA_EMAIL="gitlab-cicd@${PROJECT_ID}.iam.gserviceaccount.com"

# Grant access to all secrets
for secret in db-password nextauth-secret db-url; do
  gcloud secrets add-iam-policy-binding $secret \
    --member="serviceAccount:${SA_EMAIL}" \
    --role="roles/secretmanager.secretAccessor"
done

# Also grant to Cloud Run default service account
export COMPUTE_SA="${PROJECT_ID_NUMBER}-compute@developer.gserviceaccount.com"
for secret in db-password nextauth-secret db-url; do
  gcloud secrets add-iam-policy-binding $secret \
    --member="serviceAccount:${COMPUTE_SA}" \
    --role="roles/secretmanager.secretAccessor"
done
```

---

### 8. Cloud Build Timeout

**Symptom:**
```
ERROR: build step 0 exceeded timeout of 10m0s
```

**Cause:** Build taking too long

**Solution:**
```yaml
# Increase timeout in .gitlab-ci.yml
build:
  script:
    - docker build --progress=plain -t ${IMAGE_NAME} .
  timeout: 20 minutes  # Increase from default
```

---

### 9. Out of Memory During Build

**Symptom:**
```
npm ERR! errno 137
npm ERR! Exit status 137
```

**Cause:** Build running out of memory (137 = killed by OOM)

**Solutions:**

#### Option 1: Optimize Dockerfile

```dockerfile
# Use multi-stage build (already in our Dockerfile)
# Reduce layer size

FROM node:18-alpine AS deps
# Install only production dependencies
RUN npm ci --only=production
```

#### Option 2: Increase Cloud Run Memory

```bash
# Increase memory limit
gcloud run services update budget-app \
  --region=$REGION \
  --memory=1Gi
```

#### Option 3: Use Cloud Build Instead

```yaml
# In .gitlab-ci.yml, use Cloud Build service
build:
  script:
    - gcloud builds submit --tag ${IMAGE_NAME} .
```

---

### 10. Migration Fails in Pipeline

**Symptom:**
```
Error: P1001: Can't reach database server
prisma migrate deploy failed
```

**Cause:** Can't connect to database from GitLab runner

**Solution:**

#### Option 1: Run Migrations in Cloud Run Job

Instead of in pipeline, create a job:

```bash
# Create migration job
gcloud run jobs create budget-migrate \
  --image ${IMAGE_NAME}:latest \
  --region $REGION \
  --set-secrets "DATABASE_URL=db-url:latest" \
  --add-cloudsql-instances $CONNECTION_NAME \
  --command "npx" \
  --args "prisma,migrate,deploy"

# Run manually after deployment
gcloud run jobs execute budget-migrate --region $REGION
```

#### Option 2: Remove Migration from Pipeline

```yaml
# Comment out migrate stage in .gitlab-ci.yml
# migrate:
#   stage: post-deploy
#   ...
```

---

### 11. VPC Connector Error

**Symptom:**
```
ERROR: (gcloud.run.deploy) INVALID_ARGUMENT: 
The provided VPC connector does not exist
```

**Cause:** VPC connector not created or wrong region

**Solution:**
```bash
# Check VPC connector
gcloud compute networks vpc-access connectors list

# Create if missing
gcloud compute networks vpc-access connectors create budget-connector \
  --network=default \
  --region=$REGION \
  --range=10.8.0.0/28

# Wait for READY state (2-3 minutes)
gcloud compute networks vpc-access connectors describe budget-connector \
  --region=$REGION
```

---

### 12. Image Too Large

**Symptom:**
```
ERROR: Container image size exceeds limit
```

**Cause:** Docker image too big

**Solutions:**

#### Check Image Size

```bash
docker images ${IMAGE_NAME}:latest
# Should be under 1GB ideally
```

#### Optimize Dockerfile

```dockerfile
# Use alpine images (smaller)
FROM node:18-alpine

# Remove unnecessary files in .dockerignore
node_modules
.git
*.log
.next
```

#### Multi-stage Build

```dockerfile
# Already using multi-stage in our Dockerfile
# Copies only necessary files to final stage
```

---

### 13. Rate Limit Errors

**Symptom:**
```
ERROR: RESOURCE_EXHAUSTED: Quota exceeded
```

**Cause:** Too many API calls

**Solution:**
```bash
# Check quotas
gcloud compute project-info describe --project=$PROJECT_ID

# Request quota increase if needed:
# https://console.cloud.google.com/iam-admin/quotas
```

---

### 14. Health Check Fails

**Symptom:**
```
❌ Health check failed! Status: 502
```

**Cause:** App not responding or crashed

**Solutions:**

#### Add Health Endpoint

```javascript
// pages/api/health.js or app/api/health/route.ts
export async function GET() {
  return Response.json({ status: 'ok', timestamp: new Date().toISOString() })
}
```

#### Check Cloud Run Logs

```bash
gcloud run services logs tail budget-app --region=$REGION --limit=50

# Look for startup errors
```

#### Increase Timeout

```yaml
# In .gitlab-ci.yml health_check job
script:
  - sleep 30  # Increase wait time
  - curl -f $SERVICE_URL/ || exit 1
```

---

## 🔧 Debugging Commands

### View All Resources

```bash
export PROJECT_ID="budget-app-prod"
export REGION="us-central1"

# Cloud Run services
gcloud run services list --region=$REGION

# Cloud SQL instances
gcloud sql instances list

# Artifact Registry repositories
gcloud artifacts repositories list

# Secrets
gcloud secrets list

# VPC connectors
gcloud compute networks vpc-access connectors list --region=$REGION

# IAM service accounts
gcloud iam service-accounts list
```

### Force Redeploy

```bash
# Redeploy with same image
gcloud run services update budget-app \
  --region=$REGION \
  --update-env-vars="LAST_DEPLOY=$(date +%s)"
```

### Delete and Recreate Service

```bash
# Delete service
gcloud run services delete budget-app --region=$REGION --quiet

# Redeploy (run GitLab pipeline or manual deploy)
```

### View Build History

```bash
# List recent builds
gcloud builds list --limit=10

# View specific build log
gcloud builds log BUILD_ID
```

---

## 📊 Monitoring & Logging

### Real-time Log Streaming

```bash
# Stream logs
gcloud run services logs tail budget-app \
  --region=$REGION \
  --follow

# Filter by severity
gcloud logging read "resource.type=cloud_run_revision AND severity>=ERROR" \
  --limit=50 \
  --format=json \
  --freshness=1h
```

### Check Service Metrics

```bash
# Get request count
gcloud monitoring time-series list \
  --filter='metric.type="run.googleapis.com/request_count"' \
  --interval-start-time="$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --interval-end-time="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

---

## 🆘 Emergency Rollback

### Rollback to Previous Revision

```bash
# List revisions
gcloud run revisions list \
  --service=budget-app \
  --region=$REGION

# Rollback to specific revision
gcloud run services update-traffic budget-app \
  --region=$REGION \
  --to-revisions=budget-app-00005-xyz=100

# Or rollback to previous
gcloud run services update-traffic budget-app \
  --region=$REGION \
  --to-revisions=LATEST-1=100
```

---

## 📝 Get Help

### Official Documentation

- [Cloud Run Docs](https://cloud.google.com/run/docs)
- [GitLab CI/CD Docs](https://docs.gitlab.com/ee/ci/)
- [Cloud SQL Docs](https://cloud.google.com/sql/docs)

### Community Support

- [Stack Overflow - google-cloud-run](https://stackoverflow.com/questions/tagged/google-cloud-run)
- [GitLab Forum](https://forum.gitlab.com/)
- [GCP Community](https://www.googlecloudcommunity.com/)

### Check GCP Status

- [GCP Status Dashboard](https://status.cloud.google.com/)

---

## ✅ Prevention Checklist

- [ ] Use version tags for images (not just `latest`)
- [ ] Set up monitoring and alerts
- [ ] Test deployments in staging first
- [ ] Keep service account keys secure
- [ ] Regularly review IAM permissions
- [ ] Monitor costs daily
- [ ] Keep backup of critical secrets
- [ ] Document all configuration changes
- [ ] Use semantic versioning for releases
- [ ] Test locally before pushing

---

## 🎯 Still Having Issues?

1. **Check GitLab Pipeline Logs**: Go to your pipeline and click on failed job
2. **Check Cloud Run Logs**: `gcloud run services logs tail budget-app --region=$REGION`
3. **Verify Environment Variables**: Make sure all GitLab CI/CD variables are set
4. **Test Docker Build Locally**: `docker build -t test .` and `docker run -p 8080:8080 test`
5. **Contact Support**: If all else fails, contact GCP or GitLab support

Good luck debugging! 🚀
