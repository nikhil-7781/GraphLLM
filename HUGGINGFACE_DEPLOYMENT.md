# Hugging Face Spaces Deployment Guide

## 🚀 Quick Deploy (10 Minutes)

Hugging Face Spaces is perfect for ML/AI apps with **FREE** hosting and great Docker support.

---

## Why Hugging Face Spaces?

✅ **Free Tier:**
- 16GB RAM
- 2 vCPU
- 50GB storage
- **$0/month forever**

✅ **Features:**
- Docker native
- GitHub auto-sync
- Built-in secrets management
- Public/Private spaces
- Custom domains
- Great for ML models

✅ **Perfect for GraphLLM:**
- Free GPU support (if needed)
- Large model hosting
- No cold starts
- Persistent storage

---

## Prerequisites

1. **Hugging Face Account** (free)
   - Sign up: https://huggingface.co/join
   - Can use GitHub OAuth

2. **GitHub Repository**
   - Already have: https://github.com/nikhil-7781/GraphLLM

3. **Gemini API Key**
   - From: https://makersuite.google.com/app/apikey

---

## Step-by-Step Deployment

### **Step 1: Create a New Space**

1. Go to https://huggingface.co/spaces
2. Click **"Create new Space"**
3. Fill in details:
   - **Space name:** `graphllm` (or your choice)
   - **License:** MIT
   - **SDK:** **Docker** (important!)
   - **Space hardware:** CPU basic (free) - sufficient for demo
   - **Visibility:** Public or Private

4. Click **"Create Space"**

---

### **Step 2: Connect Your GitHub Repository**

**Option A: Push to HF Spaces Git (Simpler)**

```bash
cd /Users/nikhilvaidyanath/Desktop/GraphLLM

# Add HF Spaces as remote
git remote add hf https://huggingface.co/spaces/YOUR_USERNAME/graphllm

# Copy README for HF Spaces
cp README_HF.md README.md

# Commit the HF-specific files
git add Dockerfile .dockerignore README.md
git commit -m "Add Hugging Face Spaces deployment config"

# Push to HF Spaces
git push hf main
```

**Option B: Sync from GitHub (Continuous)**

1. In your Space settings, go to **"Files and versions"**
2. Click **"⚙️ Settings"** → **"Sync with GitHub"**
3. Connect your GitHub account
4. Select repository: `nikhil-7781/GraphLLM`
5. Branch: `main`
6. Enable **"Automatic sync"**

---

### **Step 3: Add Secrets (Environment Variables)**

1. Go to your Space settings
2. Click **"Settings"** → **"Repository secrets"**
3. Click **"New secret"**
4. Add required secrets:

```bash
# Required
GEMINI_API_KEY=AIzaSyCCWyinHGS3nL_urDnRLRd1xD6BWcdVqds

# Optional (with defaults)
GEMINI_MODEL=gemini-2.5-flash
LLM_TEMPERATURE=0.0
ENVIRONMENT=production
LOG_LEVEL=INFO
```

**Important:** Mark `GEMINI_API_KEY` as **Secret** (not visible in logs)

---

### **Step 4: Deploy!**

After pushing files or syncing from GitHub, HF Spaces will:

1. **Build Docker image** (~8-12 minutes first time)
2. **Start container**
3. **Health check** the app
4. **Assign public URL**: `https://huggingface.co/spaces/YOUR_USERNAME/graphllm`

**Watch the build logs:**
- Go to your Space
- Click **"Logs"** tab
- Watch for:
  ```
  Building Docker image...
  Installing dependencies...
  Starting application...
  ✓ App running on http://0.0.0.0:7860
  ```

---

## File Structure for HF Spaces

Your repository should have:

```
GraphLLM/
├── Dockerfile              # Docker config (port 7860)
├── .dockerignore           # Files to exclude from image
├── README.md               # HF Spaces homepage (use README_HF.md)
├── requirements.txt        # Python dependencies
├── main.py                 # FastAPI app
├── config.py               # Configuration
├── frontend/               # Frontend files
│   ├── index.html
│   ├── app.js
│   └── styles.css
└── ... (other Python files)
```

---

## Monitor Deployment

### **Build Logs:**

Watch real-time build progress:
```
Step 1/10 : FROM python:3.12-slim
Step 2/10 : WORKDIR /app
Step 3/10 : RUN apt-get update...
...
Step 10/10 : CMD ["python3", "main.py"]
Successfully built image
```

### **Runtime Logs:**

Once running, check application logs:
```
INFO: ✓ PDFProcessor initialized
INFO: ✓ EmbeddingService initialized
INFO: ✓ GraphStore initialized
INFO: ✓ LLMService initialized
INFO: ✓ RAGAgent initialized
INFO: Uvicorn running on http://0.0.0.0:7860
```

---

## Access Your App

1. **Public URL:**
   ```
   https://huggingface.co/spaces/YOUR_USERNAME/graphllm
   ```

2. **Embedded iframe:** Can be embedded in other sites

3. **API Endpoints:**
   ```bash
   curl https://YOUR_USERNAME-graphllm.hf.space/
   # Should return: {"message":"GraphLLM API is running"}
   ```

---

## Testing Your Deployment

```bash
# Set your Space URL
HF_URL="https://YOUR_USERNAME-graphllm.hf.space"

# Test health
curl $HF_URL/

# Upload PDF
curl -X POST "$HF_URL/upload" \
  -F "file=@test.pdf"

# Get graph
curl "$HF_URL/graph?pdf_id=YOUR_PDF_ID"

# Chat
curl -X POST "$HF_URL/chat" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is this document about?",
    "pdf_id": "YOUR_PDF_ID"
  }'
```

---

## Updating Your App

### **Automatic (GitHub Sync):**

If you enabled GitHub sync:

```bash
# Make changes locally
git add .
git commit -m "Update feature"
git push origin main

# HF Spaces automatically:
# ✓ Detects push
# ✓ Rebuilds Docker image
# ✓ Deploys new version
```

### **Manual:**

```bash
# Push directly to HF Spaces
git push hf main
```

---

## Configuration

### **Change Hardware:**

1. Go to Space **Settings** → **"Space hardware"**
2. Options:
   - **CPU basic** (free) - 2 vCPU, 16GB RAM
   - **CPU upgrade** ($0.03/hour) - 8 vCPU, 32GB RAM
   - **T4 small GPU** ($0.60/hour) - if you need GPU
   - **A10G GPU** ($3.15/hour) - for heavy ML

For GraphLLM demo: **CPU basic** (free) is sufficient!

### **Persistent Storage:**

HF Spaces provides `/data` for persistence:

Your app already uses `/app/data` which persists across deploys.

### **Custom Domain:**

1. Go to **Settings** → **"Domains"**
2. Add your custom domain
3. Update DNS records (instructions provided)
4. HF handles SSL automatically

---

## Features

### **Secrets Management:**

All secrets are encrypted and injected at runtime. Never exposed in:
- Build logs
- Runtime logs
- Code inspection

### **Automatic Rebuilds:**

Space rebuilds when:
- You push new code
- You update Docker base image
- You change environment variables (that affect build)

### **Sleep Mode:**

Free Spaces sleep after 48 hours of inactivity:
- **Sleep:** Container stops, state saved
- **Wake:** First request wakes it up (~10s delay)
- **Upgrade:** To keep always-on, upgrade hardware

---

## Monitoring

### **View Logs:**

```bash
# HF Spaces Dashboard → Logs tab
# Shows:
# - Build logs (during deployment)
# - Application logs (stdout/stderr)
# - Error traces
```

### **Metrics:**

Dashboard shows:
- Request count
- Error rate
- Uptime
- Memory usage
- CPU usage

### **Restart App:**

1. Go to **Settings**
2. Click **"Restart Space"**
3. Container rebuilds and restarts

---

## Troubleshooting

### **Build Fails**

**Issue:** Docker build timeout or errors

**Solution:**
```bash
# Check Dockerfile syntax
docker build -t graphllm .

# Common fixes:
# 1. Reduce image size (remove unnecessary files)
# 2. Check .dockerignore excludes large files
# 3. Verify requirements.txt is valid
```

### **App Doesn't Start**

**Issue:** Container starts but app crashes

**Solution:**
1. Check runtime logs in HF Spaces dashboard
2. Verify `GEMINI_API_KEY` is set in Secrets
3. Check port is 7860 in Dockerfile
4. Verify main.py reads from `API_PORT` env var

### **502 Bad Gateway**

**Issue:** Can't connect to app

**Solution:**
1. Check app is listening on `0.0.0.0:7860`
2. Verify health check path `/` returns 200
3. Check logs for startup errors
4. Try restarting the Space

### **Out of Memory**

**Issue:** Container killed (OOM)

**Solution:**
1. Upgrade hardware (Settings → Space hardware)
2. Or optimize model loading (lazy load embeddings)
3. Free tier: 16GB RAM should be sufficient

---

## Cost Breakdown

### **Free Tier (CPU Basic):**
- **RAM:** 16GB
- **CPU:** 2 vCPU
- **Storage:** 50GB
- **Cost:** **$0/month forever**
- **Sleep:** Yes (after 48h inactivity)

### **Paid Plans:**
- **CPU Upgrade:** $0.03/hour (~$22/month)
  - 8 vCPU, 32GB RAM
- **T4 GPU:** $0.60/hour (~$432/month)
  - Good for GPU-accelerated inference

**For GraphLLM interviews:** Free tier is perfect!

---

## Production Checklist

Before sharing with interviewers:

- [ ] Space deployed and running
- [ ] `GEMINI_API_KEY` secret set
- [ ] Test PDF upload works
- [ ] Test graph generation works
- [ ] Test chat functionality works
- [ ] Check logs for errors
- [ ] Test public URL accessible
- [ ] README.md displayed correctly on Space homepage

---

## Comparison: HF Spaces vs Others

| Feature | HF Spaces | Railway | Heroku | Northflank |
|---------|-----------|---------|--------|------------|
| Free Tier | ✅ Forever | ⚠️ Trial | ❌ No | ✅ Forever |
| RAM | 16GB | 512MB | - | 2GB |
| Storage | 50GB | 1GB | - | 20GB |
| ML Optimized | ✅ Yes | ❌ No | ❌ No | ⚠️ Limited |
| GPU Support | ✅ Yes | ❌ No | ❌ No | ⚠️ Limited |
| Docker | ✅ Native | ✅ Yes | ✅ Yes | ✅ Yes |
| Sleep Mode | After 48h | Immediate | - | No |

**Winner for ML apps:** Hugging Face Spaces (free + generous resources)

---

## Support & Resources

- **HF Docs:** https://huggingface.co/docs/hub/spaces
- **Docker Spaces Guide:** https://huggingface.co/docs/hub/spaces-sdks-docker
- **Community:** https://discuss.huggingface.co
- **Status:** https://status.huggingface.co

---

## Interview Talking Points

> "I've deployed GraphLLM to Hugging Face Spaces, which is optimized for ML applications. HF Spaces provides 16GB RAM and 50GB storage on the free tier, perfect for hosting ML models. The deployment uses Docker for reproducible builds, with automated CI/CD from GitHub. The platform handles SSL, secrets management, and provides built-in monitoring - all the features needed for a production ML service."

Shows: ML platform knowledge, Docker proficiency, cloud deployment, modern MLOps practices.

---

Good luck with your deployment! 🚀
