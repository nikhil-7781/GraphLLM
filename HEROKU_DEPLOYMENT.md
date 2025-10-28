# Heroku Deployment Guide for GraphLLM

## 🚀 Quick Deploy (10 Minutes)

Heroku is a mature, production-ready platform with excellent Docker support and GitHub integration.

---

## Why Heroku?

✅ **Features:**
- Mature platform (15+ years)
- Excellent Docker support
- GitHub auto-deployment
- Built-in monitoring & logging
- Easy scaling
- Add-ons marketplace
- CLI tools

✅ **Perfect for ML:**
- Docker-native deployment
- Large slug support (via Docker)
- Long request timeouts
- Process scaling
- Persistent storage via add-ons

⚠️ **Pricing:**
- **No free tier** (ended Nov 2022)
- **Eco Dyno:** $5/month (sleeps after 30 min inactivity)
- **Basic Dyno:** $7/month (never sleeps)
- **Standard Dynos:** $25-50/month (production-grade)

---

## Prerequisites

1. **Heroku Account**
   - Sign up: https://signup.heroku.com
   - Requires credit card (even for Eco plan)

2. **Heroku CLI**
   - Install: https://devcenter.heroku.com/articles/heroku-cli

3. **GitHub Repository**
   - Already have: https://github.com/nikhil-7781/GraphLLM

4. **Gemini API Key**
   - From: https://makersuite.google.com/app/apikey

---

## Step-by-Step Deployment

### **Step 1: Install Heroku CLI**

**macOS:**
```bash
brew tap heroku/brew && brew install heroku
```

**Windows:**
Download from: https://devcenter.heroku.com/articles/heroku-cli

**Linux:**
```bash
curl https://cli-assets.heroku.com/install.sh | sh
```

**Verify installation:**
```bash
heroku --version
# heroku/9.x.x
```

---

### **Step 2: Login to Heroku**

```bash
cd /Users/nikhilvaidyanath/Desktop/GraphLLM

# Login to Heroku
heroku login
# Opens browser for authentication

# Login to Heroku Container Registry (for Docker)
heroku container:login
```

---

### **Step 3: Create Heroku App**

```bash
# Create new Heroku app
heroku create graphllm-rag

# Or with custom name:
heroku create your-custom-name

# Output:
# Creating ⬢ graphllm-rag... done
# https://graphllm-rag-xxxxx.herokuapp.com/ | https://git.heroku.com/graphllm-rag.git
```

**Note:** App name must be unique across all Heroku. If taken, try: `graphllm-rag-nikhil` or `graphllm-demo-2024`

---

### **Step 4: Set Stack to Container**

```bash
# Tell Heroku to use Docker (via heroku.yml)
heroku stack:set container -a graphllm-rag

# Verify
heroku stack -a graphllm-rag
# Should show: container
```

---

### **Step 5: Configure Environment Variables**

```bash
# Set Gemini API key
heroku config:set GEMINI_API_KEY=AIzaSyCCWyinHGS3nL_urDnRLRd1xD6BWcdVqds -a graphllm-rag

# Set other environment variables
heroku config:set \
  PORT=8000 \
  ENVIRONMENT=production \
  LOG_LEVEL=INFO \
  LLM_TEMPERATURE=0.0 \
  GEMINI_MODEL=gemini-2.5-flash \
  -a graphllm-rag

# Verify config
heroku config -a graphllm-rag
```

**Security Tip:** Never commit API keys to GitHub!

---

### **Step 6: Deploy to Heroku**

**Option A: Deploy from Local Git**

```bash
# Add Heroku remote
heroku git:remote -a graphllm-rag

# Push to Heroku (triggers build)
git push heroku main

# Build output:
# -----> Building with Docker
# -----> Building web (Dockerfile)
# -----> Pushing web image to Heroku
# -----> Releasing
# -----> Deployed to https://graphllm-rag-xxxxx.herokuapp.com/
```

**Build time:** 8-12 minutes (first time)

---

**Option B: Connect GitHub (Auto-Deploy)**

1. Go to Heroku Dashboard: https://dashboard.heroku.com/apps/graphllm-rag
2. Click **"Deploy"** tab
3. **Deployment method:** Select **"GitHub"**
4. **Connect to GitHub:** Search and connect `nikhil-7781/GraphLLM`
5. **Automatic deploys:** Enable for `main` branch
6. **Manual deploy:** Click "Deploy Branch" for first deployment

Heroku will now auto-deploy every time you push to GitHub!

---

### **Step 7: Scale Up Dyno**

```bash
# Scale to 1 Eco dyno (cheapest option)
heroku ps:scale web=1 -a graphllm-rag

# Or scale to Basic dyno (never sleeps)
heroku ps:scale web=1:basic -a graphllm-rag

# Check dyno status
heroku ps -a graphllm-rag
```

---

### **Step 8: Open Your App**

```bash
# Open in browser
heroku open -a graphllm-rag

# Or get URL
heroku info -a graphllm-rag | grep "Web URL"
# https://graphllm-rag-xxxxx.herokuapp.com/
```

---

## Monitor Deployment

### **View Logs:**

```bash
# Real-time logs
heroku logs --tail -a graphllm-rag

# Last 1000 lines
heroku logs -n 1000 -a graphllm-rag

# Filter by source
heroku logs --source app --tail -a graphllm-rag
```

**Expected startup logs:**
```
INFO: ✓ PDFProcessor initialized
INFO: ✓ EmbeddingService initialized
INFO: ✓ GraphStore initialized
INFO: ✓ LLMService initialized
INFO: ✓ RAGAgent initialized
INFO: Uvicorn running on http://0.0.0.0:8000
```

---

### **Check Build Logs:**

```bash
# View build logs
heroku builds -a graphllm-rag

# Get specific build
heroku builds:info <BUILD_ID> -a graphllm-rag
```

---

### **Check Dyno Status:**

```bash
# Check running dynos
heroku ps -a graphllm-rag

# Output:
# === web (Eco): python main.py (1)
# web.1: up 2024/01/15 10:30:00 (~ 5m ago)
```

---

## Test Your Deployment

```bash
# Get your Heroku URL
HEROKU_URL=$(heroku info -a graphllm-rag -j | python3 -c "import sys, json; print(json.load(sys.stdin)['app']['web_url'])")

# Test root endpoint
curl $HEROKU_URL
# Should return: {"message":"GraphLLM API is running"}

# Upload a PDF
curl -X POST "$HEROKU_URL/upload" \
  -F "file=@your_pdf.pdf"

# Chat with PDF
curl -X POST "$HEROKU_URL/chat" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is this document about?",
    "pdf_id": "YOUR_PDF_ID"
  }'
```

---

## Configuration Files

### **heroku.yml**
```yaml
build:
  docker:
    web: Dockerfile
run:
  web: python main.py
```

### **Procfile** (fallback if not using Docker)
```
web: python main.py
```

### **runtime.txt** (Python version)
```
python-3.12.0
```

### **Dockerfile** (already exists)
```dockerfile
FROM python:3.12-slim
# ... rest of Dockerfile
```

---

## Scaling & Performance

### **Dyno Types:**

| Dyno Type | RAM | CPU | Price | Sleep? | Best For |
|-----------|-----|-----|-------|--------|----------|
| Eco | 512MB | 1x | $5/mo | Yes (30min) | Development/demos |
| Basic | 512MB | 1x | $7/mo | No | Light production |
| Standard-1X | 512MB | 1x | $25/mo | No | Production |
| Standard-2X | 1GB | 2x | $50/mo | No | Production |
| Performance-M | 2.5GB | Various | $250/mo | No | High-traffic |

### **Scale Dynos:**

```bash
# Scale to Basic (recommended for interviews)
heroku ps:scale web=1:basic -a graphllm-rag

# Scale to Standard-2X (more RAM for ML)
heroku ps:scale web=1:standard-2x -a graphllm-rag

# Scale to multiple dynos (load balancing)
heroku ps:scale web=2:standard-1x -a graphllm-rag
```

### **Increase Build Timeout:**

```bash
# Default timeout: 15 minutes
# For slower builds, contact Heroku support
```

---

## Persistent Storage

**Warning:** Heroku's filesystem is **ephemeral** (resets on each deploy). For persistent data:

### **Option 1: PostgreSQL Add-on (for metadata)**

```bash
# Add PostgreSQL (free tier available)
heroku addons:create heroku-postgresql:mini -a graphllm-rag

# Get database URL
heroku config:get DATABASE_URL -a graphllm-rag
```

### **Option 2: AWS S3 (for files/graphs)**

```bash
# Install AWS S3 add-on
heroku addons:create bucketeer:hobbyist -a graphllm-rag

# Get S3 credentials
heroku config -a graphllm-rag | grep BUCKETEER
```

**Update config.py to use S3 for data storage instead of local filesystem.**

---

## Custom Domain (Optional)

```bash
# Add custom domain
heroku domains:add graphllm.yourdomain.com -a graphllm-rag

# Get DNS target
heroku domains -a graphllm-rag

# Add CNAME record to your DNS:
# CNAME graphllm -> xxx-xxx-xxx.herokudns.com

# Enable Automated Certificate Management (SSL)
heroku certs:auto:enable -a graphllm-rag
```

**SSL certificate is automatically provisioned (free).**

---

## Updating Your App

### **Auto-Deploy from GitHub:**

If you connected GitHub (Step 6, Option B):

```bash
# Make changes locally
git add .
git commit -m "Update feature X"
git push origin main

# Heroku automatically:
# ✓ Detects push
# ✓ Rebuilds Docker image
# ✓ Deploys new version
```

### **Manual Deploy:**

```bash
# Push directly to Heroku
git push heroku main
```

---

## Monitoring & Debugging

### **View Metrics:**

```bash
# Open metrics dashboard
heroku metrics -a graphllm-rag

# Or in browser:
# https://dashboard.heroku.com/apps/graphllm-rag/metrics
```

Shows: Response time, throughput, memory usage, errors

### **Restart Dyno:**

```bash
# Restart all dynos
heroku restart -a graphllm-rag

# Restart specific dyno
heroku restart web.1 -a graphllm-rag
```

### **Run One-Off Commands:**

```bash
# Open bash in running dyno
heroku run bash -a graphllm-rag

# Run Python shell
heroku run python -a graphllm-rag

# Check installed packages
heroku run pip list -a graphllm-rag
```

---

## Troubleshooting

### **Build Fails**

**Issue:** Docker build timeout or errors

**Solution:**
```bash
# Check build logs
heroku builds -a graphllm-rag

# Common fixes:
# 1. Increase dyno size during build
heroku ps:scale web=1:standard-2x -a graphllm-rag

# 2. Check Dockerfile syntax
docker build -t graphllm .

# 3. Clear build cache
heroku plugins:install heroku-builds
heroku builds:cache:purge -a graphllm-rag
```

### **App Crashes on Start**

**Issue:** Container exits immediately after start

**Solution:**
```bash
# Check logs
heroku logs --tail -a graphllm-rag

# Common fixes:
# 1. Verify environment variables
heroku config -a graphllm-rag

# 2. Check GEMINI_API_KEY is set
heroku config:get GEMINI_API_KEY -a graphllm-rag

# 3. Increase dyno RAM
heroku ps:scale web=1:standard-2x -a graphllm-rag
```

### **Out of Memory (R14)**

**Issue:** `Error R14 (Memory quota exceeded)`

**Solution:**
```bash
# Upgrade to dyno with more RAM
heroku ps:scale web=1:standard-2x -a graphllm-rag  # 1GB RAM
heroku ps:scale web=1:performance-m -a graphllm-rag  # 2.5GB RAM
```

### **Slow First Request (Cold Start)**

**Issue:** First request takes 30+ seconds

**Solution:**
```bash
# Upgrade from Eco to Basic (never sleeps)
heroku ps:scale web=1:basic -a graphllm-rag

# Or use Heroku Scheduler to keep warm
heroku addons:create scheduler:standard -a graphllm-rag
# Add job: curl https://your-app.herokuapp.com/
# Runs every 10 minutes to keep dyno awake
```

### **H12 Request Timeout**

**Issue:** Request exceeds 30-second timeout

**Solution:**
- Heroku has a **30-second request timeout** (unchangeable)
- For long-running tasks, use background workers with Celery/RQ
- Or switch to a platform without timeouts (Northflank, Modal, etc.)

---

## Cost Breakdown

### **Monthly Cost Estimate:**

| Component | Cost | Notes |
|-----------|------|-------|
| **Eco Dyno** | $5/mo | Sleeps after 30min |
| **Basic Dyno** | $7/mo | Never sleeps (recommended) |
| **Standard-2X** | $50/mo | 1GB RAM (better for ML) |
| **PostgreSQL (Mini)** | $5/mo | Optional (for metadata) |
| **Total (Basic)** | **$7-12/mo** | Good for demos/interviews |

**For GraphLLM interviews:** Basic dyno ($7/mo) is sufficient!

---

## Production Checklist

Before sharing with interviewers:

- [ ] App deployed successfully
- [ ] Environment variables set (especially `GEMINI_API_KEY`)
- [ ] Using Basic or Standard dyno (not Eco for demos)
- [ ] Test PDF upload works
- [ ] Test graph generation works
- [ ] Test chat functionality works
- [ ] Check logs for errors
- [ ] Custom domain configured (optional)
- [ ] Monitoring enabled

---

## Heroku CLI Cheat Sheet

```bash
# Login
heroku login

# Create app
heroku create app-name

# Set stack to Docker
heroku stack:set container -a app-name

# Set environment variable
heroku config:set KEY=VALUE -a app-name

# View config
heroku config -a app-name

# Deploy
git push heroku main

# View logs
heroku logs --tail -a app-name

# Restart
heroku restart -a app-name

# Open app
heroku open -a app-name

# Scale dynos
heroku ps:scale web=1:basic -a app-name

# Check status
heroku ps -a app-name

# Run command
heroku run bash -a app-name

# Add add-on
heroku addons:create addon-name:plan -a app-name

# Delete app (careful!)
heroku apps:destroy app-name --confirm app-name
```

---

## Alternative: GitHub Actions + Heroku

Automate deployment via GitHub Actions:

**.github/workflows/heroku-deploy.yml:**
```yaml
name: Deploy to Heroku

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: akhileshns/heroku-deploy@v3.12.14
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_app_name: "graphllm-rag"
          heroku_email: "your-email@example.com"
          usedocker: true
```

Get API key: `heroku auth:token`

Add to GitHub Secrets: `HEROKU_API_KEY`

---

## Interview Talking Points

> "I've deployed GraphLLM to Heroku using Docker containerization for better ML dependency management. Heroku provides excellent platform-as-a-service features including auto-scaling, built-in monitoring, and zero-downtime deployments. The app uses a Docker-based workflow with automated CI/CD from GitHub, so every push triggers a new build and deployment. I've configured environment variables for the Gemini API and set up appropriate resource limits for the ML workload."

Shows: DevOps knowledge, Docker proficiency, PaaS experience, CI/CD automation.

---

## Support & Resources

- **Heroku Docs:** https://devcenter.heroku.com
- **Heroku Status:** https://status.heroku.com
- **Support:** https://help.heroku.com
- **Pricing:** https://www.heroku.com/pricing
- **Dev Center:** https://devcenter.heroku.com/categories/reference

---

Good luck with your deployment! 🚀
