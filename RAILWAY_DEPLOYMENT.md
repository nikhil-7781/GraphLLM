# Railway.app Deployment Guide for GraphLLM

## 🚂 Railway Deployment (2 Minutes)

Railway.app is perfect for quick demos and prototypes. It offers automatic deployments from GitHub and great developer experience.

---

## Prerequisites

1. **Railway Account** (free)
   - Sign up at https://railway.app (use GitHub OAuth)
   - No credit card required for trial ($5 credit)

2. **Gemini API Key** (required)
   - Get from: https://makersuite.google.com/app/apikey
   - Keep it ready for environment variable setup

---

## Deployment Options

### Option A: Deploy from GitHub (Recommended ⭐)

**Step 1: Push to GitHub**
```bash
cd /Users/nikhilvaidyanath/Desktop/GraphLLM

# Initialize git (if not already)
git init
git add .
git commit -m "Initial commit for Railway deployment"

# Create GitHub repo and push
gh repo create graphllm --public --source=. --remote=origin --push
# OR manually: create repo on GitHub, then:
git remote add origin https://github.com/YOUR_USERNAME/graphllm.git
git push -u origin main
```

**Step 2: Deploy on Railway**
1. Go to https://railway.app/new
2. Click "Deploy from GitHub repo"
3. Select your `graphllm` repository
4. Railway will auto-detect Python and start deploying
5. Wait 3-5 minutes for first build

**Step 3: Add Environment Variables**
1. Click on your service in Railway dashboard
2. Go to "Variables" tab
3. Add these variables:
   ```
   GEMINI_API_KEY=your_actual_gemini_key_here
   PORT=8000
   ENVIRONMENT=production
   LOG_LEVEL=INFO
   ```

**Step 4: Generate Domain**
1. Go to "Settings" tab
2. Click "Generate Domain"
3. Your app will be available at: `https://graphllm-production-XXXX.up.railway.app`

**Done!** Auto-deploys on every git push.

---

### Option B: Deploy from CLI (Faster for Testing)

**Step 1: Install Railway CLI**
```bash
# macOS (Homebrew)
brew install railway

# Or using NPM
npm install -g @railway/cli

# Verify installation
railway --version
```

**Step 2: Login**
```bash
railway login
# Opens browser for authentication
```

**Step 3: Initialize and Deploy**
```bash
cd /Users/nikhilvaidyanath/Desktop/GraphLLM

# Initialize Railway project
railway init
# Choose: "Empty Project"
# Name: graphllm

# Link to project
railway link

# Add environment variables
railway variables set GEMINI_API_KEY="your_gemini_key_here"
railway variables set PORT="8000"
railway variables set ENVIRONMENT="production"
railway variables set LOG_LEVEL="INFO"

# Deploy
railway up
# Uploads code and starts build

# Get deployment URL
railway open
```

**Done!** Your app is live in 2-3 minutes.

---

## Configuration Files

Railway uses these files (already created):

1. **railway.toml** - Railway configuration
2. **nixpacks.toml** - Build configuration (Nixpacks)
3. **Procfile** - Start command
4. **.railwayignore** - Files to exclude from deployment

---

## Environment Variables Required

Set these in Railway dashboard or via CLI:

```bash
# Required
GEMINI_API_KEY=your_gemini_api_key_here

# Optional (have defaults in .env)
PORT=8000
ENVIRONMENT=production
DEBUG=false
LOG_LEVEL=INFO
LLM_TEMPERATURE=0.0
LLM_MAX_TOKENS=3072
GEMINI_MODEL=gemini-2.5-flash
EMBEDDING_MODEL=sentence-transformers/multi-qa-MiniLM-L6-cos-v1
MAX_FILE_SIZE_MB=50
```

---

## Monitoring Your Deployment

### Railway Dashboard

1. **Deployments Tab**
   - View build logs
   - See deployment status
   - Rollback to previous versions

2. **Metrics Tab**
   - CPU usage
   - Memory usage
   - Network traffic

3. **Logs Tab**
   - Real-time application logs
   - Filter by severity

### CLI Monitoring

```bash
# View logs
railway logs

# Follow logs in real-time
railway logs -f

# Check service status
railway status

# View metrics
railway metrics
```

---

## Storage & Volumes

Railway provides ephemeral storage by default. For persistent storage:

### Option 1: Railway Volumes (Recommended)

```bash
# Add volume via Railway dashboard
# Settings → Volumes → New Volume
# Mount path: /app/data
# Size: 1GB (free tier)
```

Then update code to use `/app/data` for persistence.

### Option 2: External Storage (S3, GCS)

Use cloud storage for PDFs and graphs:
- AWS S3
- Google Cloud Storage
- Cloudflare R2 (free 10GB)

---

## Troubleshooting

### Build Fails

**Issue:** Build timeout or dependency errors

**Solution:**
```bash
# Check build logs
railway logs --deployment

# Common fixes:
# 1. Increase build timeout in railway.toml
# 2. Remove unused dependencies from requirements.txt
# 3. Use lighter packages (e.g., torch-cpu instead of torch)
```

### App Crashes on Startup

**Issue:** `ModuleNotFoundError` or import errors

**Solution:**
```bash
# Verify requirements.txt is complete
# Check Railway build logs for missing packages

# Test locally first
pip install -r requirements.txt
python main.py
```

### Out of Memory

**Issue:** App crashes with memory errors

**Solution:**
```bash
# Upgrade Railway plan (more RAM)
# Or optimize code:
# - Reduce embedding batch size
# - Process smaller PDFs
# - Clear caches more frequently
```

### Slow Response Times

**Issue:** API takes > 5s to respond

**Solution:**
```bash
# Check logs for bottlenecks
railway logs -f

# Common causes:
# 1. Cold start (first request after idle) - normal
# 2. Large PDF processing - expected
# 3. Gemini API latency - external
```

---

## Costs & Limits

### Free Trial
- **$5 credit** (lasts ~1 month with light usage)
- 500 hours execution time
- 100GB network egress
- 1GB storage

### After Trial
- **Hobby Plan:** $5/month (included credit)
- **Pro Plan:** $20/month (more resources)

### Cost Optimization
```bash
# Monitor usage
railway metrics

# Optimize:
# 1. Reduce memory allocation (Settings → Memory)
# 2. Use sleep mode for development (Settings → Sleep)
# 3. Implement caching (reduce LLM calls)
```

---

## CI/CD with GitHub

### Automatic Deployments

Railway auto-deploys on every push to `main`:

```bash
# Make changes
git add .
git commit -m "Update feature X"
git push origin main

# Railway automatically:
# 1. Detects push
# 2. Builds new image
# 3. Deploys if build succeeds
# 4. Keeps old version running until new one is healthy
```

### Branch Deployments

Deploy feature branches for testing:

```bash
# Create feature branch
git checkout -b feature/new-ui
git push origin feature/new-ui

# Railway creates separate deployment
# URL: https://graphllm-feature-new-ui-XXXX.up.railway.app
```

---

## Scaling Up

### When to Upgrade

Upgrade when you hit these limits:
- > 100 PDFs uploaded
- > 1000 queries/day
- > 512MB memory needed
- Need > 1GB storage

### Upgrade Path

1. **Railway Pro:** $20/month
   - 8GB RAM
   - 100GB storage
   - Priority support

2. **Migrate to GCP/AWS:** For production scale
   - See TECHNICAL_OVERVIEW.md for architecture changes

---

## Production Checklist

Before sharing with interviewers:

- [ ] Environment variables set correctly
- [ ] Custom domain configured (optional)
- [ ] HTTPS enabled (automatic on Railway)
- [ ] Logs verified (no errors)
- [ ] Test PDF upload works
- [ ] Test graph generation works
- [ ] Test chat functionality works
- [ ] Response times acceptable (< 3s)
- [ ] Error handling graceful

---

## Quick Commands Reference

```bash
# Deploy
railway up

# View logs
railway logs -f

# Add environment variable
railway variables set KEY=value

# Open app in browser
railway open

# Check status
railway status

# Restart service
railway restart

# Delete project
railway delete
```

---

## Support & Resources

- **Railway Docs:** https://docs.railway.app
- **Railway Discord:** https://discord.gg/railway
- **Status Page:** https://status.railway.app

---

## Estimated Timeline

| Step | Time |
|------|------|
| Create Railway account | 1 min |
| Push to GitHub | 2 min |
| Deploy from GitHub | 3-5 min |
| Configure env vars | 1 min |
| Test deployment | 2 min |
| **Total** | **10-12 minutes** ⚡ |

---

## What Interviewers Will See

When you share your Railway URL:

✅ **Fast load times** (Railway has global CDN)
✅ **Always-on** (no cold starts during demo)
✅ **Professional URL** (custom domain optional)
✅ **SSL certificate** (automatic HTTPS)
✅ **Stable deployment** (no random crashes)

---

## Example Interview Talking Points

> "I've deployed GraphLLM to Railway, which provides automatic CI/CD from GitHub. The deployment process takes under 3 minutes, and it auto-scales based on demand. Railway uses Nixpacks for build automation, which detected my Python app and installed all dependencies automatically. I've configured persistent volumes for graph storage and integrated environment variables for secure API key management."

Shows: DevOps knowledge, cloud deployment experience, security awareness.

---

Good luck with your deployment! 🚀
