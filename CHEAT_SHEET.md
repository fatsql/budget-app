# GitLab + GCP Deployment Cheat Sheet

## 🚀 Quick Commands

### Initial Setup (One-Time)

```bash
# Set variables
export PROJECT_ID="budget-app-prod"
export REGION="us-central1"
export SA_EMAIL="gitlab-cicd@${PROJECT_ID}.iam.gserviceaccount.com"

# Create project & enable APIs
gcloud projects create $PROJECT_ID
gcloud config set project $PROJECT_ID
gcloud services enable run.googleapis.com sqladmin.googleapis.com cloudbuild.googleapis.com secretmanager.googleapis.com artifactregistry.googleapis.com

# Create database
export DB_PASSWORD="$(openssl rand -base64 32)"
gcloud sql instances create budget-db --database-version=POSTGRES_15 --tier=db-f1-micro --region=$REGION --assign-ip --root-password="$DB_PASSWORD"
gcloud sql databases create budgetapp --instance=budget-db

# Store secrets
echo -n "$DB_PASSWORD" | gcloud secrets create db-password --data-file=-
echo -n "$(openssl rand -base64 32)" | gcloud secrets create nextauth-secret --data-file=-

# Create Artifact Registry
gcloud artifacts repositories create budget-app --repository-format=docker --location=$REGION

# Create service account
gcloud iam service-accounts create gitlab-cicd
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${SA_EMAIL}" --role="roles/run.admin"
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${SA_EMAIL}" --role="roles/iam.serviceAccountUser"
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${SA_EMAIL}" --role="roles/artifactregistry.writer"
gcloud iam service-accounts keys create gitlab-key.json --iam-account=$SA_EMAIL

# Get base64 key for GitLab
cat gitlab-key.json | base64 -w 0
```

---

## 📝 Daily Operations

### Deploy Manually

```bash
# Build and push image
export IMAGE_NAME="${REGION}-docker.pkg.dev/${PROJECT_ID}/budget-app/app"
docker build -t ${IMAGE_NAME}:$(git rev-parse --short HEAD) .
docker push ${IMAGE_NAME}:$(git rev-parse --short HEAD)

# Deploy to Cloud Run
gcloud run deploy budget-app \
  --image ${IMAGE_NAME}:$(git rev-parse --short HEAD) \
  --platform managed \
  --region $REGION \
  --allow-unauthenticated \
  --set-secrets "DATABASE_URL=db-url:latest,NEXTAUTH_SECRET=nextauth-secret:latest" \
  --min-instances 0 \
  --max-instances 5 \
  --memory 512Mi \
  --cpu 1
```

### View Logs

```bash
# Real-time logs
gcloud run services logs tail budget-app --region=$REGION --follow

# Last 50 lines
gcloud run services logs tail budget-app --region=$REGION --limit=50

# Errors only
gcloud logging read "resource.type=cloud_run_revision AND severity>=ERROR" --limit=20
```

### Check Status

```bash
# Service URL
gcloud run services describe budget-app --region=$REGION --format='value(status.url)'

# Service status
gcloud run services describe budget-app --region=$REGION --format='value(status.conditions[0].message)'

# All services
gcloud run services list --region=$REGION
```

### Database Operations

```bash
# Connect to database
gcloud sql connect budget-db --user=postgres --database=budgetapp

# Run migrations
export DB_PASSWORD=$(gcloud secrets versions access latest --secret=db-password)
export DATABASE_URL="postgresql://postgres:${DB_PASSWORD}@PUBLIC_IP:5432/budgetapp"
npx prisma migrate deploy

# Backup database
gcloud sql backups create --instance=budget-db

# List backups
gcloud sql backups list --instance=budget-db
```

---

## 🔧 Troubleshooting

### Common Fixes

```bash
# Restart service
gcloud run services update budget-app --region=$REGION --update-env-vars="RESTART=$(date +%s)"

# Rollback to previous revision
gcloud run services update-traffic budget-app --region=$REGION --to-revisions=LATEST-1=100

# Delete and redeploy
gcloud run services delete budget-app --region=$REGION --quiet
# Then run deployment again

# Check IAM permissions
gcloud projects get-iam-policy $PROJECT_ID --flatten="bindings[].members" --filter="bindings.members:serviceAccount:${SA_EMAIL}"
```

### View Resources

```bash
# All Cloud Run services
gcloud run services list --region=$REGION

# All Cloud SQL instances
gcloud sql instances list

# All secrets
gcloud secrets list

# All images
gcloud artifacts docker images list ${REGION}-docker.pkg.dev/${PROJECT_ID}/budget-app

# Recent builds
gcloud builds list --limit=5
```

---

## 💰 Cost Management

### Check Costs

```bash
# View billing account
gcloud billing accounts list

# View project costs (via console)
# https://console.cloud.google.com/billing/
```

### Cost Optimization

```bash
# Scale to zero when idle
gcloud run services update budget-app --region=$REGION --min-instances=0

# Reduce max instances
gcloud run services update budget-app --region=$REGION --max-instances=3

# Reduce memory
gcloud run services update budget-app --region=$REGION --memory=256Mi

# Stop database (when not needed)
gcloud sql instances patch budget-db --activation-policy=NEVER

# Start database
gcloud sql instances patch budget-db --activation-policy=ALWAYS
```

---

## 🔒 Security

### Update Secrets

```bash
# Update database password
NEW_PASSWORD="$(openssl rand -base64 32)"
gcloud sql users set-password postgres --instance=budget-db --password="$NEW_PASSWORD"
echo -n "$NEW_PASSWORD" | gcloud secrets versions add db-password --data-file=-

# Rotate service account key
gcloud iam service-accounts keys list --iam-account=$SA_EMAIL
gcloud iam service-accounts keys delete OLD_KEY_ID --iam-account=$SA_EMAIL
gcloud iam service-accounts keys create new-key.json --iam-account=$SA_EMAIL
cat new-key.json | base64 -w 0  # Update in GitLab
```

### Grant/Revoke Access

```bash
# Grant Cloud Run access to user
gcloud run services add-iam-policy-binding budget-app \
  --region=$REGION \
  --member="user:email@example.com" \
  --role="roles/run.invoker"

# Revoke access
gcloud run services remove-iam-policy-binding budget-app \
  --region=$REGION \
  --member="user:email@example.com" \
  --role="roles/run.invoker"

# Make service require authentication
gcloud run services update budget-app --region=$REGION --no-allow-unauthenticated
```

---

## 📊 Monitoring

### Create Alerts

```bash
# Create email notification channel
gcloud alpha monitoring channels create \
  --display-name="Email" \
  --type=email \
  --channel-labels=email_address=your@email.com

# Get channel ID
export CHANNEL_ID=$(gcloud alpha monitoring channels list --filter="displayName:'Email'" --format="value(name)")

# Create uptime check
gcloud monitoring uptime-check-configs create budget-app-uptime \
  --display-name="Budget App" \
  --http-check-path="/" \
  --monitored-resource-type="uptime_url" \
  --monitored-resource="host=$(gcloud run services describe budget-app --region=$REGION --format='value(status.url)' | sed 's|https://||')"
```

---

## 🌐 Domain Setup

### Map Custom Domain

```bash
# Map domain
gcloud run domain-mappings create \
  --service budget-app \
  --domain app.yourdomain.com \
  --region $REGION

# Get DNS records to add
gcloud run domain-mappings describe app.yourdomain.com --region=$REGION

# Check status
gcloud run domain-mappings list --region=$REGION
```

---

## 🔄 CI/CD Management

### GitLab Variables

Set these in GitLab: **Settings → CI/CD → Variables**

- `GCP_SERVICE_KEY` (base64 service account key)
- `GCP_PROJECT_ID`
- `GCP_REGION`

### Trigger Manual Pipeline

```bash
# Via GitLab UI
# Go to: CI/CD → Pipelines → Run pipeline

# Via API (requires GitLab token)
curl -X POST \
  -F token=YOUR_TRIGGER_TOKEN \
  -F ref=main \
  https://gitlab.com/api/v4/projects/PROJECT_ID/trigger/pipeline
```

---

## 📦 Image Management

### List Images

```bash
# List all images
gcloud artifacts docker images list ${REGION}-docker.pkg.dev/${PROJECT_ID}/budget-app

# List tags for specific image
gcloud artifacts docker tags list ${REGION}-docker.pkg.dev/${PROJECT_ID}/budget-app/app
```

### Delete Old Images

```bash
# Delete specific image
gcloud artifacts docker images delete \
  ${REGION}-docker.pkg.dev/${PROJECT_ID}/budget-app/app:old-tag \
  --delete-tags

# Delete all untagged images (free up space)
gcloud artifacts docker images list ${REGION}-docker.pkg.dev/${PROJECT_ID}/budget-app \
  --include-tags \
  --filter="-tags:*" \
  --format="get(image)" | \
  xargs -I {} gcloud artifacts docker images delete {} --quiet
```

---

## 🧪 Testing

### Test Locally

```bash
# Build image
docker build -t test-app .

# Run locally
docker run -p 8080:8080 \
  -e DATABASE_URL="your-db-url" \
  -e NEXTAUTH_SECRET="test-secret" \
  test-app

# Test endpoint
curl http://localhost:8080
```

### Load Testing

```bash
# Install hey (HTTP load generator)
go install github.com/rakyll/hey@latest

# Run load test
export SERVICE_URL=$(gcloud run services describe budget-app --region=$REGION --format='value(status.url)')
hey -n 1000 -c 10 $SERVICE_URL
```

---

## 📚 Useful Links

### Documentation
- [Cloud Run](https://cloud.google.com/run/docs)
- [Cloud SQL](https://cloud.google.com/sql/docs)
- [GitLab CI/CD](https://docs.gitlab.com/ee/ci/)
- [Next.js Deployment](https://nextjs.org/docs/deployment)

### Consoles
- [GCP Console](https://console.cloud.google.com/)
- [GitLab Pipelines](https://gitlab.com/fatsql-group/budget-simple-app/-/pipelines)
- [Cloud Run Services](https://console.cloud.google.com/run)
- [Cloud SQL Instances](https://console.cloud.google.com/sql/instances)

---

## 💡 Pro Tips

1. **Use tags for images**: Deploy with `git SHA` tags instead of `latest`
2. **Set budget alerts**: Avoid surprise bills
3. **Enable request logging**: Helps with debugging
4. **Use staging environment**: Test before production
5. **Automate backups**: Schedule regular database backups
6. **Monitor cold starts**: Optimize if startup time is slow
7. **Use Cloud Build caching**: Speeds up builds
8. **Keep secrets in Secret Manager**: Never in code
9. **Use Cloud Armor**: Protect against DDoS
10. **Review IAM regularly**: Remove unnecessary permissions

---

## 🆘 Emergency Contacts

- **GCP Support**: https://cloud.google.com/support
- **GitLab Support**: https://about.gitlab.com/support/
- **Status Pages**:
  - GCP: https://status.cloud.google.com/
  - GitLab: https://status.gitlab.com/

---

## 📋 Pre-Flight Checklist

Before pushing to production:

- [ ] All tests passing locally
- [ ] Database migrations tested
- [ ] Environment variables set
- [ ] Secrets properly configured
- [ ] Monitoring and alerts set up
- [ ] Backup strategy in place
- [ ] Cost budget configured
- [ ] Security review completed
- [ ] Documentation updated
- [ ] Staging deployment successful

---

**Quick Access Commands:**

```bash
# Save these aliases for faster access
alias gr-logs='gcloud run services logs tail budget-app --region=us-central1'
alias gr-status='gcloud run services describe budget-app --region=us-central1 --format="value(status.url)"'
alias gr-deploy='gcloud run deploy budget-app --region=us-central1'
alias sql-connect='gcloud sql connect budget-db --user=postgres'
```

Save this cheat sheet and refer back anytime! 🚀
