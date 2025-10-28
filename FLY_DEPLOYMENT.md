# Fly.io Deployment Guide for GraphLLM

Complete guide to deploy GraphLLM on Fly.io with minimal disk usage on your local machine.

## Why Fly.io?

- ✅ **Generous Free Tier**: 3 shared VMs, 3GB persistent storage
- ✅ **Minimal Local Requirements**: Only 50MB for CLI tool
- ✅ **Builds in Cloud**: Docker image built remotely, not on your machine
- ✅ **Persistent Storage**: 3GB volumes included free
- ✅ **Automatic HTTPS**: SSL certificates included
- ✅ **Global CDN**: Fast worldwide access
- ✅ **No Credit Card**: Free tier works without payment method

---

## Prerequisites

- **Disk Space**: Only 50MB (for flyctl CLI)
- **RAM**: Any amount (builds happen in cloud)
- **Gemini API Key**: [Get free key](https://makersuite.google.com/app/apikey)

---

## Step-by-Step Deployment (15 minutes)

### Step 1: Install Fly.io CLI

**macOS/Linux:**
```bash
curl -L https://fly.io/install.sh | sh
```

**Windows (PowerShell):**
```powershell
pwsh -Command "iwr https://fly.io/install.ps1 -useb | iex"
```

Verify installation:
```bash
flyctl version
```

### Step 2: Create Fly.io Account (Free)

```bash
# Sign up / login
flyctl auth signup
# Or if you already have account
flyctl auth login
```

This opens your browser for authentication. **No credit card required!**

### Step 3: Navigate to Your Project

```bash
cd /Users/nikhilvaidyanath/Desktop/GraphLLM
```

### Step 4: Create Fly.io App

```bash
# Create app (choose unique name)
flyctl apps create graphllm-YOUR-NAME

# Or let Fly generate random name
flyctl apps create
```

**Note**: App name must be globally unique. If "graphllm" is taken, try:
- `graphllm-yourname`
- `graphllm-demo`
- `my-graphllm`

### Step 5: Update fly.toml with Your App Name

Edit `fly.toml` and change the first line:
```toml
app = "graphllm-YOUR-NAME"  # Use the name from Step 4
```

### Step 6: Choose Region (Optional)

Check available regions:
```bash
flyctl platform regions
```

Common regions:
- **sjc** - San Jose, California (US West)
- **ord** - Chicago, Illinois (US Central)
- **iad** - Ashburn, Virginia (US East)
- **lhr** - London (Europe)
- **syd** - Sydney (Australia)
- **sin** - Singapore (Asia)

Update in `fly.toml` if needed:
```toml
primary_region = "ord"  # Change to your preferred region
```

### Step 7: Create Persistent Volumes

**For data (FAISS index, graphs):**
```bash
flyctl volumes create graphllm_data --size 3 --region sjc
```

**For uploads (PDF files):**
```bash
flyctl volumes create graphllm_uploads --size 3 --region sjc
```

**Important**: Replace `sjc` with your chosen region from Step 6.

### Step 8: Set Environment Secrets

```bash
# Set your Gemini API key (REQUIRED)
flyctl secrets set GEMINI_API_KEY=your_gemini_api_key_here

# Optional: Override other settings
flyctl secrets set GEMINI_MODEL=gemini-1.5-flash
flyctl secrets set LLM_TEMPERATURE=0.0
```

View secrets (values hidden):
```bash
flyctl secrets list
```

### Step 9: Deploy! 🚀

```bash
# Deploy (builds in cloud, not on your machine!)
flyctl deploy

# This will:
# - Upload your code to Fly.io
# - Build Docker image in the cloud
# - Create and start your VM
# - Configure networking and SSL
```

**Expected output:**
```
==> Building image
...
==> Pushing image to fly
...
==> Deploying
...
 ✓ [1/1] Machine deployed successfully
```

**Time**: 5-10 minutes for first deployment

### Step 10: Access Your App

```bash
# Open in browser
flyctl open

# Or get the URL
flyctl info
```

Your app will be at: `https://your-app-name.fly.dev`

---

## Verification

### Check Status

```bash
# View app status
flyctl status

# Expected output:
# Machines
# PROCESS ID              VERSION REGION  STATE   
# app     xxxxxxxxxxxxx   1       sjc     started
```

### View Logs

```bash
# Stream logs in real-time
flyctl logs

# Or follow logs
flyctl logs -f
```

### Test Endpoints

```bash
# Health check
curl https://your-app-name.fly.dev/

# Admin status
curl https://your-app-name.fly.dev/admin/status

# API docs
# Visit: https://your-app-name.fly.dev/docs
```

---

## Common Issues & Solutions

### Issue 1: "App name already taken"

**Solution**: Choose a different name in Step 4
```bash
flyctl apps create graphllm-myname123
# Then update fly.toml with this name
```

### Issue 2: "Volume region mismatch"

**Error**: Volume in different region than app

**Solution**: Create volume in same region as `primary_region` in fly.toml
```bash
flyctl volumes create graphllm_data --size 3 --region YOUR_REGION
```

### Issue 3: "Out of memory during build"

**Solution**: This shouldn't happen (builds in cloud). If it does:
```bash
# Increase build VM size temporarily
flyctl deploy --vm-size shared-cpu-2x
```

### Issue 4: "App keeps crashing"

**Check logs**:
```bash
flyctl logs

# Common causes:
# 1. Missing GEMINI_API_KEY
flyctl secrets set GEMINI_API_KEY=your_key

# 2. Python dependencies failed
# Check logs for pip install errors

# 3. Port mismatch
# Ensure API_PORT=8000 in fly.toml
```

### Issue 5: "Slow first request"

**Expected behavior**: First request after deploy takes 30-60s
- Model loading (sentence-transformers)
- spaCy model initialization

**Solution**: Just wait, subsequent requests are fast.

---

## Management Commands

### Update Application (After Code Changes)

```bash
# Deploy new version
flyctl deploy

# Or force rebuild
flyctl deploy --no-cache
```

### View App Info

```bash
# General info
flyctl info

# VM status
flyctl status

# Resource usage
flyctl vm status

# Volumes
flyctl volumes list
```

### Scaling

**Upgrade to more RAM** (if hitting limits):
```bash
# Edit fly.toml and change:
[vm]
  memory_mb = 512  # or 1024

# Then redeploy
flyctl deploy
```

**Note**: Free tier is 256MB. Upgrading costs ~$3-5/month.

### Restart Application

```bash
# Restart all VMs
flyctl apps restart graphllm

# Or restart specific machine
flyctl machine restart MACHINE_ID
```

### SSH into VM

```bash
# Open SSH session
flyctl ssh console

# Once inside, check application
cd /app
ls -la data/
ls -la uploads/
```

### View Environment Variables

```bash
# Non-secret vars (from fly.toml)
flyctl config display

# Secret vars (values hidden)
flyctl secrets list
```

---

## Monitoring & Debugging

### Real-Time Logs

```bash
# Follow all logs
flyctl logs -f

# Filter by level
flyctl logs -f | grep ERROR

# Last 100 lines
flyctl logs -n 100
```

### Check Resource Usage

```bash
# Memory, CPU usage
flyctl vm status

# Detailed metrics (if enabled)
flyctl metrics
```

### Access Dashboard

```bash
# Open Fly.io dashboard
flyctl dashboard
```

Or visit: https://fly.io/dashboard

---

## Persistent Storage Management

### View Volumes

```bash
# List all volumes
flyctl volumes list

# Expected output:
# ID        NAME              SIZE  REGION  ATTACHED VM
# vol_xxx   graphllm_data     3GB   sjc     yes
# vol_yyy   graphllm_uploads  3GB   sjc     yes
```

### Backup Volumes

```bash
# Create snapshot
flyctl volumes snapshot create graphllm_data

# List snapshots
flyctl volumes snapshots list graphllm_data
```

### Extend Volume Size (if needed)

```bash
# Increase to 5GB (costs extra)
flyctl volumes extend graphllm_data --size 5
```

### Delete Volume (WARNING: Data loss!)

```bash
flyctl volumes delete vol_xxxxx
```

---

## Cost Breakdown

### Free Tier Includes:
- ✅ Up to 3 shared-cpu-1x VMs (256MB RAM each)
- ✅ 3GB persistent storage (volumes)
- ✅ 160GB outbound data transfer/month
- ✅ Automatic SSL certificates
- ✅ Global Anycast network

### What's Included for GraphLLM:
- ✅ 1 VM (256MB RAM) - **FREE**
- ✅ 2 volumes (3GB each = 6GB total) - **FREE**
- ✅ SSL/HTTPS - **FREE**
- ✅ Data transfer - **FREE** (under 160GB/month)

**Total monthly cost**: **$0** (Free tier)

### If You Need More:

| Resource | Free Tier | Upgrade | Cost |
|----------|-----------|---------|------|
| RAM | 256MB | 512MB | +$1.94/month |
| RAM | 256MB | 1024MB | +$5.82/month |
| Storage | 6GB | 10GB | +$0.15/GB/month |
| VMs | 1 | 2 (HA) | +$5-10/month |

For most use cases, **free tier is sufficient**.

---

## Custom Domain (Optional)

### Add Your Domain

```bash
# Add domain
flyctl certs create yourdomain.com

# Get DNS records to add
flyctl certs show yourdomain.com
```

Add these DNS records at your domain registrar:
```
CNAME @ your-app-name.fly.dev
```

Wait for SSL cert (5-10 minutes):
```bash
flyctl certs check yourdomain.com
```

---

## Production Best Practices

### 1. Use Secrets for Sensitive Data

```bash
# Never put API keys in fly.toml
# Always use secrets
flyctl secrets set GEMINI_API_KEY=xxx
```

### 2. Set Up Health Checks

Already configured in `fly.toml`:
```toml
[[services.tcp_checks]]
  grace_period = "60s"
  interval = "15s"
  timeout = "5s"
```

### 3. Enable Auto-Scaling (Optional)

Edit `fly.toml`:
```toml
[auto_stop_machines]
  enabled = true
  min_machines_running = 0

[auto_start_machines]
  enabled = true
```

This stops VMs when idle (save costs) and starts on request.

### 4. Monitor Logs

```bash
# Set up log shipping (optional)
flyctl logs -f | tee app.log

# Or use external service like:
# - Papertrail
# - Logtail
# - Sentry
```

### 5. Regular Backups

```bash
# Add to cron/schedule
flyctl volumes snapshot create graphllm_data
flyctl volumes snapshot create graphllm_uploads
```

---

## Troubleshooting Checklist

If your app isn't working:

1. **Check deployment status**
   ```bash
   flyctl status
   ```

2. **View recent logs**
   ```bash
   flyctl logs -n 100
   ```

3. **Verify secrets are set**
   ```bash
   flyctl secrets list
   # Should show: GEMINI_API_KEY
   ```

4. **Check volumes are attached**
   ```bash
   flyctl volumes list
   # Should show "yes" in ATTACHED VM column
   ```

5. **Test from inside VM**
   ```bash
   flyctl ssh console
   curl localhost:8000
   ```

6. **Verify environment**
   ```bash
   flyctl ssh console
   env | grep GEMINI
   ```

---

## Updating Your App

After making code changes:

```bash
# 1. Commit changes (optional, if using git)
git add .
git commit -m "Update feature"

# 2. Deploy new version
flyctl deploy

# 3. Verify deployment
flyctl status
flyctl logs -f
```

That's it! Fly.io handles:
- Building new Docker image
- Graceful shutdown of old version
- Starting new version
- Health checks
- Rollback if health checks fail

---

## Cleanup / Deletion

### Delete App (Keeps volumes)

```bash
flyctl apps destroy graphllm
```

### Delete Volumes (Delete data)

```bash
flyctl volumes delete vol_xxxxx
```

### Complete Cleanup

```bash
# 1. Delete app
flyctl apps destroy graphllm

# 2. Delete volumes
flyctl volumes list
flyctl volumes delete vol_xxxxx
flyctl volumes delete vol_yyyyy

# 3. Verify
flyctl apps list
flyctl volumes list
```

---

## Getting Help

### Official Resources
- **Fly.io Docs**: https://fly.io/docs
- **Community Forum**: https://community.fly.io
- **Status Page**: https://status.fly.io

### Common Commands Reference

```bash
# Deploy
flyctl deploy

# Status
flyctl status

# Logs
flyctl logs -f

# SSH
flyctl ssh console

# Restart
flyctl apps restart

# Secrets
flyctl secrets list
flyctl secrets set KEY=value

# Volumes
flyctl volumes list
flyctl volumes snapshot create NAME

# Info
flyctl info
flyctl dashboard
```

---

## Next Steps

After successful deployment:

1. ✅ Upload a test PDF at `https://your-app.fly.dev`
2. ✅ Check graph visualization works
3. ✅ Test RAG chat functionality
4. ✅ Monitor logs for any errors
5. ✅ Set up regular volume snapshots
6. ✅ (Optional) Add custom domain

---

## Support

If you encounter issues:

1. Check logs: `flyctl logs -f`
2. Check Fly.io status: https://status.fly.io
3. Search Fly.io community: https://community.fly.io
4. GitHub issues: (your repo)

**Your app is now live at**: `https://your-app-name.fly.dev` 🎉

---

## Quick Reference Card

```bash
# Deploy
flyctl deploy

# Status & Logs
flyctl status
flyctl logs -f

# Manage
flyctl apps restart
flyctl ssh console

# Secrets
flyctl secrets set GEMINI_API_KEY=xxx

# Access
flyctl open
flyctl dashboard

# Help
flyctl help
flyctl help deploy
```

**Total deployment time**: ~15 minutes  
**Local disk usage**: ~50MB  
**Monthly cost**: $0 (free tier)

Enjoy your deployed GraphLLM app! 🚀
