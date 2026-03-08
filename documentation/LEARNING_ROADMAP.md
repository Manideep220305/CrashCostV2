# MERN Stack Learning Roadmap
### Placement-Ready in 2 Months
### Tailored for Manideep — based on CrashCost V2 architecture

---

> **READ THIS FIRST:** You are not a beginner. You just built and deployed a production full-stack AI web application with JWT auth, MongoDB, Gemini API, custom React hooks, IoU-based deduplication logic, rate limiting, CORS policies, and multi-key rotation. Most placement candidates have not done any of this. Your goal for the next 2 months is to UNDERSTAND what you built deeply, fill knowledge gaps, and practice DSA enough not to get filtered in the first round.

---

## Current Skill Inventory (Honest Assessment)

**What you BUILT in CrashCost V2 (can speak about confidently):**
- ✅ React 19 — functional components, hooks (useState, useEffect, useContext), React Router
- ✅ Custom hooks (`useMockStore`, `useApi`) — state abstraction pattern
- ✅ React Context API — `AuthContext` (JWT persistence), `ThemeContext` (dark mode)
- ✅ REST API design — correct HTTP methods, JSON bodies, status codes
- ✅ Express + Mongoose — MVC pattern, routes, controllers, models, middleware
- ✅ JWT authentication — sign, verify, decode, 30-day expiry, Bearer token pattern
- ✅ bcrypt password hashing — hash at register, compare at login
- ✅ File uploads — Multer, multipart/form-data, temp file lifecycle
- ✅ CORS configuration — allowlist, credentials, origin function
- ✅ Rate limiting — express-rate-limit, windowMs, max, trust proxy
- ✅ Third-party APIs — Axios calls to Hugging Face, Gemini SDK integration
- ✅ MongoDB Atlas — cloud deployment, Mongoose schema, CRUD, sort
- ✅ Deployment — Render (backend), Vercel (frontend), env vars, CI/CD
- ✅ Git — add, commit, push, pull, .gitignore, fixing tracked files
- ✅ Debugging production issues — CORS errors, MIME type errors, cold starts

**What you need to learn/solidify:**
- ⚠️ JavaScript deep fundamentals (closures, prototype, event loop, `this`)
- ⚠️ React internals (virtual DOM, reconciliation, render optimization)
- ⚠️ Node.js internals (event loop, non-blocking I/O, streams)
- ⚠️ Data Structures & Algorithms (mandatory for placement rounds)
- ⚠️ System design basics (for technical interviews at product companies)
- ⚠️ Testing (zero tests in CrashCost V2 — placement projects need this)

---

## Month 1 — Understanding What You Built + Filling Gaps

### Week 1: JavaScript Fundamentals That Interviewers Actually Ask

These are the exact JS topics that appear in every MERN placement interview. You need to answer these in 30 seconds without searching.

#### Closures
```javascript
// What is this output and why?
function counter() {
  let count = 0;
  return function() {
    count++;
    return count;
  };
}
const c = counter();
console.log(c()); // 1
console.log(c()); // 2
// Answer: The inner function "closes over" count — it keeps a reference to
// the variable even after counter() has returned. Each call to c() uses the SAME count.
```
You already used closures in `getNextGeminiModel()` — the `keyIndex` variable is closed over by the returned function.

#### `this` Keyword
```javascript
const obj = {
  name: 'CrashCost',
  greet: function() { console.log(this.name); },        // 'CrashCost'
  arrowGreet: () => { console.log(this.name); }          // undefined (arrow has no 'this')
};
obj.greet();         // logs 'CrashCost'
obj.arrowGreet();    // logs undefined (or global)
```
In Express controllers: `const protect = async (req, res, next) => { ... }` — arrow functions are used so `this` doesn't change when Express calls the function.

#### Promises and async/await
```javascript
// These 3 are equivalent:
fetch('/api/claims')
  .then(res => res.json())
  .then(data => console.log(data))
  .catch(err => console.error(err));

// vs async/await (what you use in CrashCost):
const res = await fetch('/api/claims');
const data = await res.json();

// What happens if you forget await?
const res = fetch('/api/claims');  // res is a Promise object, not the response!
```

#### Event Loop
```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);  // Queued in task queue
Promise.resolve().then(() => console.log('3'));  // Queued in microtask queue
console.log('4');

// Output: 1, 4, 3, 2
// Why: Call stack runs 1 and 4 first. Microtasks (Promise) run before
// task queue (setTimeout) even if setTimeout is 0ms.
```
This is why your `async/await` calls in Express controllers work correctly — Node's event loop handles them without blocking other requests.

#### Array Methods (Write these from scratch)
```javascript
const detections = [
  { label: 'DENT', price: 5000, severity: 'SEVERE' },
  { label: 'CRACK', price: 3000, severity: 'MODERATE' },
  { label: 'SCRATCH', price: 1000, severity: 'MINOR' }
];

// map: transform each element
const prices = detections.map(d => d.price);         // [5000, 3000, 1000]

// filter: keep matching elements
const severe = detections.filter(d => d.severity === 'SEVERE');  // [DENT obj]

// reduce: accumulate to a single value
const total = detections.reduce((sum, d) => sum + d.price, 0);  // 9000

// find: first match
const crack = detections.find(d => d.label === 'CRACK');  // CRACK obj

// flatMap: map then flatten (used in analyticsPage)
const allDrivers = claims.flatMap(c => c.aiReport.detections);
```

**Practice:** Every day, open your `analyticsPage.jsx` and re-read the derived state calculations. You already wrote `flatMap`, `reduce`, and `filter` — explain each one out loud.

---

### Week 2: React Internals

#### Virtual DOM & Reconciliation
React maintains a lightweight copy of the DOM in memory (the Virtual DOM). When state changes, React:
1. Creates a new Virtual DOM
2. Diffs it against the old one (reconciliation — uses fiber algorithm)
3. Calculates the minimal set of real DOM changes needed
4. Applies only those changes (commit phase)

This is why React is fast — it batches DOM writes instead of making one per state change.

**Why you need to know this for interviews:** "Why is React better than vanilla JS?" → Virtual DOM + reconciliation is the answer.

#### Why `useEffect` Dependencies Matter
```javascript
// In analyticsPage.jsx:
useEffect(() => {
  fetchClaims();    // GET /api/claims
}, [user]);         // Re-runs ONLY when user object changes

// What happens if you put nothing?
useEffect(() => {
  fetchClaims();    // Runs ONCE after component mounts
}, []);

// What happens if you omit the array?
useEffect(() => {
  fetchClaims();    // Runs after EVERY render — will cause infinite loop if fetchClaims updates state
});
```

**Your real example:** In `AuthContext.jsx`, the `useEffect` with `[]` runs once on mount to read from localStorage. Perfect use case.

#### Controlled vs Uncontrolled Inputs
```javascript
// Controlled (what you use in the wizard):
const [value, setValue] = useState('');
<input value={value} onChange={e => setValue(e.target.value)} />
// React controls the input value. The source of truth is React state.

// Uncontrolled (using ref):
const inputRef = useRef();
<input ref={inputRef} />
// DOM controls the value. React just has a reference to the DOM node.
```

#### `useMemo` and `useCallback`
```javascript
// useMemo: cache an expensive computation
const damageChartData = useMemo(() => {
  return claims.flatMap(c => c.aiReport.detections)
    .reduce((acc, d) => { /* count by label */ }, {});
}, [claims]);  // Only recalculate when claims array changes

// useCallback: cache a function reference
const handleAnalyze = useCallback(async () => {
  // ... fetch logic
}, [claimData]);  // Only recreate function when claimData changes
```

Your `analyticsPage.jsx` recomputes all charts on every render because calculations are inline. This is fine at your scale but worth knowing for interviews.

---

### Week 3: Node.js & Express Internals

#### How Node Handles Multiple Requests
Node.js is single-threaded. It uses a non-blocking I/O model:
- When Express receives a request, it starts handling it
- When it hits `await Claim.findById(id)` (a MongoDB query), Node doesn't wait
- Instead, Node moves on to handle the next incoming request
- When MongoDB finishes, Node's event loop picks up the callback and continues

This is why your Express server can handle multiple users at once despite being single-threaded. The bottleneck is CPU-bound work (which you offload to HF), not I/O.

#### Middleware Chain in Detail
Every Express middleware `(req, res, next)` works like a pipeline:
```
Request comes in
      ↓
cors() middleware — adds headers, blocks if origin not allowed
      ↓
express.json() — parses body if Content-Type: application/json
      ↓
aiLimiter — checks request count, blocks if over limit
      ↓
router.post('/segment-car', protect, upload.single('image'), analyzeClaim)
      ↓
protect() — verifies JWT, sets req.user, calls next() or sends 401
      ↓
upload.single('image') — multer reads multipart body, saves file, calls next()
      ↓
analyzeClaim() — your controller
```

If any middleware sends a response without calling `next()`, the chain stops there.

#### Error Handling Middleware (you're missing this)
```javascript
// 4-parameter middleware — Express knows this is an error handler
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.statusCode || 500).json({
    message: err.message || 'Server Error'
  });
});
```
Right now your CrashCost backend has try/catch blocks in every controller. A centralized error handler is cleaner. Add this to `server.js` as a future improvement.

---

### Week 4: MongoDB & Mongoose Deeply

#### Understand Your Claim Schema
The nested `detections` array inside `aiReport` is an embedded document pattern.

**When to embed:** When data is always read/written together (a detection never exists without its claim)

**When to reference:** When you need to query the sub-document independently. For example, if we needed a "search by detection label" feature, we'd reference instead.

#### Aggregation Pipeline (Write this for interviews)
```javascript
// "How many claims per severity for user X?"
const stats = await Claim.aggregate([
  { $match: { userId: mongoose.Types.ObjectId(userId) } },
  { $unwind: '$aiReport.detections' },
  { $group: {
    _id: '$aiReport.detections.severity',
    count: { $sum: 1 },
    totalCost: { $sum: '$aiReport.detections.price' }
  }},
  { $sort: { count: -1 } }
]);
```

This is what a real analytics endpoint would look like. Your current `/api/claims` returns raw arrays and the frontend does the grouping — not ideal for large datasets.

#### Indexes
```javascript
// In User.js, email has unique: true — Mongoose auto-creates an index.
// Indexes speed up queries on that field by avoiding full collection scans.

// If you add this to your Claim model:
ClaimSchema.index({ userId: 1, createdAt: -1 });
// Now queries for a user's claims sorted by date are instant, even with 1M docs
```

---

## Month 2 — Placement Preparation

### Week 5: Data Structures & Algorithms

You need this to pass the first technical screening at most companies.

**Target:** 50 LeetCode Easy problems solved in JavaScript by end of this week. That's 7/day.

**Most important patterns for MERN placements:**

| Pattern | LeetCode Problems | Why It Comes Up |
|---|---|---|
| Two Pointers | Valid Palindrome, Container with Water | Array manipulation |
| Sliding Window | Max Subarray, Longest Substring | String/array problems |
| Hash Map | Two Sum, Group Anagrams, Contains Duplicate | All interviewers love these |
| Binary Search | Search in Rotated Array, Find Peak Element | Efficiency questions |
| BFS/DFS | Number of Islands, Clone Graph | Anything tree/graph shaped |
| Recursion | Fibonacci, Subsets, Permutations | Fundamental thinking |

**Your IoU deduplication algorithm IS an example of computational geometry.** You already coded this:
```javascript
const interArea = Math.max(0, xb - xa) * Math.max(0, yb - ya);
const iou = interArea / (areaA + areaB - interArea);
```
Be ready to explain what IoU means and why you derived it this way.

---

### Week 6: System Design Fundamentals

For MERN developer roles at product companies (Zepto, Razorpay, Meesho, etc.), expect 1-2 system design questions.

**Designs to prepare based on your project:**

#### "Design the CrashCost AI Pipeline"
Draw this confidently:
```
Client → CDN (Vercel) → Load Balancer → Express API (Render)
                                               ↓
                                      Rate Limiter (Redis in prod)
                                               ↓
                                      Message Queue (Bull/Redis) ← async
                                               ↓
                                      AI Worker Service (HF/GPU)
                                               ↓
                                      MongoDB Atlas
```

**Improvements you can propose:**
- Replace synchronous HF call (current) with async job queue (Bull) so the user doesn't wait 30s
- Redis for caching repeat claim analyses on same vehicle/damage combo
- S3 for storing original uploaded images (currently deleted after processing)

#### "Design a Secure Authentication System"
You built this. Talk through:
- bcrypt hashing (why not MD5? — not reversible vs reversible)
- JWT (why stateless? — no server-side session storage needed, scales horizontally)
- JWT expiry (30 days) + token refresh (you haven't implemented refresh yet)
- CORS + HTTPS (you did implement both)
- Rate limiting on auth endpoints (you haven't added this — a good answer for "what would you improve?")

#### HTTP Status Codes You Must Know Cold
| Code | Meaning | Your usage |
|---|---|---|
| 200 | OK | All successful API responses |
| 201 | Created | Should be on /register and /segment-car success |
| 400 | Bad Request | No image uploaded |
| 401 | Unauthorized | Invalid/missing JWT |
| 403 | Forbidden | CORS origin not allowed |
| 404 | Not Found | Claim not in MongoDB |
| 429 | Too Many Requests | Rate limiter triggered |
| 500 | Internal Server Error | Unhandled exceptions |
| 504 | Gateway Timeout | HF cold start timeout |

---

### Week 7: Interview Talking Points (Your Strongest Asset)

CrashCost V2 is genuinely impressive for a placement candidate. Every interviewer will ask you to walk them through it. Prepare these answers:

#### "Tell me about your most challenging project"
**Template answer:**
> "I built CrashCost V2 — an AI-powered automobile damage assessment system. The user uploads a photo of car damage and gets an instant repair cost estimate in Indian Rupees. The system uses three AI models in a pipeline: YOLOv11 for damage detection, CLIP for surface material classification, and CatBoost for cost regression. I built the full stack — React frontend on Vercel, Node.js Express API on Render, MongoDB Atlas for storage, and Hugging Face for the AI compute. One of the hardest problems I solved was that YOLO was outputting duplicate bounding boxes for the same damage, inflating the cost 2-3x. I implemented an IoU-based deduplication filter in the backend that removes boxes where more than 30% of the area overlaps another box of the same class, then recalculates the total."

#### "What was the hardest bug you fixed?"
**The Vercel white screen answer:**
> "When I deployed to Vercel, the site showed a completely blank white page even though the build succeeded. The browser console showed a MIME type error — the server was returning HTML when the browser expected JavaScript for the main bundle. I traced it to the `vercel.json` routing config. I was using `routes` which intercepts ALL requests including requests for static JS files and returns `index.html` (HTML) instead. The fix was switching to `rewrites` which only acts as a fallback when no static file exists on disk, so JS bundles are served correctly while React Router routes still fall back to index.html."

#### "How did you handle authentication?"
> "I used the standard JWT pattern. On registration, passwords are hashed with bcrypt (10 salt rounds). On login, I compare the input password against the stored hash. If they match, I sign a JWT with the user's MongoDB ObjectId and a 30-day expiry using a secret key stored in environment variables. The frontend stores this token in localStorage and sends it as a Bearer token header on every API call. The backend has an authMiddleware that verifies the JWT signature and attaches the decoded user to req.user before the controller runs."

#### "What would you improve if you had more time?"
> "Three things: First, protect all GET /api/claims routes with JWT so users can only see their own data — right now those are public. Second, replace the synchronous Hugging Face API call with an async job queue using Bull and Redis so the user gets a pending status immediately instead of waiting 30-90 seconds. Third, add a comprehensive test suite with Jest and React Testing Library — I have zero automated tests right now which is a clear gap for production readiness."

---

### Week 8: Mock Interviews + Final Review

**Do only this in week 8:**

1. **Morning:** 3 LeetCode problems in JavaScript (aim for Medium now)
2. **Afternoon:** 1 mock interview (use Pramp.com — it's free and live)
3. **Evening:** Re-read your project code. Pick one file per day and explain EVERY line to yourself.

**Files to prioritize for re-reading:**
- `claimController.js` — the IoU algorithm, the FormData building, the MongoDB save
- `authMiddleware.js` — the middleware chain, jwt.verify, req.user
- `AuthContext.jsx` — localStorage, axios defaults, the login/logout cycle
- `server.js` — the exact middleware order and why each piece is there

---

## Quick Reference: Things Interviewers Ask About MERN

| Question | Answer Points |
|---|---|
| "What is the event loop?" | Call stack → microtask queue (Promises) → task queue (setTimeout) |
| "REST vs GraphQL" | REST: multiple endpoints, fixed structure. GraphQL: one endpoint, client specifies exactly what data it needs |
| "What is middleware?" | Functions in Express that handle req/res before the controller. Can modify req, res, or call next() |
| "SQL vs NoSQL" | SQL: structured table/row, relations, ACID. NoSQL: Document/key-value, flexible schema, horizontal scale |
| "What is a JWT?" | Signed token = base64(header) + base64(payload) + signature. Stateless — server doesn't store session |
| "What is bcrypt?" | One-way hash function for passwords. Can't reverse. compare() verifies without storing original |
| "What is CORS?" | Browser security: blocks requests to a different domain unless that domain explicitly allows it |
| "Virtual DOM vs Real DOM?" | Virtual DOM is a JS object representation. React diffs it and only writes necessary real DOM changes |
| "Why hooks instead of classes?" | Simpler mental model, easier to share logic (custom hooks), no `this` confusion |
| "What is useEffect?" | Side effect runner. Runs after render. Dependencies array controls when it re-runs |

---

## Resources (Minimal — Only the Best)

| Resource | Purpose |
|---|---|
| [javascript.info](https://javascript.info) | Best JS fundamentals reference anywhere |
| [NeetCode.io](https://neetcode.io) | Structured DSA practice with video explanations |
| [react.dev](https://react.dev) | Official React docs — read the "Thinking in React" guide |
| [ByteByteGo YouTube](https://youtube.com/@ByteByteGo) | System design concepts for free |
| [Pramp.com](https://pramp.com) | Free live mock interviews (peer-to-peer) |

---

## Stop Doing This

- ❌ Building new projects instead of deeply understanding CrashCost V2
- ❌ Watching tutorials while feeling productive (you're not writing code)
- ❌ Skipping DSA because "I'm a web dev" — every company tests it
- ❌ Not reading error messages fully — they always tell you exactly what's wrong
- ❌ Using AI to write code you don't understand — you won't be able to explain it
