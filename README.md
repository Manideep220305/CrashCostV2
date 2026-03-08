<div align="center">

# CrashCost V2

**AI-Powered Automotive Damage Assessment & Insurance Cost Estimation**

[![Live Demo](https://img.shields.io/badge/Live%20Demo-crash--cost--v2.vercel.app-brightgreen?style=for-the-badge)](https://crash-cost-v2.vercel.app)
[![Backend](https://img.shields.io/badge/Backend-Render-blue?style=for-the-badge)](https://crashcostv2-1.onrender.com)
[![AI Engine](https://img.shields.io/badge/AI%20Engine-Hugging%20Face-yellow?style=for-the-badge)](https://huggingface.co/spaces/SaiManideep/CrashCostV2)

*Upload a photo of a damaged vehicle. Get an instant, explainable AI-powered repair cost estimate in seconds.*

</div>

---

## What is CrashCost?

CrashCost V2 is a full-stack MERN web application that uses a three-model AI pipeline to assess vehicle damage from photos and estimate repair costs in Indian Rupees. It goes beyond simple cost estimation by providing:

- **Explainable AI (XAI):** Every cost is broken down into individual damage detections with confidence scores, surface material analysis, and human-readable summaries.
- **Natural Language Explanations:** Ask the AI chatbot why a specific cost was generated in plain English. Powered by Google Gemini.
- **Deduplication Logic:** Backend filtering ensures the same damage is never counted twice, preventing inflated estimates.
- **Replacement Recommendations:** Automatically flags damage that is severe enough to require part replacement rather than repair.

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | React 19 + Vite + TailwindCSS | Multi-step claim wizard, analytics dashboard, XAI chat |
| Backend | Node.js + Express | REST API, authentication, data orchestration |
| Database | MongoDB Atlas + Mongoose | Storing claims, users, AI reports |
| AI Engine | FastAPI on Hugging Face Spaces | YOLOv11 + CLIP + CatBoost inference |
| Auth | JWT + bcryptjs | Secure stateless authentication |
| Deployment | Vercel (frontend) + Render (backend) | Cloud hosting |

---

## AI Pipeline

A submitted claim photo goes through three models in sequence:

```
Photo Upload
    │
    ▼
YOLOv11-Small (Instance Segmentation)
→ Identifies damage regions: DENT, CRACK, GLASS_SHATTER, LAMP_BROKEN, SCRATCH
→ Outputs: bounding boxes, confidence score, damage ratio
    │
    ▼
CLIP ViT-L/14 (Vision-Language Model)
→ Classifies surface material: metal, glass, plastic, rubber
→ Adds material context to the cost calculation
    │
    ▼
CatBoost Regressor (Cost Estimation)
→ Inputs: damage type, severity, surface, vehicle tier, segment, age, ratio
→ Output: Repair cost in INR for each detected damage
    │
    ▼
Backend Deduplication (IoU Filter)
→ Removes overlapping duplicate YOLO detections
→ Recalculates total from unique detections only
    │
    ▼
Google Gemini (XAI Explanations)
→ Provides natural language reasoning on demand
```

---

## Features

- **5-Step Claim Wizard** — Guided form collecting vehicle details, incident info, and photos
- **Real-time Damage Report** — Per-detection cards with severity badges, cost breakdown, and cost drivers
- **Panel Replacement Warnings** — Automatic ⚠️ badge for critical damage (ratio ≥ 40%)
- **Claims Analytics Dashboard** — History of all claims with charts and aggregate stats
- **XAI Lab Chat** — Ask Gemini questions about any specific claim in natural language
- **Secure Authentication** — JWT-based login and registration
- **Dark Mode** — Full dark/light mode support with aurora background animations
- **Actuarial Cost Ranges** — ±15% for estimates under ₹50,000, ±8% above ₹50,000

---

## Project Structure

```
InsureVision3/
├── frontend/          # React + Vite (deployed on Vercel)
│   ├── src/
│   │   ├── pages/     # dashboardPage, analyticsPage, xaiLabPage, landingPage
│   │   ├── context/   # AuthContext (JWT state management)
│   │   └── components/
│   └── vercel.json    # SPA routing config for React Router
│
├── backend/           # Node.js + Express (deployed on Render)
│   ├── controllers/   # claimController (AI pipeline + dedup), authController
│   ├── models/        # Claim.js, User.js (Mongoose schemas)
│   ├── routes/        # claimRoutes, authRoutes
│   ├── middleware/     # authMiddleware (JWT verification)
│   └── config/        # gemini.js (multi-key rotator)
│
└── huggingface-api/   # FastAPI AI Engine (deployed on Hugging Face Spaces)
    ├── main.py        # YOLO + CLIP + CatBoost inference endpoint
    └── requirements.txt
```

---

## Getting Started (Local Development)

### Prerequisites
- Node.js v18+
- MongoDB Atlas account (or local MongoDB)
- Google Gemini API key

### 1. Clone the repo
```bash
git clone https://github.com/Manideep220305/CrashCostV2.git
cd CrashCostV2
```

### 2. Setup the Backend
```bash
cd backend
npm install
```

Create `backend/.env`:
```env
MONGO_URI=your_mongodb_atlas_connection_string
JWT_SECRET=your_random_secret_string
GEMINI_API_KEY=your_gemini_api_key
FRONTEND_URL=http://localhost:5173
PORT=5000
```

```bash
npm run dev
```

### 3. Setup the Frontend
```bash
cd frontend
npm install
```

Create `frontend/.env`:
```env
VITE_API_URL=http://localhost:5000
```

```bash
npm run dev
```

Visit `http://localhost:5173` — the Vite dev proxy forwards `/api/*` requests to your local backend automatically.

---

## Deployment

| Platform | Service | Environment Variables Required |
|---|---|---|
| **Render** | Backend (Express) | `MONGO_URI`, `JWT_SECRET`, `GEMINI_API_KEY`, `FRONTEND_URL` |
| **Vercel** | Frontend (React) | `VITE_API_URL` |
| **Hugging Face** | AI Engine (FastAPI) | Built-in via Space secrets |

> **Important:** Set `FRONTEND_URL` on Render to your exact Vercel domain (e.g., `https://crash-cost-v2.vercel.app`) to enable CORS for your live site.

---

## API Reference

### Authentication
| Method | Endpoint | Body | Description |
|---|---|---|---|
| POST | `/api/auth/register` | `{ name, email, password }` | Register a new user |
| POST | `/api/auth/login` | `{ email, password }` | Login + receive JWT |

### Claims
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/segment-car` | ✅ JWT | Submit image for AI analysis |
| GET | `/api/claims?userId=X` | ❌ | Get all claims for a user |
| GET | `/api/claims/:id` | ❌ | Get a single claim by ID |
| POST | `/api/explain` | ❌ | Gemini XAI explanation for a claim |

---

## License

MIT © 2026 Manideep Kattagoni
