Project Context: CrashCost V2 (Production Architecture — March 2026)
=======================================================================

This file is the complete engineering source of truth for CrashCost V2.
It is intended to be consumed by AI assistants, future developers, or the project owner
to immediately understand the entire system without reading source code.

Author: Manideep Kattagoni (SaiManideep220305)
GitHub: https://github.com/Manideep220305/CrashCostV2
Live Frontend: https://crash-cost-v2.vercel.app
Live Backend: https://crashcostv2-1.onrender.com
AI Engine: https://saimanideep-crashcostv2.hf.space

=======================================================================
SECTION 1: WHAT THIS SYSTEM IS
=======================================================================

CrashCost V2 is a full-stack MERN + AI web application for automobile damage
assessment and insurance cost estimation. It allows a user to:

  1. Register and log in (JWT-based auth)
  2. Submit a damage claim by filling a multi-step form (vehicle details, incident
     details, photos) on the Dashboard page
  3. The backend sends the photo + metadata to a Hugging Face-hosted AI engine
     which runs three models in a pipeline:
       a. YOLOv11-Small (instance segmentation) — detects damage regions and types
       b. CLIP ViT-L/14 (visual-language model) — classifies surface material
       c. CatBoost Regressor — estimates repair cost in Indian Rupees (₹)
  4. The backend runs an IoU-based deduplication filter to remove duplicate
     YOLO detections (preventing inflated costs), then recalculates the total
  5. The claim is saved to MongoDB Atlas with full vehicle metadata, incident
     details, and the raw AI report
  6. The frontend displays an Assessment Report with:
       - Total estimate + actuarial range
       - Per-detection cards (severity badge, cost, summary, drivers)
       - "Panel Replacement Recommended" warning for high-ratio critical damage
  7. The Analytics page fetches all past claims from MongoDB and displays charts
     and a history table with inline AI chat capability
  8. The XAI Lab is a dedicated page where users can ask natural language questions
     about any specific claim. Questions are answered by Google Gemini 2.5 Flash
     which receives full claim context (vehicle, detections, prices) as a prompt.

=======================================================================
SECTION 2: FOUR-LAYER ARCHITECTURE
=======================================================================

Layer 1 — USER BROWSER
  Technology: React 19 + Vite (SPA)
  Hosted on: Vercel (CDN, global edge network)
  URL: https://crash-cost-v2.vercel.app
  Communicates with: Layer 2 via HTTPS REST API (JSON + multipart/form-data)
  Local dev URL: http://localhost:5173 (Vite dev server with /api proxy)

Layer 2 — BACKEND API SERVER
  Technology: Node.js 22 + Express
  Hosted on: Render (free tier, may cold start after 15min inactivity)
  URL: https://crashcostv2-1.onrender.com
  Communicates with: Layer 1 (responses), Layer 3 (outgoing HTTP POST), Layer 4 (MongoDB)
  Responsibilities:
    - JWT-based authentication (bcryptjs + jsonwebtoken)
    - CORS enforcement (allowlist: Vercel domain + localhost only)
    - Rate limiting (15 req/min/IP on AI routes via express-rate-limit)
    - File upload handling (Multer saves to uploads/ temporarily)
    - Forwarding image + vehicle metadata to HF AI engine
    - IoU deduplication of YOLO detections
    - Saving/reading claim documents in MongoDB
    - Calling Google Gemini for XAI explanations

Layer 3 — AI ENGINE
  Technology: FastAPI (Python) running YOLOv11 + CLIP + CatBoost
  Hosted on: Hugging Face Spaces (free GPU compute)
  URL: https://saimanideep-crashcostv2.hf.space/api/v1/audit
  Receives: multipart/form-data (image buffer + brand/tier/segment/location/age)
  Returns: JSON { detections[], total_estimate, estimate_range, context }
  Also: Gemini 2.5 Flash is called from Layer 2 for XAI (not from this layer)

Layer 4 — DATABASE
  Technology: MongoDB Atlas (cloud NoSQL, free tier M0)
  Collections: users, claims
  ORM layer: Mongoose (schema validation + queries in Layer 2)
  All claims are permanently stored here for Analytics + XAI recall

=======================================================================
SECTION 3: COMPLETE REPOSITORY FILE MAP
=======================================================================

InsureVision3/ (root)
├── README.md                     ← Professional readme with live demo badges
├── .gitignore                    ← Ignores: *.pt, *.cbm, *.csv, node_modules
│
├── documentation/                ← All project documentation
│   ├── project_context.md        ← THIS FILE
│   ├── PROJECT_DOCUMENTATION.md  ← Exhaustive technical breakdown
│   ├── LEARNING_ROADMAP.md       ← 8-week MERN placement roadmap
│   ├── ERROR_LOG.md              ← Deployment/bug error log with fixes
│   └── LOGIN_INSTRUCTIONS.md     ← How to run locally + test credentials
│
├── frontend/ (React SPA — deployed to Vercel)
│   ├── .gitignore                ← Ignores: node_modules/, dist/, .env
│   ├── .env                      ← VITE_API_URL=http://localhost:5000 (local only)
│   ├── vercel.json               ← { "rewrites": [{ "source":"/(.*)", "destination":"/index.html" }] }
│   │                               CRITICAL: Must use "rewrites" not "routes" — "routes" intercepts
│   │                               JS asset requests and returns HTML causing MIME crash
│   ├── vite.config.js            ← Vite bundler config + dev proxy: /api → localhost:5000
│   │                               The proxy ONLY works in dev. Production uses VITE_API_URL.
│   ├── tailwind.config.js        ← Tailwind config with darkMode: 'class' strategy
│   ├── postcss.config.js         ← PostCSS config for Tailwind
│   ├── index.html                ← HTML shell with <div id="root"> where React mounts
│   ├── package.json              ← react@19, react-router-dom@7, framer-motion, recharts,
│   │                               lucide-react, @radix-ui/react-dialog, axios, zustand
│   └── src/
│       ├── main.jsx              ← React entry: ReactDOM.createRoot('#root').render(<App />)
│       ├── App.jsx               ← Route config + provider wrapping (ThemeProvider > AuthProvider > Router)
│       │                           Routes: / → LandingPage, /dashboard → DashboardPage,
│       │                           /analytics → AnalyticsPage, /xai-lab → XaiLabPage,
│       │                           /insurance-101 → InsuranceGuidePage, * → LandingPage
│       ├── index.css             ← Global CSS, Tailwind @layer base/components, aurora animations
│       │
│       ├── context/
│       │   ├── AuthContext.jsx   ← Global auth state. Manages: user, token, loading, error.
│       │   │                       Functions: login(email,pw), register(name,email,pw), logout().
│       │   │                       On mount: reads user+token from localStorage, sets axios default header.
│       │   │                       API calls: POST /api/auth/login, POST /api/auth/register via axios.
│       │   │                       After success: stores in localStorage + React state + axios default.
│       │   └── ThemeContext.jsx  ← Dark/light mode. Reads from localStorage. Toggles class="dark"
│       │                           on <html> element. All Tailwind dark: classes react to this.
│       │
│       ├── pages/
│       │   ├── landingPage.jsx   ← Public homepage. Has: hero section, interactive car damage
│       │   │                       simulator, "How it works" steps, AI model cards, comparison table.
│       │   │                       Uses: Navbar, AuroraBackground, AuthModal components.
│       │   │                       No API calls made from this page.
│       │   │
│       │   ├── dashboardPage.jsx ← The most important frontend file. Multi-step claim wizard.
│       │   │   INTERNAL STATE:    useMockStore() custom hook manages all 5-step state:
│       │   │   - currentStep (1-5)
│       │   │   - claimData: { vehicleDetails, incidentDetails, uploadedImages: [{previewUrl, rawFile}] }
│       │   │   - reportResult (the full aiReport from backend)
│       │   │   - currentClaimId (MongoDB _id of saved claim)
│       │   │   - resetClaim() → resets everything back to step 1 (called by "New Claim" buttons)
│       │   │
│       │   │   STEP 1 - VehicleDetailsStep:
│       │   │     Fields: brand (text), model (text), car_age (number), tier (select:
│       │   │     budget/mid/premium/luxury), segment (select: hatchback/sedan/suv/compact_suv),
│       │   │     damageLocation (select: front/rear/side/top), car_model_val (number ₹ market value)
│       │   │
│       │   │   STEP 2 - IncidentDetailsStep:
│       │   │     Fields: date (date input), description (textarea)
│       │   │
│       │   │   STEP 3 - ImageUploadStep:
│       │   │     Stores raw File object AND a preview URL (URL.createObjectURL(file)) in state.
│       │   │     The rawFile is what gets sent to the backend.
│       │   │     The previewUrl is what gets displayed in the AssessmentReport later.
│       │   │
│       │   │   STEP 4 - ReviewStep:
│       │   │     Read-only summary of all entered data. "Analyze" button triggers handleAnalyze().
│       │   │
│       │   │   handleAnalyze() — THE CRITICAL FUNCTION:
│       │   │     1. Builds FormData: image (rawFile), vehicleDetails (JSON string), incidentDetails (JSON string)
│       │   │     2. Reads token from localStorage
│       │   │     3. fetch(VITE_API_URL + '/api/segment-car', { method:'POST',
│       │   │           headers: { Authorization: 'Bearer <token>' }, body: formData })
│       │   │        NOTE: No Content-Type header set manually — browser auto-sets multipart with boundary
│       │   │     4. On success: setReportResult(data.report), setCurrentClaimId(data.claimId), setCurrentStep(5)
│       │   │
│       │   │   STEP 5 - AssessmentReport (after API returns):
│       │   │     Renders: uploaded image, 3 stat cards (detections/confidence/location),
│       │   │     one card per detection with:
│       │   │       - Severity badge (color: MINOR=emerald, MODERATE=amber, SEVERE=red)
│       │   │       - Label (DENT, CRACK, etc.)
│       │   │       - ⚠️ Panel Replacement badge IF (ratio >= 0.40 AND label in critical_list)
│       │   │       - Price ₹
│       │   │       - Summary text
│       │   │       - Confidence %, Surface, Ratio %
│       │   │       - Cost driver pills (SHAP feature names)
│       │   │     Buttons: "Explain Logic" → navigate('/xai-lab?claimId=<id>'),
│       │   │              "New Claim" (navbar) → store.resetClaim()
│       │   │              "New Claim" (report) → onReset prop which calls store.resetClaim()
│       │   │
│       │   ├── analyticsPage.jsx ← Analytics + claims history. Protected: redirects to / if !user.
│       │   │   ON MOUNT: fetch(API_URL + '/api/claims?userId=' + user._id)
│       │   │   DISPLAYS:
│       │   │     - Stat cards: totalClaims, avgConfidence (flatMap detections), totalCost (sum)
│       │   │     - BarChart (Recharts): damage type distribution (all detections)
│       │   │     - PieChart (Recharts): severity distribution (MINOR/MODERATE/SEVERE counts)
│       │   │     - Claims history table: each row = one claim, click to expand detection list
│       │   │     - "Ask AI" button per claim: opens inline chat box, calls POST /api/explain
│       │   │
│       │   ├── xaiLabPage.jsx    ← Dedicated Gemini XAI chat. Protected: redirects to / if !user.
│       │   │   URL parameter: ?claimId=<id> pre-selects a claim
│       │   │   ON MOUNT: fetch(API_URL + '/api/claims?userId=' + user._id) to populate dropdown
│       │   │   CHAT FLOW:
│       │   │     - User types question → POST /api/explain { claimId, message }
│       │   │     - Renders Gemini response in chat history UI
│       │   │   RIGHT PANEL: Selected claim summary (vehicle, tier, segment, total estimate, detection list)
│       │   │
│       │   └── insuranceGuidePage.jsx ← Static education page about insurance basics. No API calls.
│       │
│       ├── components/
│       │   ├── Sidebar.jsx       ← Left navigation for authenticated pages (dashboard/analytics/xai-lab).
│       │   │                       Shows: CrashCost logo, nav links (Dashboard/Analytics/XAI Lab/Insurance 101),
│       │   │                       theme toggle button, logout button (calls AuthContext.logout()).
│       │   │                       Collapsed on mobile, expanded on desktop.
│       │   ├── Navbar.jsx        ← Top navigation for landing page only.
│       │   │                       Shows: Logo, nav links, theme toggle, Login/Register buttons.
│       │   │                       Login/Register buttons open AuthModal.
│       │   ├── AuthModal.jsx     ← Radix UI Dialog containing login/register tabs.
│       │   │                       Calls AuthContext.login() or AuthContext.register().
│       │   │                       On success: modal closes and auth state is updated globally.
│       │   ├── AuroraBackground.jsx ← Three.js canvas with animated aurora waves using React Three Fiber.
│       │   │                          Used on landing page for visual impact.
│       │   └── ui/glowing-effect.jsx ← Reusable component that adds a glowing CSS border effect.
│       │
│       ├── services/
│       │   └── api.js            ← Axios wrapper (get/post/put/delete). Base URL from VITE_API_URL.
│       │                           NOT used by main pages — main pages use fetch() directly.
│       └── controllers/
│           └── useApi.js         ← usApi() hook wrapping api.js with loading/error state management.
│                                   Also not used in main flows.
│
├── backend/ (Express API — deployed to Render, root dir: backend/)
│   ├── .gitignore                ← Ignores: node_modules/, uploads/, .env
│   ├── package.json              ← express, mongoose, multer, axios, form-data, dotenv, cors,
│   │                               jsonwebtoken, bcryptjs, express-rate-limit, @google/generative-ai
│   │
│   ├── server.js                 ← PRIMARY ENTRY POINT. Exact initialization order:
│   │                               1. require('dotenv').config() — loads .env
│   │                               2. const app = express()
│   │                               3. app.set('trust proxy', 1) — REQUIRED for Render: makes rate
│   │                                  limiter use real client IP not Render load balancer IP
│   │                               4. CORS middleware with allowedOrigins array:
│   │                                  [process.env.FRONTEND_URL, 'http://localhost:5173', 'http://localhost:3000']
│   │                                  .filter(Boolean) — removes undefined if FRONTEND_URL not set
│   │                               5. express.json() + express.urlencoded()
│   │                               6. Rate limiter: 15 req/min/IP on /api/segment-car and /api/explain
│   │                               7. fs.mkdirSync('uploads') if not exists
│   │                               8. mongoose.connect(MONGO_URI) — logs success/failure
│   │                               9. app.use('/api', claimRoutes)
│   │                               10. app.use('/api/auth', authRoutes)
│   │                               11. app.get('/healthz', res => res.send('OK')) — Render health check
│   │                               12. app.listen(PORT)
│   │
│   ├── config/
│   │   └── gemini.js             ← Multi-key Gemini rotator.
│   │                               Loads GEMINI_API_KEY, GEMINI_API_KEY_1, GEMINI_API_KEY_2 from env.
│   │                               Filters empty keys. process.exit(1) if zero keys found (fail fast).
│   │                               getNextGeminiModel(): round-robin key selection, returns Gemini
│   │                               2.5 Flash model with safety BLOCK_NONE for dangerous+harassment.
│   │
│   ├── models/
│   │   ├── User.js               ← Schema: { name: String required, email: String required unique,
│   │   │                           password: String required (bcrypt hash) }. timestamps: true.
│   │   └── Claim.js              ← Schema: { userId: ObjectId ref User, vehicleDetails: { vin, model,
│   │                               year, make, brand, tier, segment, damageLocation, car_model_val,
│   │                               car_age }, incidentDetails: { date, description },
│   │                               aiReport: { total_estimate, estimate_range[2], context: {brand,tier,
│   │                               segment,location}, detections: [{ id, label, confidence,
│   │                               surface_detected, severity, ratio, bbox(Mixed), price, drivers[],
│   │                               summary, narrative }] }, status: default 'Auto-Assessed' }. timestamps.
│   │
│   ├── middleware/
│   │   └── authMiddleware.js     ← protect function:
│   │                               1. Checks Authorization: Bearer <token> header
│   │                               2. jwt.verify(token, JWT_SECRET) → decoded = { id, iat, exp }
│   │                               3. User.findById(decoded.id).select('-password') → req.user
│   │                               4. next() → controller runs
│   │                               On any failure → 401 { message: 'Not authorized' }
│   │
│   ├── controllers/
│   │   ├── authController.js     ← registerUser: validate, findOne(email), bcrypt.hash, User.create,
│   │   │                           generateToken, return { _id, name, email, token }.
│   │   │                           loginUser: special case test@test.com/password bypass,
│   │   │                           otherwise User.findOne(email), bcrypt.compare, generateToken.
│   │   │                           generateToken(id): jwt.sign({ id }, JWT_SECRET, { expiresIn: '30d' })
│   │   │
│   │   └── claimController.js    ← MOST IMPORTANT FILE. Four exported functions:
│   │
│   │       analyzeClaim (POST /api/segment-car):
│   │         A. JSON.parse(req.body.vehicleDetails) and incidentDetails
│   │         B. fs.readFileSync(req.file.path) → imageBuffer
│   │            Build FormData for HF: image(buffer), brand, tier, segment, location, age
│   │         C. axios.post(HF_API_URL='https://saimanideep-crashcostv2.hf.space/api/v1/audit',
│   │            form, { headers: form.getHeaders(), timeout: 120000 })
│   │            → aiReport = { detections, total_estimate, estimate_range, context }
│   │         C.5. IoU DEDUPLICATION:
│   │            computeOverlap(a, b): calculates IoU = intersectionArea/unionArea
│   │                                  and containment = intersectionArea/aArea
│   │            Loop: for each detection, compare with already-kept detections of same label.
│   │            Drop if IoU > 0.30 OR containment > 0.50.
│   │            If any were dropped: recalculate total_estimate = sum of unique prices.
│   │            Recalculate estimate_range: under ₹50k → ±15%, over ₹50k → ±8%.
│   │         D. fs.unlinkSync(req.file.path) — delete temp upload
│   │         E. new Claim({ userId: req.user?.id, vehicleDetails, incidentDetails, aiReport }).save()
│   │         F. res.json({ success: true, claimId: savedClaim._id, report: aiReport })
│   │         ERROR: timeout → 504 with helpful message. Other errors → 500.
│   │
│   │       explainClaim (POST /api/explain):
│   │         1. const { claimId, message } = req.body
│   │         2. Claim.findById(claimId)
│   │         3. getNextGeminiModel()
│   │         4. Build prompt with full claim context:
│   │            "You are XAI module... CLAIM CONTEXT: Vehicle: [brand model], Tier, Segment,
│   │             Location, Age, Incident description. AI ASSESSMENT: Total, Range,
│   │             Detections: [list with label/severity/price/summary]. USER QUESTION: [message]"
│   │         5. model.generateContent(prompt) → res.json({ answer: text })
│   │
│   │       getAllClaims (GET /api/claims):
│   │         const { userId } = req.query
│   │         Claim.find(userId ? { userId } : {}).sort({ createdAt: -1 })
│   │         res.json(claims)
│   │
│   │       getClaimById (GET /api/claims/:id):
│   │         Claim.findById(req.params.id) → 404 if not found, else res.json(claim)
│   │
│   └── routes/
│       ├── authRoutes.js         ← POST /register → registerUser, POST /login → loginUser (both public)
│       └── claimRoutes.js        ← POST /segment-car → [protect, upload.single('image'), analyzeClaim]
│                                   POST /explain → explainClaim (public)
│                                   GET /claims → getAllClaims (public)
│                                   GET /claims/:id → getClaimById (public)
│                                   Multer config: dest: 'uploads/' (disk storage, temp files)
│
└── huggingface-api/ (FastAPI — deployed to HF Spaces separately, NOT via this repo's CI/CD)
    ├── main.py                   ← FastAPI app. Models loaded at startup (not per request).
    │                               POST /api/v1/audit: receives multipart, runs YOLO+CLIP+CatBoost,
    │                               returns JSON with detections array and total_estimate.
    ├── best.pt                   ← YOLOv11-Small custom weights (60MB, excluded from git)
    ├── crashcost_pricing_model.cbm ← CatBoost regressor weights (4MB, excluded from git)
    └── requirements.txt          ← Python deps: fastapi, uvicorn, ultralytics, transformers,
                                    torch, catboost, pillow, opencv-python

=======================================================================
SECTION 4: COMPLETE API ROUTE MAP
=======================================================================

BASE URL (production): https://crashcostv2-1.onrender.com
BASE URL (development): http://localhost:5000

AUTH ROUTES (public, no JWT required):

  POST /api/auth/register
    Body: { "name": "...", "email": "...", "password": "..." }
    Success 200: { "_id": "...", "name": "...", "email": "...", "token": "<JWT>" }
    Error 400: { "message": "User already exists" }
    Handler: authController.registerUser

  POST /api/auth/login
    Body: { "email": "...", "password": "..." }
    Success 200: { "_id": "...", "name": "...", "email": "...", "token": "<JWT>" }
    Error 401: { "message": "Invalid credentials" }
    Special: email=test@test.com, password=password → returns mock user without DB hit
    Handler: authController.loginUser

CLAIM ROUTES:

  POST /api/segment-car  [JWT REQUIRED]
    Middleware chain: protect → upload.single('image') → analyzeClaim
    Body: multipart/form-data
      image: <File> (JPEG/PNG of damaged vehicle)
      vehicleDetails: '{"brand":"Toyota","tier":"mid","segment":"sedan","damageLocation":"front","car_age":3}'
      incidentDetails: '{"date":"2026-03-08","description":"Hit a pole"}'
    Success 200: { "success": true, "claimId": "<MongoDB ObjectId>", "report": { <aiReport> } }
    Error 400: { "error": "No image uploaded" }
    Error 504: { "error": "AI Engine timeout" }  (HF cold start)
    Error 500: { "error": "Analysis failed", "details": "..." }
    Rate limited: 15 req/min/IP

  POST /api/explain  [PUBLIC]
    Body: { "claimId": "<MongoDB ObjectId>", "message": "Why is this SEVERE?" }
    Success 200: { "answer": "<Gemini generated text>" }
    Error 404: { "error": "Claim not found in database" }
    Rate limited: 15 req/min/IP

  GET /api/claims  [PUBLIC]
    Query: ?userId=<MongoDB ObjectId>  (if omitted, returns ALL claims — intended for admin only)
    Success 200: [ { <claim1> }, { <claim2> }, ... ]  (sorted newest first)
    Handler: claimController.getAllClaims

  GET /api/claims/:id  [PUBLIC]
    Params: id = MongoDB ObjectId of claim
    Success 200: { <full claim document> }
    Error 404: { "error": "Claim not found" }
    Handler: claimController.getClaimById

  GET /healthz  [PUBLIC]
    Success 200: "OK"
    Purpose: Render health check to keep service alive

=======================================================================
SECTION 5: DATA FLOW SEQUENCES
=======================================================================

--- CLAIM SUBMISSION FLOW ---

Browser → POST /api/segment-car (multipart, with JWT)
    → authMiddleware: jwt.verify → User.findById → req.user set
    → multer: saves image to uploads/tmpXXX.jpg
    → analyzeClaim():
        → fs.readFileSync → imageBuffer
        → build FormData (image + vehicle metadata)
        → axios.post(HF FastAPI, formData)
            → inside HF:
               → YOLOv11 detects damage boxes + classes + confidence + masks
               → CLIP classifies surface material for each detection
               → ratio = box_area / total_image_area
               → severity = MINOR/MODERATE/SEVERE based on ratio thresholds
               → CatBoost predicts price per detection
               → returns JSON with all detections + total
        → IoU deduplication (backend):
            → for each pair with same label:
               → intersectionArea / unionArea > 0.30 → duplicate
               → intersectionArea / currentArea > 0.50 → contained, duplicate
               → keep only unique detections
            → recalculate total + range
        → fs.unlinkSync (delete temp file)
        → Claim.save({ userId, vehicleDetails, incidentDetails, aiReport })
        → return { success, claimId, report }
Browser ← { success: true, claimId: "...", report: { detections[], total_estimate, ... } }
    → setReportResult(report) → AssessmentReport renders

--- XAI FLOW ---

User navigates to /xai-lab?claimId=<id>
Browser → GET /api/claims?userId=<id> → gets all claims list
User types question → clicks send
Browser → POST /api/explain { claimId, message }
    → Claim.findById(claimId) from MongoDB
    → getNextGeminiModel() (round-robin key rotation)
    → model.generateContent(prompt with full claim context + question)
    → Google AI API call
Browser ← { answer: "<Gemini text response>" }
    → renders in chat history

=======================================================================
SECTION 6: ENVIRONMENT VARIABLES
=======================================================================

BACKEND (set in Render dashboard for production):
  MONGO_URI          = mongodb+srv://<user>:<pw>@cluster.mongodb.net/insurevision
  JWT_SECRET         = <long random string>
  GEMINI_API_KEY     = AIzaSy... (primary key)
  GEMINI_API_KEY_1   = AIzaSy... (secondary key for rotation)
  GEMINI_API_KEY_2   = AIzaSy... (tertiary key for rotation)
  FRONTEND_URL       = https://crash-cost-v2.vercel.app (CORS allowlist)
  PORT               = automatically set by Render

FRONTEND (set in Vercel dashboard for production):
  VITE_API_URL       = https://crashcostv2-1.onrender.com (NO trailing slash)

FRONTEND LOCAL DEV (frontend/.env file, NOT committed to git):
  VITE_API_URL       = http://localhost:5000

NOTE: In Vite, only variables prefixed with VITE_ are exposed to client-side code.
At build time, Vite replaces import.meta.env.VITE_API_URL with the literal string.
This is safe because these values are baked into the public JS bundle.

=======================================================================
SECTION 7: AUTHENTICATION & SECURITY MODEL
=======================================================================

PASSWORD STORAGE:
  Never stored in plaintext. bcrypt.hash(password, 10) at registration.
  bcrypt.compare(plain, hash) at login. 2^10 = 1024 hash iterations.

JWT TOKENS:
  Signed with JWT_SECRET using HS256 algorithm.
  Payload: { id: <MongoDB ObjectId>, iat: <unix timestamp>, exp: <unix timestamp> }
  Expiry: 30 days.
  Stored in browser localStorage (persists across page refreshes and browser restarts).
  Sent as Authorization: Bearer <token> header on every protected request.

CORS POLICY:
  In production, only these origins are allowed:
    - https://crash-cost-v2.vercel.app (set via FRONTEND_URL env var)
    - http://localhost:5173
    - http://localhost:3000
  Requests with no Origin header (Postman, curl, server-to-server) are allowed.
  All other origins receive a CORS error.

RATE LIMITING:
  15 requests per minute per IP on /api/segment-car and /api/explain.
  These are the routes that call external paid APIs (HF + Gemini).
  Returns 429 Too Many Requests with message about cooling down.

SECURITY GAPS (known, not yet fixed):
  - GET /api/claims, GET /api/claims/:id, POST /api/explain are NOT JWT-protected.
    Any user who knows a claimId can read the claim details.
  - No per-user filtering enforcement on the server — the frontend filters by userId
    as a query param which the client controls.

=======================================================================
SECTION 8: DEPLOYMENT CONFIGURATION
=======================================================================

RENDER (backend):
  Root Directory: backend
  Build Command: npm install
  Start Command: npm start  (runs "node server.js")
  Health Check Path: /healthz
  Free tier: cold starts after 15min inactivity (~30sec wake-up on first request)

VERCEL (frontend):
  Root Directory: frontend
  Framework Preset: Vite (auto-detected)
  Build Command: vite build (auto-detected)
  Output Directory: dist (auto-detected)
  vercel.json: { "rewrites": [{ "source":"/(.*)", "destination":"/index.html" }] }
  CRITICAL: Use rewrites, not routes. Routes intercepts all requests including JS/CSS
  assets and returns HTML, causing a MIME type crash and white screen.

GITHUB → VERCEL CI/CD:
  Every push to main branch triggers automatic Vercel rebuild.
  Build time: ~45 seconds for the Vite React app.

GITHUB → RENDER CI/CD:
  Every push to main branch triggers automatic Render redeploy.
  Build time: ~2-3 minutes including npm install.

=======================================================================
SECTION 9: KNOWN ISSUES, BUGS FIXED, AND FUTURE WORK
=======================================================================

BUGS FIXED (chronological, see ERROR_LOG.md for full detail):
1. YOLO duplicate detections → inflated costs. Fixed by IoU deduplication in claimController.js.
2. Vite dev proxy not available in production → all API calls returned 404 on Vercel.
   Fixed by adding VITE_API_URL env var and using it in all fetch() calls.
3. CORS policy was open (app.use(cors())) → restricted to FRONTEND_URL allowlist.
4. No .gitignore → node_modules, model files, datasets in GitHub.
   Fixed by adding .gitignore at root, frontend/, and backend/.
5. "New Claim" buttons used window.location.href → full page reload to landing page.
   Fixed by using store.resetClaim() to reset local state in-place.
6. AlertTriangle missing from lucide-react imports → build error in dashboard/analytics.
   Fixed by adding AlertTriangle to import statements.
7. vercel.json used "routes" instead of "rewrites" → JS assets returned HTML → white screen.
   Fixed by replacing routes with rewrites.
8. FRONTEND_URL not set on Render → CORS blocked Vercel from hitting any API.
   Fixed by setting FRONTEND_URL env var in Render dashboard.

KNOWN CURRENT GAPS:
- GET /api/claims, /api/claims/:id, POST /api/explain not JWT-protected
- No server-side enforcement of per-user claim ownership
- Mixed API calling patterns (fetch vs axios, inline vs api.js helper)
- Panel replacement warning only checks 4 labels — could be expanded

FUTURE IMPROVEMENTS:
- Protect all routes with JWT, enforce per-user data ownership
- PDF export of assessment report
- WebSocket real-time notifications for claim processing status
- Image annotation overlay showing YOLO bounding boxes on the uploaded photo
- Admin dashboard with API usage analytics and Gemini key health
- Redis caching for frequently accessed claims
- Unit tests (Jest + Testing Library) for controllers and pages

=======================================================================
SECTION 10: HOW TO RUN LOCALLY
=======================================================================

Prerequisites: Node.js v18+, MongoDB (local or Atlas), Gemini API key

Step 1: Clone the repo
  git clone https://github.com/Manideep220305/CrashCostV2.git
  cd CrashCostV2

Step 2: Start the backend
  cd backend
  npm install
  Create backend/.env with:
    MONGO_URI=mongodb://localhost:27017/insurevision  (or Atlas URI)
    JWT_SECRET=any-random-string
    GEMINI_API_KEY=your-gemini-key
    FRONTEND_URL=http://localhost:5173
  npm run dev  (uses nodemon if installed, otherwise: node server.js)
  Backend starts at http://localhost:5000

Step 3: Start the frontend
  cd frontend
  npm install
  Create frontend/.env with:
    VITE_API_URL=http://localhost:5000
  npm run dev
  Frontend starts at http://localhost:5173

Step 4: Test with quick login
  Email: test@test.com
  Password: password
  (This bypasses MongoDB — works even if DB is not connected)

The Vite dev server proxy in vite.config.js forwards /api/* requests to localhost:5000
so the frontend and backend can communicate without CORS issues in dev.
