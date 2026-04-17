# Lab 12 — Complete Production Agent

Kết hợp TẤT CẢ những gì đã học trong 1 project hoàn chỉnh.

## Checklist Deliverable

- [x] Dockerfile (multi-stage, < 500 MB)
- [x] docker-compose.yml (agent + redis)
- [x] .dockerignore
- [x] Health check endpoint (`GET /health`)
- [x] Readiness endpoint (`GET /ready`)
- [x] API Key authentication
- [x] Rate limiting (10 req/min)
- [x] Cost guard (daily budget)
- [x] Config từ environment variables
- [x] Structured JSON logging
- [x] Graceful shutdown (SIGTERM)
- [x] Public URL ready (Railway / Render config)

---

## Cấu Trúc

```
06-lab-complete/
├── app/
│   ├── __init__.py
│   ├── main.py         # Entry point — kết hợp tất cả concepts
│   └── config.py       # 12-factor config từ env vars
├── utils/
│   └── mock_llm.py     # Mock LLM (không cần API key)
├── Dockerfile          # Multi-stage, production-ready
├── docker-compose.yml  # agent + redis stack
├── railway.toml        # Deploy Railway
├── render.yaml         # Deploy Render
├── .env.example        # Template — copy thành .env
├── .dockerignore
└── requirements.txt
```

---

## Chạy Local

```bash
# 1. Setup env
cp .env.example .env
# Sửa AGENT_API_KEY trong .env nếu muốn

# 2. Chạy với Docker Compose
docker compose up --build

# 3. Test health
curl http://localhost:8000/health

# 4. Test với API key
API_KEY=$(grep AGENT_API_KEY .env | cut -d= -f2)
curl -H "X-API-Key: $API_KEY" \
     -X POST http://localhost:8000/ask \
     -H "Content-Type: application/json" \
     -d '{"question": "What is deployment?"}'
```

---

## Deploy Render (Khuyến nghị — Free tier)

### Cách 1: New Web Service (đơn giản nhất)

1. Push repo lên GitHub
2. Render Dashboard → **New → Web Service**
3. Connect GitHub repo
4. Cấu hình:
   - **Name**: `ai-agent-production`
   - **Root Directory**: `06-lab-complete`
   - **Runtime**: Docker
   - **Region**: Singapore
5. Environment Variables → Add:
   - `ENVIRONMENT` = `production`
   - `AGENT_API_KEY` = (click "Generate" để Render tự tạo)
   - `JWT_SECRET` = (click "Generate")
   - `OPENAI_API_KEY` = (để trống — dùng mock LLM)
6. Click **Create Web Service** → Nhận public URL!

### Cách 2: Blueprint (dùng render.yaml)

1. Copy `render.yaml` ra thư mục gốc của repo:
   ```bash
   cp 06-lab-complete/render.yaml ./render.yaml
   ```
2. Push lên GitHub
3. Render Dashboard → **New → Blueprint** → Connect repo
4. Render tự đọc `render.yaml`, tạo service
5. Set `OPENAI_API_KEY` nếu cần

**Test sau khi deploy:**
```bash
# Lấy AGENT_API_KEY từ Render dashboard → Environment
curl https://your-app.onrender.com/health

curl -X POST https://your-app.onrender.com/ask \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello agent!"}'
```

---

## Deploy Railway

```bash
# Cài Railway CLI
npm i -g @railway/cli

# Login và deploy
railway login
railway init
railway variables set AGENT_API_KEY=your-secret-key
railway variables set OPENAI_API_KEY=sk-...  # bỏ qua nếu dùng mock
railway up

# Nhận public URL
railway domain
```

---

## Kiểm Tra Production Readiness

```bash
python check_production_ready.py
```

Script này kiểm tra tất cả items trong checklist và báo cáo những gì còn thiếu.

---

## Endpoints

| Method | Path | Auth | Mô tả |
|--------|------|------|-------|
| GET | `/` | ❌ | Info |
| GET | `/health` | ❌ | Liveness probe |
| GET | `/ready` | ❌ | Readiness probe |
| POST | `/ask` | ✅ X-API-Key | Gửi câu hỏi cho agent |
| GET | `/metrics` | ✅ X-API-Key | Usage metrics |
| GET | `/docs` | ❌ | Swagger UI (non-prod only) |
