# Northflank Deployment Guide for GraphLLM

## 🚀 Quick Deploy (10 Minutes)

Northflank is perfect for Docker-based ML apps with generous free tier and production features.

---

## Why Northflank?

✅ **Free Tier:**
- 2 vCPUs
- 2GB RAM
- 20GB storage
- $0/month forever

✅ **Features:**
- Docker/K8s native
- Auto-scaling
- Persistent volumes
- GitHub integration
- Built-in monitoring

✅ **Perfect for ML:**
- Supports large Docker images
- No size limits
- Long timeouts
- Persistent storage

---

## Prerequisites

1. **Northflank Account** (free)
   - Sign up: https://northflank.com
   - Use GitHub OAuth (easiest)

2. **GitHub Repository**
   - Already have: https://github.com/nikhil-7781/GraphLLM

3. **Gemini API Key**
   - From: https://makersuite.google.com/app/apikey

---

## Step-by-Step Deployment

### **Step 1: Sign Up & Create Project**

1. Go to https://northflank.com
2. Click "Sign up with GitHub"
3. Create a new project: "GraphLLM"

---

### **Step 2: Connect GitHub Repository**

1. In Northflank dashboard, click **"Create Service"**
2. Select **"Combined Service"** (runs continuously)
3. **Source:**
   - Type: **Git repository**
   - Provider: **GitHub**
   - Repository: **nikhil-7781/GraphLLM**
   - Branch: **main**
4. Click **"Next"**

---

### **Step 3: Configure Build**

1. **Build Settings:**
   - Builder: **Dockerfile**
   - Dockerfile path: `Dockerfile` (auto-detected)
   - Context: `.` (root directory)

2. **Build Arguments:** (leave empty)

3. Click **"Next"**

---

### **Step 4: Configure Deployment**

1. **Resources:**
   - CPU: **0.2 vCPU** (free tier)
   - RAM: **2048 MB** (2GB)
   - Replicas: **1**

2. **Port:**
   - Container Port: **8000**
   - Protocol: **HTTP**
   - Public: **✓ Yes**

3. **Health Check:**
   - Path: `/`
   - Initial Delay: **60s**
   - Timeout: **10s**

4. Click **"Next"**

---

### **Step 5: Add Environment Variables**

Click **"Add Environment Variable"** and add these:

```bash
# Required
GEMINI_API_KEY=AIzaSyCCWyinHGS3nL_urDnRLRd1xD6BWcdVqds
PORT=8000
ENVIRONMENT=production

# Optional
LOG_LEVEL=INFO
LLM_TEMPERATURE=0.0
GEMINI_MODEL=gemini-2.5-flash
```

**Security Tip:** Mark `GEMINI_API_KEY` as **Secret** (click the lock icon)

Click **"Next"**

---

### **Step 6: Add Persistent Storage (Optional)**

For persistent data storage:

1. Click **"Add Volume"**
2. **Mount Path:** `/app/data`
3. **Size:** **5GB** (free tier)
4. **Type:** **Persistent Volume**

This preserves graphs/embeddings across deployments.

Click **"Next"**

---

### **Step 7: Deploy!**

1. Review all settings
2. Click **"Create Service"**

Northflank will now:
1. Clone your GitHub repo
2. Build Docker image (~5-8 minutes)
3. Deploy and start the container
4. Assign a public URL

---

## Monitor Deployment

### **Build Logs:**

1. Go to your service
2. Click **"Builds"** tab
3. Watch real-time logs:

```
✓ Cloning repository...
✓ Building Docker image...
  - Installing system dependencies
  - Installing Python packages
  - Copying application code
✓ Build complete!
✓ Pushing to registry...
```

**Build time:** 5-8 minutes (first time)

---

### **Runtime Logs:**

1. Click **"Logs"** tab
2. Watch application start:

```
INFO: ✓ PDFProcessor initialized
INFO: ✓ EmbeddingService initialized
INFO: ✓ GraphStore initialized
INFO: ✓ LLMService initialized
INFO: ✓ RAGAgent initialized
INFO: Uvicorn running on http://0.0.0.0:8000
```

---

## Get Your App URL

1. Go to **"Networking"** tab
2. Copy the **Public URL:**
   ```
   https://graphllm-XXXXX.northflank.app
   ```

3. Test it:
   ```bash
   curl https://your-url.northflank.app/
   # Should return: {"message":"GraphLLM API is running"}
   ```

---

## Auto-Deploy on Git Push

Northflank automatically redeploys when you push to GitHub:

```bash
# Make changes locally
git add .
git commit -m "Update feature X"
git push origin main

# Northflank automatically:
# ✓ Detects push
# ✓ Rebuilds Docker image
# ✓ Deploys new version (zero-downtime)
```

**Rebuild time:** 3-5 minutes (after first deploy)

---

## Features & Monitoring

### **Metrics:**

Go to **"Metrics"** tab to see:
- CPU usage
- Memory usage
- Network traffic
- Request count
- Response times

### **Logs:**

- **Real-time logs** with filtering
- **Log persistence** (last 7 days)
- **Search & download** logs

### **Scaling:**

To scale up (if needed):
1. Go to **"Resources"** tab
2. Increase CPU/RAM:
   - Free tier: 0.2 vCPU, 2GB RAM
   - Paid: Up to 16 vCPU, 64GB RAM
3. Click **"Save"**

Auto-restart happens with no downtime.

---

## Troubleshooting

### **Build Fails**

**Issue:** Build timeout or errors

**Solution:**
1. Check build logs for specific error
2. Common fixes:
   - Increase build timeout (Settings → Build timeout → 20 min)
   - Check Dockerfile syntax
   - Verify requirements.txt

### **App Crashes on Start**

**Issue:** Container restarts repeatedly

**Solution:**
1. Check runtime logs
2. Verify environment variables set
3. Check `GEMINI_API_KEY` is correct
4. Increase memory to 2GB

### **Out of Memory**

**Issue:** Container killed (OOMKilled)

**Solution:**
1. Increase RAM in Resources tab
2. Free tier: max 2GB
3. Or upgrade to paid plan

### **502 Bad Gateway**

**Issue:** Can't connect to app

**Solution:**
1. Check app is listening on port 8000
2. Verify health check path is `/`
3. Check logs for startup errors

---

## Cost Breakdown

### **Free Tier:**
- **Resources:** 2GB RAM, 0.2 vCPU
- **Storage:** 20GB
- **Bandwidth:** 100GB/month
- **Cost:** **$0/month forever**

### **Paid Plans:**
- **Starter:** $20/month
  - 4GB RAM, 1 vCPU
  - 100GB storage
- **Pro:** $100/month
  - 16GB RAM, 4 vCPU
  - 500GB storage

**For GraphLLM:** Free tier is sufficient for demos/interviews!

---

## Production Checklist

Before sharing with interviewers:

- [ ] Service deployed successfully
- [ ] Environment variables set (especially `GEMINI_API_KEY`)
- [ ] Public URL accessible
- [ ] Test PDF upload works
- [ ] Test graph generation works
- [ ] Test chat functionality works
- [ ] Check logs for errors
- [ ] Persistent volume configured (optional)
- [ ] Custom domain configured (optional)

---

## Custom Domain (Optional)

Northflank supports custom domains:

1. Go to **"Networking"** → **"Domains"**
2. Click **"Add Domain"**
3. Enter your domain: `graphllm.yourdomain.com`
4. Add DNS records (Northflank provides instructions)
5. SSL certificate auto-generated

---

## Comparison: Northflank vs Others

| Feature | Northflank | Railway | Render |
|---------|-----------|---------|--------|
| Free Tier | ✅ Forever | ⚠️ $5 trial | ✅ Forever |
| RAM | 2GB | 512MB | 512MB |
| Storage | 20GB | 1GB | 1GB |
| Build Timeout | 20min | 10min | 15min |
| Docker Support | ✅ Native | ✅ Yes | ✅ Yes |
| Auto-Deploy | ✅ Yes | ✅ Yes | ✅ Yes |
| K8s Features | ✅ Yes | ❌ No | ❌ No |
| Persistent Volumes | ✅ Yes | ✅ Yes | ⚠️ Limited |

---

## Support & Resources

- **Docs:** https://northflank.com/docs
- **Discord:** https://discord.gg/northflank
- **Status:** https://status.northflank.com
- **Support:** support@northflank.com

---

## Quick Commands Reference

### Via Dashboard:
- **Deploy:** Create Service → Connect GitHub → Configure → Deploy
- **Logs:** Service → Logs tab
- **Restart:** Service → ... → Restart
- **Scale:** Service → Resources → Update

### Via CLI (optional):
```bash
# Install Northflank CLI
npm install -g @northflank/cli

# Login
northflank login

# Deploy
northflank deploy
```

---

## Interview Talking Points

> "I've deployed GraphLLM to Northflank, which provides Kubernetes-native container orchestration with persistent volumes for data storage. The deployment uses Docker with automated CI/CD from GitHub, so every push triggers a new build. Northflank offers 2GB of RAM and 20GB storage on the free tier, which is perfect for ML applications like this. The platform provides built-in monitoring, logging, and auto-scaling capabilities."

Shows: DevOps knowledge, container orchestration, CI/CD, cloud deployment.

---

Good luck with your deployment! 🚀
