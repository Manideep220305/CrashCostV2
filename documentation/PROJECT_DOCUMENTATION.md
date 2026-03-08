# CrashCost V2 — Exhaustive Project Documentation

> This document covers every single file, function, API call, route, data model, middleware decision, and flow in the CrashCost V2 system. It is written to be understood by someone who has never seen this codebase before and also serves as a complete reference for interviews.

---

## Table of Contents

1. [System Overview & Big Picture](#1-system-overview--big-picture)
2. [Repository Structure](#2-repository-structure)
3. [Technology Decisions & Why](#3-technology-decisions--why)
4. [Complete Data Flow (Request Lifecycle)](#4-complete-data-flow-request-lifecycle)
5. [Frontend Architecture](#5-frontend-architecture)
6. [Backend Architecture](#6-backend-architecture)
7. [AI Pipeline Deep Dive](#7-ai-pipeline-deep-dive)
8. [All API Routes Reference](#8-all-api-routes-reference)
9. [Authentication & Security](#9-authentication--security)
10. [Deployment Architecture](#10-deployment-architecture)
11. [Environment Variables](#11-environment-variables)
12. [Known Gaps & Future Work](#12-known-gaps--future-work)

---

## 1. System Overview & Big Picture

CrashCost V2 is a full-stack web application for AI-powered automobile insurance damage assessment. A user uploads a photo of a damaged car and receives:

- A precise repair cost estimate in Indian Rupees (₹)
- A list of individual damage detections (DENT, CRACK, GLASS_SHATTER, etc.)
- Severity classification per damage (MINOR, MODERATE, SEVERE)
- A natural-language explanation of why each cost was generated (powered by Gemini AI)
- A historical analytics dashboard of all past claims

### The Four-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 1: USER BROWSER                            │
│   React 19 + Vite SPA deployed on Vercel (CDN)                     │
│   URL: https://crash-cost-v2.vercel.app                            │
└─────────────────────────┬───────────────────────────────────────────┘
                          │  HTTPS REST API calls
                          │  (fetch / axios with JWT Bearer token)
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 2: BACKEND API SERVER                      │
│   Node.js 22 + Express deployed on Render                          │
│   URL: https://crashcostv2-1.onrender.com                          │
│   Responsibilities:                                                 │
│   - Authentication (JWT + bcrypt)                                  │
│   - Rate limiting (15 req/min on AI routes)                        │
│   - CORS enforcement (only Vercel domain allowed)                  │
│   - File upload handling (Multer → uploads/ folder)               │
│   - Deduplication of YOLO detections                              │
│   - Orchestrating calls to HF and Gemini                          │
│   - Saving/reading claims from MongoDB                             │
└──────────────┬────────────────────────────────┬────────────────────┘
               │  HTTP POST (form-data)         │  MongoDB Wire Protocol
               │  (image + vehicle metadata)   │  (Mongoose)
               ▼                               ▼
┌──────────────────────────┐   ┌───────────────────────────────────┐
│   LAYER 3: AI ENGINE     │   │   LAYER 4: DATABASE               │
│   FastAPI on HF Spaces   │   │   MongoDB Atlas (Cloud)           │
│   URL: saimanideep-      │   │   - users collection              │
│   crashcostv2.hf.space   │   │   - claims collection             │
│   Models:                │   │                                   │
│   - YOLOv11-Small        │   │   Everything is stored here so    │
│   - CLIP ViT-L/14        │   │   the Analytics page and XAI Lab  │
│   - CatBoost V4          │   │   can access historical data.     │
│   Also: Gemini 2.5 Flash │   │                                   │
│   (via Google AI SDK)    │   │                                   │
└──────────────────────────┘   └───────────────────────────────────┘
```

---

## 2. Repository Structure

```
InsureVision3/                         ← Root of GitHub repo
│
├── README.md                          ← Professional readme with badges
├── .gitignore                         ← Root-level ignores (*.pt, *.cbm, *.csv)
│
├── frontend/                          ← React SPA (Vercel deploys from here)
│   ├── .gitignore                     ← Ignores node_modules, dist, .env
│   ├── .env                           ← VITE_API_URL=http://localhost:5000 (local dev)
│   ├── vercel.json                    ← Vercel SPA routing: rewrites /* → index.html
│   ├── vite.config.js                 ← Vite bundler + dev-only /api proxy
│   ├── tailwind.config.js             ← Tailwind with custom animation keyframes
│   ├── postcss.config.js              ← PostCSS for Tailwind processing
│   ├── index.html                     ← HTML shell; React mounts into #root
│   ├── package.json                   ← dependencies (React, Framer, Recharts, etc.)
│   └── src/
│       ├── main.jsx                   ← React entry point: renders <App /> into #root
│       ├── App.jsx                    ← Route definitions, wraps ThemeProvider + AuthProvider
│       ├── index.css                  ← Global CSS, Tailwind imports, custom animations
│       │
│       ├── context/
│       │   ├── AuthContext.jsx        ← Global JWT auth state (login, register, logout)
│       │   └── ThemeContext.jsx       ← Dark/light mode toggle, stores in localStorage
│       │
│       ├── pages/
│       │   ├── landingPage.jsx        ← Public homepage with car damage simulator
│       │   ├── dashboardPage.jsx      ← 5-step claim wizard + AI report display
│       │   ├── analyticsPage.jsx      ← Claims history, charts, inline AI chat
│       │   ├── xaiLabPage.jsx         ← Dedicated Gemini XAI chat interface
│       │   └── insuranceGuidePage.jsx ← Insurance 101 educational page
│       │
│       ├── components/
│       │   ├── Sidebar.jsx            ← Left nav for dashboard/analytics/XAI pages
│       │   ├── Navbar.jsx             ← Top nav for landing page
│       │   ├── AuthModal.jsx          ← Radix Dialog containing login/register form
│       │   ├── AuroraBackground.jsx   ← Animated 3D background on landing page
│       │   └── ui/glowing-effect.jsx  ← Reusable glowing border component
│       │
│       ├── services/
│       │   └── api.js                 ← Axios wrapper helper (used by legacy code)
│       └── controllers/
│           └── useApi.js              ← Custom hook around api.js
│
├── backend/                           ← Express API (Render deploys from here)
│   ├── .gitignore                     ← Ignores node_modules, uploads/, .env
│   ├── package.json                   ← Node deps (express, mongoose, multer, etc.)
│   ├── server.js                      ← App bootstrap: middleware, routes, DB connect
│   │
│   ├── config/
│   │   └── gemini.js                  ← Multi-key Gemini rotator
│   │
│   ├── models/
│   │   ├── User.js                    ← Mongoose schema for registered users
│   │   └── Claim.js                   ← Mongoose schema for a full damage claim
│   │
│   ├── middleware/
│   │   └── authMiddleware.js          ← JWT verify → attaches req.user
│   │
│   ├── controllers/
│   │   ├── authController.js          ← register/login logic, JWT generation
│   │   └── claimController.js         ← analyzeClaim, explainClaim, getAllClaims, getClaimById
│   │
│   └── routes/
│       ├── authRoutes.js              ← POST /api/auth/register, POST /api/auth/login
│       └── claimRoutes.js             ← POST /api/segment-car, POST /api/explain, GET /api/claims
│
└── huggingface-api/                   ← FastAPI AI engine (deployed on HF Spaces separately)
    ├── main.py                        ← YOLO + CLIP + CatBoost inference
    ├── best.pt                        ← YOLOv11 model weights (60MB, NOT in git)
    ├── crashcost_pricing_model.cbm    ← CatBoost model (4MB, NOT in git)
    └── requirements.txt
```

---

## 3. Technology Decisions & Why

### Frontend: React 19 + Vite
- **React 19** for its component model and hooks. All UI is built as composable components.
- **Vite** instead of Create React App because it's significantly faster in dev mode (HMR in <100ms) and produces smaller production bundles.
- **Tailwind CSS** for styling. Every class is a CSS utility. No separate stylesheet files needed. Dark mode is implemented via Tailwind's `dark:` prefix and the `class="dark"` on the `<html>` tag.
- **Framer Motion** for every animation (aurora waves, page transitions, card fade-ins). All `<motion.div>` components.
- **Recharts** for the bar chart and pie chart on the analytics page.
- **Lucide React** for all SVG icons throughout the UI.
- **React Router v7** for client-side routing between pages. Uses `BrowserRouter` so URLs look like `/dashboard` instead of `/#/dashboard`.
- **@radix-ui/react-dialog** for the AuthModal. Radix provides accessible, unstyled headless UI primitives.

### Backend: Node.js + Express
- **Express** because it's the standard for Node.js APIs. Extremely lightweight.
- **Mongoose** (ODM over MongoDB driver) for schema validation, easy querying, and model-based code organization.
- **Multer** for handling `multipart/form-data` image uploads. It saves uploaded files to `uploads/` temporarily, then the controller reads them, forwards them to HF, and deletes them.
- **Axios** (not `fetch`) for calling the Hugging Face API because Axios handles binary/buffer data and `form-data` better than Node's native `fetch` at the time this was written.
- **bcryptjs** for hashing passwords. Uses the `bcrypt` algorithm with salt rounds. Passwords are NEVER stored in plaintext.
- **jsonwebtoken** for creating and verifying JWTs. Stateless — no session store needed.
- **express-rate-limit** to protect expensive routes from abuse.
- **cors** middleware for Cross-Origin Resource Sharing. Restricted to specific allowed origins.

### Database: MongoDB Atlas
- Document-oriented NoSQL. Each claim is one JSON document. Perfect for storing the nested AI report with variable numbers of detections.
- Atlas is the cloud-hosted version. Free tier (512MB) is more than enough for this app.
- Mongoose adds a schema layer so your documents have a defined structure.

### AI: Hugging Face Spaces (FastAPI + Python)
- HF Spaces provides free GPU compute for the AI models.
- FastAPI is used because it's the standard Python async web framework for ML serving.
- The three models are loaded once at server startup (not per request), so inference is fast.

### XAI: Google Gemini 2.5 Flash
- Called from the Express backend (not directly from the browser). This keeps the API key secret.
- Multi-key rotator (`config/gemini.js`) multiplies the free daily rate limit by 3.

---

## 4. Complete Data Flow (Request Lifecycle)

### 4.1 Registration / Login Flow

```
Browser                          Express Backend              MongoDB Atlas
  │                                    │                           │
  │  POST /api/auth/register            │                           │
  │  body: { name, email, password }   │                           │
  │ ──────────────────────────────────►│                           │
  │                                    │  authController.js        │
  │                                    │  1. Validates fields       │
  │                                    │  2. Checks if email exists │
  │                                    │ ─────────────────────────►│
  │                                    │  User.findOne({ email })  │
  │                                    │ ◄─────────────────────────│
  │                                    │  3. bcrypt.hash(password) │
  │                                    │  4. User.create({...})    │
  │                                    │ ─────────────────────────►│
  │                                    │  saves User document      │
  │                                    │ ◄─────────────────────────│
  │                                    │  5. jwt.sign({id}, secret)│
  │  200 { _id, name, email, token }   │                           │
  │ ◄──────────────────────────────────│                           │
  │                                    │                           │
  │  AuthContext stores:               │                           │
  │  - user object in React state      │                           │
  │  - token in localStorage           │                           │
  │  - sets axios default header:      │                           │
  │    Authorization: Bearer <token>   │                           │
```

**JWT Payload Structure:**
```json
{
  "id": "64a7c8...<MongoDB ObjectId>",
  "iat": 1709901234,
  "exp": 1712493234
}
```
The token is signed with `process.env.JWT_SECRET` and expires in 30 days.

---

### 4.2 Claim Submission Flow (The Core Pipeline)

```
Browser (dashboardPage.jsx)      Express Backend              Hugging Face Space      MongoDB
         │                            │                              │                  │
         │  User fills 5-step form    │                              │                  │
         │  and clicks "Analyze"      │                              │                  │
         │                            │                              │                  │
         │  handleAnalyze() builds    │                              │                  │
         │  FormData:                 │                              │                  │
         │    - image: rawFile        │                              │                  │
         │    - vehicleDetails: JSON  │                              │                  │
         │    - incidentDetails: JSON │                              │                  │
         │                            │                              │                  │
         │  POST /api/segment-car     │                              │                  │
         │  Header: Authorization:    │                              │                  │
         │    Bearer <token>          │                              │                  │
         │ ──────────────────────────►│                              │                  │
         │                            │  1. authMiddleware runs      │                  │
         │                            │     jwt.verify(token)        │                  │
         │                            │     User.findById(id)        │                  │
         │                            │     → req.user = user        │                  │
         │                            │                              │                  │
         │                            │  2. multer runs              │                  │
         │                            │     saves image to           │                  │
         │                            │     uploads/xxxxx.jpg        │                  │
         │                            │                              │                  │
         │                            │  3. analyzeClaim() runs      │                  │
         │                            │     reads image buffer       │                  │
         │                            │     builds new FormData:     │                  │
         │                            │       image buffer           │                  │
         │                            │       brand, tier, segment   │                  │
         │                            │       location, age          │                  │
         │                            │                              │                  │
         │                            │  POST /api/v1/audit          │                  │
         │                            │  Timeout: 120 seconds        │                  │
         │                            │ ────────────────────────────►│                  │
         │                            │                              │  YOLOv11 runs    │
         │                            │                              │  detects damage  │
         │                            │                              │  CLIP classifies │
         │                            │                              │  surface         │
         │                            │                              │  CatBoost        │
         │                            │                              │  estimates cost  │
         │                            │ ◄────────────────────────────│                  │
         │                            │  { detections, total_estimate│                  │
         │                            │    estimate_range, context } │                  │
         │                            │                              │                  │
         │                            │  4. IoU Deduplication filter │                  │
         │                            │     removes duplicate boxes  │                  │
         │                            │     recalculates total       │                  │
         │                            │                              │                  │
         │                            │  5. Delete temp file         │                  │
         │                            │     fs.unlinkSync(...)       │                  │
         │                            │                              │                  │
         │                            │  6. Claim.save()             │                  │
         │                            │ ─────────────────────────────────────────────►  │
         │                            │     { userId, vehicleDetails │                  │
         │                            │       incidentDetails,       │                  │
         │                            │       aiReport }             │                  │
         │                            │ ◄─────────────────────────────────────────────  │
         │                            │  savedClaim._id returned     │                  │
         │                            │                              │                  │
         │  200 { success, claimId,   │                              │                  │
         │        report (aiReport) } │                              │                  │
         │ ◄──────────────────────────│                              │                  │
         │                            │                              │                  │
         │  setReportResult(report)   │                              │                  │
         │  setCurrentClaimId(id)     │                              │                  │
         │  setCurrentStep(5)         │                              │                  │
         │  → AssessmentReport shown  │                              │                  │
```

---

### 4.3 Analytics Page Flow

```
Browser (analyticsPage.jsx)      Express Backend              MongoDB
         │                            │                           │
         │  Component mounts          │                           │
         │  useEffect fires           │                           │
         │                            │                           │
         │  GET /api/claims?          │                           │
         │      userId=<user._id>     │                           │
         │ ──────────────────────────►│                           │
         │                            │  getAllClaims()            │
         │                            │  Claim.find({ userId })   │
         │                            │  .sort({ createdAt: -1 }) │
         │                            │ ─────────────────────────►│
         │                            │ ◄─────────────────────────│
         │  200 [ ...claims array ]   │                           │
         │ ◄──────────────────────────│                           │
         │                            │                           │
         │  React computes:           │                           │
         │  - totalClaims             │                           │
         │  - allDetections (flat)    │                           │
         │  - avgConfidence           │                           │
         │  - totalCost               │                           │
         │  - chart data              │                           │
         │  All rendered into UI      │                           │
         │                            │                           │
         │--- When user clicks "Ask AI" on a claim ------------ │
         │                            │                           │
         │  POST /api/explain         │                           │
         │  body: { claimId, message }│                           │
         │ ──────────────────────────►│  explainClaim()           │
         │                            │  Claim.findById(claimId) │
         │                            │ ─────────────────────────►│
         │                            │ ◄─────────────────────────│
         │                            │  builds prompt with full  │
         │                            │  claim context            │
         │                            │  getNextGeminiModel()     │
         │                            │  model.generateContent()  │
         │  200 { answer: "..." }     │                           │
         │ ◄──────────────────────────│                           │
         │  Renders Gemini text in    │                           │
         │  inline chat box           │                           │
```

---

### 4.4 XAI Lab Flow

```
Browser (xaiLabPage.jsx)         Express Backend              MongoDB
         │                            │                           │
         │  Page loads with           │                           │
         │  ?claimId=<id> in URL      │                           │
         │                            │                           │
         │  GET /api/claims?          │                           │
         │      userId=<user._id>     │                           │
         │ ──────────────────────────►│  getAllClaims({ userId }) │
         │  200 [...claims]           │Claim.find().sort()        │
         │ ◄──────────────────────────│                           │
         │                            │                           │
         │  selectedClaimId set       │                           │
         │  from URL param if present │                           │
         │                            │                           │
         │--- User types question and presses send ------------- │
         │                            │                           │
         │  POST /api/explain         │                           │
         │  { claimId, message: "Why  │                           │
         │    is this SEVERE?" }      │                           │
         │ ──────────────────────────►│                           │
         │                            │  1. Claim.findById()      │
         │                            │ ─────────────────────────►│
         │                            │ ◄─────────────────────────│
         │                            │                           │
         │                            │  2. Build prompt:         │
         │                            │  "You are XAI module..."  │
         │                            │  Vehicle: [brand, model]  │
         │                            │  Detections: [list]       │
         │                            │  User question: [message] │
         │                            │                           │
         │                            │  3. getNextGeminiModel()  │
         │                            │     (round-robin keys)    │
         │                            │                           │
         │                            │  4. model.generateContent │
         │                            │     (prompt)              │
         │                            │     → Google AI API call  │
         │                            │                           │
         │  200 { answer: "The        │                           │
         │    SEVERE rating means..." }│                          │
         │ ◄──────────────────────────│                           │
         │  Renders in chat history   │                           │
```

---

## 5. Frontend Architecture

### 5.1 `main.jsx` — Entry Point
```javascript
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```
This is the absolute entry point. Vite builds this file. `#root` is a `<div>` in `index.html`. StrictMode renders components twice in development to catch bugs.

---

### 5.2 `App.jsx` — Route Configuration
```javascript
function App() {
  return (
    <ThemeProvider>
      <AuthProvider>
        <Router>
          <Routes>
            <Route path="/"              element={<LandingPage />} />
            <Route path="/dashboard"     element={<DashboardPage />} />
            <Route path="/analytics"     element={<AnalyticsPage />} />
            <Route path="/xai-lab"       element={<XaiLabPage />} />
            <Route path="/insurance-101" element={<InsuranceGuidePage />} />
            <Route path="*"              element={<LandingPage />} />
          </Routes>
        </Router>
      </AuthProvider>
    </ThemeProvider>
  );
}
```

**Provider wrapping order matters:**
- `ThemeProvider` is outermost — every component can read the theme
- `AuthProvider` wraps everything — every component can call `useAuth()`
- `Router` wraps everything — every component can use `useNavigate()`, `useLocation()`

The `*` catch-all route redirects unknown URLs back to the landing page.

---

### 5.3 `AuthContext.jsx` — Global Auth State

**What it manages:**
- `user` — object `{ _id, name, email, token }` or `null`
- `loading` — boolean, true while an API call is in progress
- `error` — string, last error message or null

**On mount (cold start):**
```javascript
useEffect(() => {
  const storedUser = localStorage.getItem('user');
  const storedToken = localStorage.getItem('token');
  if (storedUser && storedToken) {
    setUser(JSON.parse(storedUser));
    setToken(storedToken);
    // So axios calls made by this component automatically include the JWT
    axios.defaults.headers.common['Authorization'] = `Bearer ${storedToken}`;
  }
}, []);
```
This is why you stay logged in even if you refresh the page. The token survives in `localStorage`.

**`login(email, password)` function:**
```javascript
const API_URL = import.meta.env.VITE_API_URL || '';
const { data } = await axios.post(`${API_URL}/api/auth/login`, { email, password });
setUser(data);
setToken(data.token);
localStorage.setItem('user', JSON.stringify(data));
localStorage.setItem('token', data.token);
axios.defaults.headers.common['Authorization'] = `Bearer ${data.token}`;
```

**`logout()` function:**
```javascript
setUser(null);
setToken(null);
localStorage.removeItem('user');
localStorage.removeItem('token');
delete axios.defaults.headers.common['Authorization'];
```

---

### 5.4 `ThemeContext.jsx` — Dark/Light Mode

```javascript
const [theme, setTheme] = useState(() => {
  return localStorage.getItem('theme') || 'dark';
});

useEffect(() => {
  document.documentElement.classList.toggle('dark', theme === 'dark');
  localStorage.setItem('theme', theme);
}, [theme]);
```

When `theme === 'dark'`, the class `"dark"` is added to `<html>`. Tailwind's `dark:` prefix classes then activate. This is the standard Tailwind "class strategy" dark mode.

---

### 5.5 `dashboardPage.jsx` — The Core Claim Wizard

#### `useMockStore()` custom hook
This hook is defined inside `dashboardPage.jsx` and acts as a local mini-store for the entire wizard. It manages all wizard state across all 5 steps:

```javascript
const useMockStore = () => {
  const [currentStep, setCurrentStep] = useState(1);
  const [claimData, setClaimData] = useState({
    vehicleDetails: { vin: '', year: '', make: '', model: '', car_model_val: '',
                      car_age: '', brand: '', tier: '', segment: '', damageLocation: '' },
    incidentDetails: { date: '', description: '' },
    uploadedImages: []
  });
  const [reportResult, setReportResult] = useState(null);
  const [currentClaimId, setCurrentClaimId] = useState(null);

  const resetClaim = () => {
    setCurrentStep(1);
    setReportResult(null);
    setCurrentClaimId(null);
    setClaimData({ /* reset to initial */ });
  };

  return { currentStep, setCurrentStep, claimData, updateVehicleDetails,
           updateIncidentDetails, addImages, removeImage, reportResult,
           setReportResult, currentClaimId, setCurrentClaimId, resetClaim };
};
```

**Why a custom hook?** It keeps the main `DashboardPage` component clean. All state is colocated in one place, and any step component gets what it needs by calling `store.updateVehicleDetails(...)`, etc.

#### The 5 Steps Array
```javascript
const steps = [
  { id: 1, label: 'Vehicle',  component: VehicleDetailsStep },
  { id: 2, label: 'Incident', component: IncidentDetailsStep },
  { id: 3, label: 'Visuals',  component: ImageUploadStep },
  { id: 4, label: 'Review',   component: ReviewStep },
  { id: 5, label: 'Analyze',  component: (props) =>
    props.store.reportResult
      ? <AssessmentReport data={{...}} onReset={() => store.resetClaim()} ... />
      : <SubmitStep />
  },
];
```
Step 5 is dynamic — it renders `<SubmitStep />` initially, then switches to `<AssessmentReport>` once `reportResult` is set.

#### `handleAnalyze()` — The API Call
```javascript
const handleAnalyze = async () => {
  setIsProcessing(true);

  const formData = new FormData();
  formData.append('image', claimData.uploadedImages[0].rawFile);
  formData.append('vehicleDetails', JSON.stringify(claimData.vehicleDetails));
  formData.append('incidentDetails', JSON.stringify(claimData.incidentDetails));

  const token = localStorage.getItem('token');
  const API_URL = import.meta.env.VITE_API_URL || '';

  const response = await fetch(`${API_URL}/api/segment-car`, {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${token}` },
    // NO Content-Type header — browser sets multipart/form-data with boundary automatically
    body: formData,
  });

  const data = await response.json();
  if (data.success) {
    setReportResult(data.report);
    setCurrentClaimId(data.claimId);
    setCurrentStep(5);
  }
};
```

**Critical detail:** When submitting `FormData`, do NOT manually set `Content-Type`. The browser needs to set it automatically with the correct `boundary` parameter (e.g., `multipart/form-data; boundary=----WebKitFormBoundaryXYZ`). Setting it manually breaks the multipart parsing.

#### `AssessmentReport` Component
Renders when `reportResult` is received. Key sections:
- **Image:** The uploaded image src (from the preview URL stored in `uploadedImages`)
- **Summary stats:** Total detections, average confidence, location
- **Per-detection cards:** For each `det` in `report.detections`:
  - Severity badge (color-coded by MINOR/MODERATE/SEVERE)
  - Label (DENT, CRACK, etc.)
  - `⚠️ Panel Replacement Recommended` badge when `det.ratio >= 0.40` AND label is in `['CRACK', 'GLASS SHATTER', 'LAMP_BROKEN', 'DENT']`
  - Price in INR
  - Summary text
  - Confidence, surface, ratio metadata
  - Cost driver pills (SHAP feature names)
- **Action buttons:** "Explain Logic" → navigates to `/xai-lab?claimId=<id>`, "New Claim" → calls `store.resetClaim()`

---

### 5.6 `analyticsPage.jsx` — Claims History & Charts

**Data fetching:**
```javascript
useEffect(() => {
  const fetchClaims = async () => {
    const API_URL = import.meta.env.VITE_API_URL || '';
    const res = await fetch(`${API_URL}/api/claims?userId=${user._id || user.id}`);
    const data = await res.json();
    setClaims(Array.isArray(data) ? data : []);
  };
  fetchClaims();
}, [user]);
```

**Computed statistics (derived state, not stored in DB):**
```javascript
const totalClaims = claims.length;
const allDetections = claims.flatMap(c => c.aiReport?.detections || []);
const avgConfidence = allDetections.length
  ? allDetections.reduce((s, d) => s + d.confidence, 0) / allDetections.length
  : 0;
const totalCost = claims.reduce((s, c) => s + (c.aiReport?.total_estimate || 0), 0);
```

**Charts:** Uses Recharts `<BarChart>` for damage type distribution, `<PieChart>` for severity. Both are built directly from `claims` array on every render.

---

### 5.7 `xaiLabPage.jsx` — XAI Chat Interface

**URL Parameter handling:**
```javascript
const [searchParams] = useSearchParams();
const initialClaimId = searchParams.get('claimId');
const [selectedClaimId, setSelectedClaimId] = useState(initialClaimId || '');
```
When navigated from `AssessmentReport`'s "Explain Logic" button with `?claimId=xyz`, the claim is pre-selected.

**Chat state:**
```javascript
const [chat, setChat] = useState([
  { role: 'ai', text: 'Hello! I am the InsureVision XAI module...' }
]);
```
Each message is `{ role: 'user' | 'ai', text: string }`. New messages are appended.

**Sending a question:**
```javascript
const API_URL = import.meta.env.VITE_API_URL || '';
const response = await fetch(`${API_URL}/api/explain`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ claimId: selectedClaimId, message: query }),
});
const data = await response.json();
setChat(prev => [...prev, { role: 'ai', text: data.answer }]);
```

---

### 5.8 `vercel.json` — SPA Routing Config

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

**Why this is needed:** Vercel hosts the compiled `dist/` folder as static files. If you navigate to `https://crash-cost-v2.vercel.app/dashboard` by hard-refreshing, Vercel looks for a file called `dashboard.html` in `dist/`. It doesn't exist, so without this rule, Vercel returns a 404.

**Why `rewrites` and not `routes`:** `routes` in Vercel intercepts ALL requests — including requests for `dist/assets/index-AbCdEfG.js`. It returns `index.html` (HTML text) instead of the JavaScript file, causing the MIME type crash. `rewrites` only kicks in when no matching static file exists on disk.

---

### 5.9 `vite.config.js` — Dev Server Configuration

```javascript
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
      },
    },
  },
})
```

**The proxy** only runs in development. When you call `fetch('/api/segment-car')` in dev, Vite intercepts it and forwards it to `http://localhost:5000/api/segment-car`. This means the browser never technically crosses origins in dev mode, so CORS is not triggered.

In production on Vercel, there is NO proxy. This is why `VITE_API_URL` must be set to the full Render URL and all `fetch()` calls must use it.

---

## 6. Backend Architecture

### 6.1 `server.js` — Bootstrap (Exact Execution Order)

When you run `npm start`, Node executes `server.js`. Here's exactly what happens line by line:

1. **`require('dotenv').config()`** — Reads `backend/.env` and populates `process.env`
2. **`const app = express()`** — Creates the Express application object
3. **`app.set('trust proxy', 1)`** — Tells Express to trust the first proxy (Render's load balancer). Without this, `req.ip` would always be the load balancer's IP, breaking rate limiting (everyone would share the same 15 req/min bucket).
4. **CORS middleware:**
   ```javascript
   const allowedOrigins = [
     process.env.FRONTEND_URL,  // 'https://crash-cost-v2.vercel.app' in production
     'http://localhost:5173',
     'http://localhost:3000'
   ].filter(Boolean);  // Remove undefined/null entries

   app.use(cors({
     origin: function(origin, callback) {
       if (!origin || allowedOrigins.includes(origin)) callback(null, true);
       else callback(new Error('Not allowed by CORS'));
     },
     credentials: true,
   }));
   ```
   The `!origin` check allows requests with no Origin header (Postman, curl, server-to-server).
5. **`app.use(express.json())`** — Parses `Content-Type: application/json` bodies
6. **`app.use(express.urlencoded({ extended: true }))`** — Parses URL-encoded form data
7. **Rate limiter:**
   ```javascript
   const aiLimiter = rateLimit({
     windowMs: 1 * 60 * 1000, // 1 minute
     max: 15,                  // 15 requests per IP per window
   });
   app.use('/api/segment-car', aiLimiter);
   app.use('/api/explain', aiLimiter);
   ```
8. **`uploads/` directory check** — Creates the folder if it doesn't exist
9. **MongoDB connection** — `mongoose.connect(process.env.MONGO_URI || 'mongodb://localhost...')`
10. **Mount routes:**
    - `app.use('/api', claimRoutes)` → handles `/api/segment-car`, `/api/explain`, `/api/claims`
    - `app.use('/api/auth', authRoutes)` → handles `/api/auth/register`, `/api/auth/login`
11. **Health check:** `app.get('/healthz', ...)` — Returns 200 "OK". Render pings this to know the server is alive.
12. **`app.listen(PORT)`** — Starts listening. Render sets `PORT` automatically.

---

### 6.2 `models/User.js` — Mongoose Schema

```javascript
const UserSchema = new mongoose.Schema({
  name:     { type: String, required: true },
  email:    { type: String, required: true, unique: true },
  password: { type: String, required: true },  // stored as bcrypt hash
}, { timestamps: true });
```

`timestamps: true` automatically adds `createdAt` and `updatedAt` fields.
`unique: true` on email tells MongoDB to create a unique index, making duplicate registration attempts fail at the DB level.

---

### 6.3 `models/Claim.js` — Mongoose Schema (Full)

```javascript
const DetectionSchema = new mongoose.Schema({
  id:               Number,
  label:            String,      // 'DENT', 'CRACK', 'GLASS_SHATTER', etc.
  confidence:       Number,      // 0.0 to 1.0
  surface_detected: String,      // 'metal', 'glass', 'plastic', 'rubber'
  severity:         String,      // 'MINOR', 'MODERATE', 'SEVERE'
  ratio:            Number,      // damage_area / total_vehicle_area
  bbox:             mongoose.Schema.Types.Mixed,  // { x1, y1, x2, y2 }
  price:            Number,      // INR cost
  drivers:          [String],    // ['Extensive Surface Area', 'Structural Check']
  summary:          String,      // "Severe dent on metal panel..."
  narrative:        String,      // Longer explanation text
}, { _id: false });

const ClaimSchema = new mongoose.Schema({
  userId:          { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  vehicleDetails:  { vin, model, year, make, brand, tier, segment,
                     damageLocation, car_model_val, car_age },
  incidentDetails: { date: Date, description: String },
  aiReport: {
    total_estimate:  Number,
    estimate_range:  [Number],      // [min, max]
    context:         { brand, tier, segment, location },
    detections:      [DetectionSchema]
  },
  status: { type: String, default: 'Auto-Assessed' },
}, { timestamps: true });
```

---

### 6.4 `middleware/authMiddleware.js` — JWT Verification

```javascript
const protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization?.startsWith('Bearer')) {
    try {
      // "Bearer <token>" → split → ["Bearer", "<token>"] → take index 1
      token = req.headers.authorization.split(' ')[1];

      // Verify signature + expiry
      const decoded = jwt.verify(token, process.env.JWT_SECRET || 'CrashCostSuperSecret123');

      // Load user from DB (minus password field)
      req.user = await User.findById(decoded.id).select('-password');

      next(); // ← Only reaches the controller if we get here
    } catch (error) {
      res.status(401).json({ message: 'Not authorized, token failed' });
    }
  }

  if (!token) {
    res.status(401).json({ message: 'Not authorized, no token' });
  }
};
```

**What `next()` does:** In Express middleware, calling `next()` passes control to the next middleware in the chain. The chain is: `protect → upload.single('image') → analyzeClaim`. So if the JWT is invalid, `next()` is never called and `analyzeClaim` never runs.

---

### 6.5 `controllers/claimController.js` — Most Important File

#### `analyzeClaim` — Step by Step

**Step A: Parse request body**
```javascript
const vehicleDetails = JSON.parse(req.body.vehicleDetails || '{}');
const incidentDetails = JSON.parse(req.body.incidentDetails || '{}');
```
Multer parses the multipart form but leaves JSON fields as strings. We parse them manually.

**Step B: Build FormData for Hugging Face**
```javascript
const imageBuffer = fs.readFileSync(req.file.path);  // Read temp file into memory
const form = new FormData();

form.append('image', imageBuffer, {
  filename: req.file.originalname || 'upload.jpg',
  contentType: req.file.mimetype || 'image/jpeg'
});

// Map frontend field names to what the FastAPI expects
form.append('brand',    vehicleDetails.brand || vehicleDetails.make || 'unknown');
form.append('tier',     vehicleDetails.tier || 'mid');
form.append('segment',  vehicleDetails.segment || 'sedan');
form.append('location', vehicleDetails.damageLocation || 'front');
form.append('age',      String(vehicleDetails.car_age || 5));
```

**Step C: Call HF API**
```javascript
const hfResponse = await axios.post(HF_API_URL, form, {
  headers: form.getHeaders(),   // Sets correct multipart Content-Type with boundary
  timeout: 120000,              // 2 minutes (HF free tier can be slow on cold start)
  maxContentLength: Infinity,
  maxBodyLength: Infinity
});
let aiReport = hfResponse.data;
```

**Step C.5: IoU Deduplication (Critical Bug Fix)**

YOLO sometimes outputs multiple overlapping bounding boxes for the same physical damage. Without this filter, if YOLO outputs 3 DENT boxes for 1 dent, the dent gets charged 3 times.

```javascript
const computeOverlap = (current, kept) => {
  // Find the intersection rectangle
  const xa = Math.max(current.bbox.x1, kept.bbox.x1);
  const ya = Math.max(current.bbox.y1, kept.bbox.y1);
  const xb = Math.min(current.bbox.x2, kept.bbox.x2);
  const yb = Math.min(current.bbox.y2, kept.bbox.y2);

  // Intersection area (0 if no overlap)
  const interArea = Math.max(0, xb - xa) * Math.max(0, yb - ya);

  const currentArea = (current.bbox.x2 - current.bbox.x1) * (current.bbox.y2 - current.bbox.y1);
  const keptArea    = (kept.bbox.x2 - kept.bbox.x1) * (kept.bbox.y2 - kept.bbox.y1);
  const unionArea   = currentArea + keptArea - interArea;

  return {
    iou:         unionArea > 0 ? interArea / unionArea : 0,        // 0 to 1
    containment: currentArea > 0 ? interArea / currentArea : 0    // % of current inside kept
  };
};

for (const current of aiReport.detections) {
  let isDuplicate = false;
  for (const kept of uniqueDetections) {
    if (current.label === kept.label) {  // Only compare same damage type
      const { iou, containment } = computeOverlap(current, kept);
      if (iou > 0.30 || containment > 0.50) {  // 30% overlap OR 50% contained
        isDuplicate = true;
        break;
      }
    }
  }
  if (!isDuplicate) uniqueDetections.push(current);
}
```

After deduplication, recalculate the total:
```javascript
let newTotal = uniqueDetections.reduce((sum, d) => sum + d.price, 0);

// Actuarial range: ±15% under ₹50k, ±8% over ₹50k
if (newTotal < 50000) {
  aiReport.estimate_range = [Math.max(800, Math.floor(newTotal * 0.85)), Math.floor(newTotal * 1.15)];
} else {
  aiReport.estimate_range = [Math.floor(newTotal * 0.92), Math.floor(newTotal * 1.08)];
}
```

**Step D: Clean up temp file**
```javascript
if (fs.existsSync(req.file.path)) fs.unlinkSync(req.file.path);
```
This MUST happen even if there's an error, so we also have it in the `catch` block.

**Step E: Save to MongoDB**
```javascript
const newClaim = new Claim({
  userId: req.user?.id,   // From authMiddleware — binds claim to logged-in user
  vehicleDetails: { ...vehicleDetails, brand: vehicleDetails.brand || vehicleDetails.make },
  incidentDetails,
  aiReport: {
    total_estimate: aiReport.total_estimate,
    estimate_range: aiReport.estimate_range,
    context:        aiReport.context,
    detections:     aiReport.detections    // Now deduplicated
  }
});
const savedClaim = await newClaim.save();
```

**Step F: Respond**
```javascript
res.json({
  success: true,
  message: 'AI analysis complete',
  claimId: savedClaim._id,
  report:  aiReport    // Frontend needs this to display the report immediately
});
```

---

#### `explainClaim` — Gemini XAI

```javascript
const { claimId, message } = req.body;
const claim = await Claim.findById(claimId);

const model = getNextGeminiModel(); // Round-robin key rotation

const detectionSummary = claim.aiReport.detections
  .map(d => `- ${d.label}: ${d.severity}, ₹${d.price} (${d.summary})`)
  .join('\n');

const prompt = `
  You are the XAI module for InsureVision CrashCost.

  CLAIM CONTEXT:
  - Vehicle: ${year} ${make} ${model}
  - Tier: ${tier}, Segment: ${segment}
  - Damage Location: ${damageLocation}
  - Vehicle Age: ${age} years
  - Incident: ${description}

  AI ASSESSMENT:
  - Total Estimate: ₹${total_estimate}
  - Estimate Range: ₹${range[0]} – ₹${range[1]}
  - Detections:
  ${detectionSummary}

  USER QUESTION: "${message}"

  Provide a clear, technical but understandable explanation.
  Use bullet points. Reference the actual data above.
`;

const result = await model.generateContent(prompt);
res.json({ answer: result.response.text() });
```

---

### 6.6 `config/gemini.js` — Multi-Key Rotator

```javascript
const ALL_KEYS = [
  process.env.GEMINI_API_KEY,
  process.env.GEMINI_API_KEY_1,
  process.env.GEMINI_API_KEY_2
].filter(k => k);

if (ALL_KEYS.length === 0) {
  console.error('❌ CRITICAL: No Gemini API keys found.');
  process.exit(1);  // Crash immediately if no keys — fail fast
}

let keyIndex = 0;

const getNextGeminiModel = () => {
  const key = ALL_KEYS[keyIndex];
  keyIndex = (keyIndex + 1) % ALL_KEYS.length;  // 0→1→2→0→1→2...

  const genAI = new GoogleGenerativeAI(key);
  return genAI.getGenerativeModel({
    model: 'gemini-2.5-flash',
    safetySettings: [
      { category: 'HARM_CATEGORY_DANGEROUS_CONTENT', threshold: 'BLOCK_NONE' },
      { category: 'HARM_CATEGORY_HARASSMENT',         threshold: 'BLOCK_NONE' }
    ]
  });
};
```

**Why `process.exit(1)` on missing keys?** Fail fast. It's better to crash loudly at startup than to silently fail during a user's claim explanation and return a cryptic error.

---

## 7. AI Pipeline Deep Dive

### 7.1 What Happens Inside Hugging Face (FastAPI `main.py`)

When the Express backend sends `POST /api/v1/audit`, this is what runs on the GPU:

**Model Loading (once at startup, not per request):**
```python
YOLO_MODEL = YOLO("best.pt")             # YOLOv11 with custom weights
CLIP_MODEL = CLIPModel.from_pretrained("openai/clip-vit-large-patch14")
CLIP_PROCESSOR = CLIPProcessor.from_pretrained("openai/clip-vit-large-patch14")
CATBOOST_MODEL = CatBoostRegressor().load_model("crashcost_pricing_model.cbm")
```

**Per-Request Inference Pipeline:**

1. **Image Preprocessing:** Image is loaded with PIL, converted to OpenCV format for YOLO.

2. **YOLO Detection:**
   ```python
   results = YOLO_MODEL(img, task='segment', conf=0.3, iou=0.4)
   ```
   Returns bounding boxes, class IDs, confidence scores, and segmentation masks per detection.

3. **For each YOLO detection:**
   - Extract the bounding box crop of the damaged area
   - **CLIP Surface Classification:** Pass the crop through CLIP with candidate labels `["metal panel", "glass surface", "plastic bumper", "rubber seal"]`. CLIP returns which label best describes the crop.
   - **Damage ratio calculation:** `ratio = (bbox_area) / (total_image_area)`
   - **Severity assignment:** MINOR if ratio < 0.1, MODERATE if ratio < 0.3, SEVERE otherwise
   - **CatBoost pricing:** Build a feature vector `[damage_type_encoded, severity_encoded, surface_encoded, vehicle_tier_encoded, vehicle_segment_encoded, car_age, ratio]` and call `CATBOOST_MODEL.predict([feature_vector])`

4. **Build response:**
   ```python
   detections = [{ "id", "label", "confidence", "surface_detected", "severity",
                   "ratio", "bbox": {"x1","y1","x2","y2"}, "price",
                   "drivers", "summary", "narrative" }, ...]
   total = sum(d["price"] for d in detections)
   ```

5. **Return JSON** back to the Express backend.

### 7.2 CatBoost Input Features

| Feature | Values | Notes |
|---|---|---|
| `damage_type` | DENT, CRACK, SCRATCH, GLASS_SHATTER, LAMP_BROKEN | From YOLO class |
| `severity` | MINOR, MODERATE, SEVERE | Computed from ratio |
| `surface` | metal, glass, plastic, rubber | From CLIP |
| `tier` | budget, mid, premium, luxury | From frontend form |
| `segment` | hatchback, sedan, suv, compact_suv | From frontend form |
| `car_age` | Integer (years) | From frontend form |
| `ratio` | 0.0 to 1.0 | damage_area / total_area |

CatBoost handles categorical features natively — no one-hot encoding needed.

---

## 8. All API Routes Reference

### Authentication Routes (`/api/auth/...`)

| Method | Path | Auth | Body | Response | Purpose |
|---|---|---|---|---|---|
| POST | `/api/auth/register` | None | `{ name, email, password }` | `{ _id, name, email, token }` | Create new user |
| POST | `/api/auth/login` | None | `{ email, password }` | `{ _id, name, email, token }` | Login, get JWT |

### Claim Routes (`/api/...`)

| Method | Path | Auth | Body/Params | Response | Purpose |
|---|---|---|---|---|---|
| POST | `/api/segment-car` | ✅ JWT Required | `multipart/form-data` with `image`, `vehicleDetails`, `incidentDetails` | `{ success, claimId, report }` | Run full AI pipeline, save claim |
| POST | `/api/explain` | ❌ Public | `{ claimId, message }` JSON | `{ answer }` | Ask Gemini to explain a saved claim |
| GET | `/api/claims` | ❌ Public | Query: `?userId=<id>` | `[...claims]` | Fetch claims (filtered by userId) |
| GET | `/api/claims/:id` | ❌ Public | URL param: `:id` | Single claim object | Fetch one claim by MongoDB ID |
| GET | `/healthz` | ❌ Public | None | `"OK"` | Render health check |

**Security Note:** `/api/explain`, `/api/claims`, `/api/claims/:id` are currently not protected by JWT. This is a known gap — any user can read any other user's claims if they know the ID.

---

## 9. Authentication & Security

### JWT Lifecycle

```
1. User registers/logs in → backend creates token:
   jwt.sign({ id: user._id }, JWT_SECRET, { expiresIn: '30d' })

2. Token stored in browser:
   localStorage.setItem('token', token)

3. Subsequent requests include token:
   Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

4. authMiddleware verifies:
   const decoded = jwt.verify(token, JWT_SECRET)
   → decoded = { id: '<userId>', iat: <timestamp>, exp: <timestamp> }
   req.user = await User.findById(decoded.id)

5. After 30 days: jwt.verify throws TokenExpiredError → 401 response
   → User must login again
```

### bcrypt Password Hashing

```javascript
// Registration
const salt = await bcrypt.genSalt(10);          // 2^10 = 1024 iterations
const hashedPassword = await bcrypt.hash(password, salt);
// Store hashedPassword in MongoDB — NEVER the plaintext

// Login
const isMatch = await bcrypt.compare(plainTextPassword, storedHash);
// Returns true if they match, false otherwise
```

bcrypt is a one-way hash. Even if the database is stolen, passwords cannot be reversed.

### CORS Policy

```javascript
const allowedOrigins = [
  process.env.FRONTEND_URL,         // 'https://crash-cost-v2.vercel.app'
  'http://localhost:5173',           // Vite dev server
  'http://localhost:3000'            // Alternative dev port
].filter(Boolean);

// Requests with no Origin header (Postman, curl) are allowed (!origin check)
// Requests from listed origins are allowed
// All others → 403 "Not allowed by CORS"
```

### Rate Limiting

```javascript
rateLimit({
  windowMs: 1 * 60 * 1000,  // 1 minute sliding window
  max: 15,                   // Max 15 requests per IP per window
})
```

Applied to `/api/segment-car` and `/api/explain` — the two routes that call expensive external AI services (Hugging Face costs compute, Gemini has a rate limit).

---

## 10. Deployment Architecture

### Render (Backend)

```
GitHub Push → Render detects change → Pulls from GitHub
                                    ↓
                         npm install (Root Directory: backend/)
                                    ↓
                         npm start → node server.js
                                    ↓
                    Express app starts on process.env.PORT
                    Render's load balancer forwards HTTPS to this port
```

**Render Environment Variables required:**
- `MONGO_URI` — MongoDB Atlas connection string
- `JWT_SECRET` — Random secret for JWT signing
- `GEMINI_API_KEY` — Primary Google Gemini key
- `GEMINI_API_KEY_1` — Secondary key (rotation)
- `GEMINI_API_KEY_2` — Tertiary key (rotation)
- `FRONTEND_URL` — `https://crash-cost-v2.vercel.app` (for CORS)
- `PORT` — Set automatically by Render

**Free tier limitation:** Service spins down after 15 minutes of inactivity. First request after sleep takes ~30 seconds (cold start).

---

### Vercel (Frontend)

```
GitHub Push → Vercel detects change → Pulls from GitHub
                                    ↓
                Root Directory: frontend/
                Framework: Vite (auto-detected)
                                    ↓
                npm install
                vite build → generates dist/ folder
                                    ↓
                Vercel CDN serves dist/ globally
                vercel.json rewrites: /* → index.html
```

**Vercel Environment Variable required:**
- `VITE_API_URL` — `https://crashcostv2-1.onrender.com` (your Render URL)

**At build time: Vite replaces `import.meta.env.VITE_API_URL` with the literal string** baked into the bundle. This is why the variable starts with `VITE_` — only `VITE_` prefixed variables are exposed to client-side code.

---

## 11. Environment Variables

### Backend (`backend/.env`)

```env
PORT=5000
MONGO_URI=mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/insurevision
JWT_SECRET=some-very-long-random-string-here
GEMINI_API_KEY=AIzaSy...
GEMINI_API_KEY_1=AIzaSy...
GEMINI_API_KEY_2=AIzaSy...
FRONTEND_URL=https://crash-cost-v2.vercel.app
```

### Frontend (`frontend/.env`) — Local dev only

```env
VITE_API_URL=http://localhost:5000
```

Note: In production on Vercel, this is set as a Vercel environment variable in the dashboard, NOT committed to the repo.

---

## 12. Known Gaps & Future Work

### Security Gaps
- `GET /api/claims`, `GET /api/claims/:id`, `POST /api/explain` are **not protected by JWT**. Any user can read any other user's claims.
- **Fix:** Add `protect` middleware to these routes and filter by `req.user.id`.

### Architecture Improvements
- All API calls in the frontend still use inline `const API_URL = import.meta.env.VITE_API_URL || ''` repeated in every component. Should be centralized in `src/services/api.js`.
- The `useMockStore` hook in `dashboardPage.jsx` is a good pattern but local to one file. If the wizard ever grows, consider `useReducer` or Zustand.

### Features to Add
- **PDF export** of the assessment report
- **Real-time WebSocket** notifications when the AI completes processing
- **Image annotation overlay** showing bounding boxes of detected damage
- **Per-user claims only** in the analytics and XAI lab (by protecting those routes)
- **Admin dashboard** showing all claims, total API usage, Gemini key health
- **Offline support** via service workers (PWA)
