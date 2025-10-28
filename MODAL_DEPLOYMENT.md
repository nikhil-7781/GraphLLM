# Modal Deployment Guide for GraphLLM

## 🚀 Quick Start (5 Minutes)

Modal is perfect for ML/AI apps with **$30/month free credits** and pay-per-use pricing.

---

## Prerequisites

1. **Modal Account** (free $30 credits)
   - Sign up at: https://modal.com
   - No credit card required

2. **Gemini API Key**
   - From: https://makersuite.google.com/app/apikey

---

## Step 1: Install Modal CLI

```bash
cd /Users/nikhilvaidyanath/Desktop/GraphLLM

# Activate venv
source venv/bin/activate

# Install Modal (already done)
pip install modal

# Authenticate
modal setup
```

This will open a browser for authentication.

---

## Step 2: Create Modal Secret

Store your Gemini API key securely:

```bash
# Create secret in Modal dashboard OR via CLI:
modal secret create graphllm-secrets \
  GEMINI_API_KEY=AIzaSyCCWyinHGS3nL_urDnRLRd1xD6BWcdVqds \
  PORT=8000 \
  ENVIRONMENT=production \
  LOG_LEVEL=INFO
```

**OR** in Modal Dashboard:
1. Go to https://modal.com/secrets
2. Click "Create Secret"
3. Name: `graphllm-secrets`
4. Add variables:
   ```
   GEMINI_API_KEY = AIzaSyCCWyinHGS3nL_urDnRLRd1xD6BWcdVqds
   PORT = 8000
   ENVIRONMENT = production
   LOG_LEVEL = INFO
   ```

---

## Step 3: Deploy to Modal

```bash
# Deploy the app
modal deploy modal_app.py

# Output:
# ✓ Created objects.
# ├── 🔨 Created mount /Users/.../GraphLLM
# ├── 🔨 Created function fastapi_app.
# └── 🔨 Created  App(graphllm)
#
# View app at https://modal.com/apps/YOUR_USERNAME/graphllm
#
# Your app URL:
# https://YOUR_USERNAME--graphllm-fastapi-app.modal.run
```

**Deployment time:** 2-3 minutes (first time), < 30s (subsequent)

---

## Step 4: Test Your Deployment

```bash
# Get your Modal URL from deploy output
MODAL_URL="https://YOUR_USERNAME--graphllm-fastapi-app.modal.run"

# Test root endpoint
curl $MODAL_URL/

# Should return:
# {"message":"GraphLLM API is running"}

# Upload a PDF
curl -X POST "$MODAL_URL/upload" \
  -F "file=@your_pdf.pdf"

# Chat with the PDF
curl -X POST "$MODAL_URL/chat" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is this document about?",
    "pdf_id": "YOUR_PDF_ID"
  }'
```

---

## Modal App Structure

```
modal_app.py
├── App("graphllm")
├── Image: Python 3.12 + dependencies
├── Volume: Persistent storage for data/
├── Secrets: GEMINI_API_KEY, etc.
└── Function: FastAPI app (4GB RAM, CPU)
```

---

## Features

### ✅ **Auto-Scaling**
- Scales to zero when idle (no cost)
- Auto-scales up on traffic
- Pay only for actual usage

### ✅ **Generous Resources**
- **Memory:** 4GB RAM (configurable up to 1TB)
- **Storage:** Persistent volume for data
- **Timeout:** 10 minutes per request
- **CPU:** Fast x86_64 CPUs

### ✅ **Free Tier**
- **$30/month credits** (lasts 2-3 months for light usage)
- After credits: ~$0.10-0.50/hour when active
- **Scales to zero** when idle = $0 cost

---

## Monitoring & Logs

### **View Logs:**
```bash
# Stream logs in real-time
modal app logs graphllm

# Or in dashboard:
# https://modal.com/apps/YOUR_USERNAME/graphllm
```

### **View Metrics:**
- Dashboard: https://modal.com/apps
- Shows: requests/sec, latency, costs, errors

---

## Updating Your App

```bash
# Make code changes
git add .
git commit -m "Update feature X"

# Redeploy (takes < 30s)
modal deploy modal_app.py

# Modal automatically:
# ✓ Builds new image (if requirements changed)
# ✓ Deploys new version
# ✓ Zero-downtime update
```

---

## Cost Breakdown

### **Free Tier:**
- $30 credits/month
- Typical usage for GraphLLM:
  - 100 PDF uploads: ~$2
  - 1000 queries: ~$3
  - **Total:** ~$5/month = **6 months free**

### **After Free Credits:**
- **CPU:** $0.0001/sec = $0.36/hour
- **RAM (4GB):** $0.0001/GB/sec = $1.44/hour
- **Storage:** $0.10/GB/month
- **Total when active:** ~$1.80/hour
- **When idle:** $0 (scales to zero)

**Example monthly cost (light usage):**
- 10 hours active/month = $18
- Storage (5GB) = $0.50
- **Total:** ~$18.50/month (after free credits)

---

## Advanced Configuration

### **Increase Memory:**
```python
@app.function(
    memory=8192,  # 8GB RAM
    ...
)
```

### **Add GPU (if needed):**
```python
@app.function(
    gpu="T4",  # NVIDIA T4 GPU
    ...
)
```

### **Custom Timeout:**
```python
@app.function(
    timeout=1800,  # 30 minutes
    ...
)
```

---

## Troubleshooting

### **Import Errors**

**Issue:** `ModuleNotFoundError`

**Fix:** Ensure all dependencies are in requirements.txt
```bash
modal deploy modal_app.py --force-build
```

### **Storage Issues**

**Issue:** Data not persisting

**Fix:** Check volume mount
```python
volumes={"/app/data": volume}  # Must match DATA_DIR in config
```

### **Timeout Errors**

**Issue:** Request timeout after 10 minutes

**Fix:** Increase timeout or optimize processing
```python
timeout=1800  # 30 minutes
```

---

## Production Checklist

Before sharing with interviewers:

- [ ] Modal secret `graphllm-secrets` created
- [ ] App deployed successfully
- [ ] Test PDF upload works
- [ ] Test graph generation works
- [ ] Test chat functionality works
- [ ] Check logs for errors
- [ ] Verify persistent storage working
- [ ] Custom domain configured (optional)

---

## Custom Domain (Optional)

Modal supports custom domains:

1. Go to https://modal.com/settings/domains
2. Add your domain (e.g., graphllm.yourdomain.com)
3. Update DNS records
4. Modal handles SSL automatically

---

## Comparison: Modal vs Railway

| Feature | Modal | Railway |
|---------|-------|---------|
| Free Tier | $30 credits/mo | $5 trial credits |
| Storage | Unlimited | 1GB (exceeded) |
| RAM | 4GB-1TB | 512MB |
| ML Optimized | ✅ Yes | ❌ No |
| Auto-Scaling | ✅ Yes | ⚠️ Limited |
| Cold Start | ~2s | ~5s |
| Cost (idle) | $0 | $0 (sleeps) |
| Cost (active) | ~$1.80/hr | $7/month |

---

## Support & Resources

- **Modal Docs:** https://modal.com/docs
- **Modal Discord:** https://discord.gg/modal
- **Status Page:** https://status.modal.com

---

## Quick Commands Reference

```bash
# Setup
modal setup

# Deploy
modal deploy modal_app.py

# View logs
modal app logs graphllm

# Run locally
modal run modal_app.py

# List apps
modal app list

# Delete app
modal app delete graphllm
```

---

## Interview Talking Points

> "I've deployed GraphLLM to Modal, which is a serverless platform optimized for ML/Python applications. Modal handles auto-scaling, so the app scales to zero when idle (no cost) and auto-scales up based on traffic. The deployment uses 4GB of RAM and persistent volumes for graph storage. Modal's architecture is perfect for ML workloads, with support for GPU acceleration if needed in the future."

Shows: Cloud deployment experience, cost optimization, scalability awareness.

---

Good luck with your deployment! 🚀
