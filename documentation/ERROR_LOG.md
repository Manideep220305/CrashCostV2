# CrashCost V2 — Deployment Error Log

A record of every major error encountered during development and deployment, what caused it, and exactly how it was resolved.

---

## Error 1 — Render Build Failed: No package.json found
**When:** First Render deployment attempt  
**Error Message:**
```
npm error code ENOENT
npm error path /opt/render/project/src/package.json
npm error enoent Could not read package.json
```
**Root Cause:** Render was looking for `package.json` at the root of the GitHub repo. But in this project, the backend code (Express) lives in a subfolder called `backend/`, not at the root.  
**Fix:** In the Render Dashboard → "Root Directory" field, set it to `backend`. This tells Render to `cd` into the `backend` folder first before running any commands.

---

## Error 2 — Vercel White Screen: MIME Type Error
**When:** First Vercel deployment attempt  
**Error Message (browser console):**
```
Failed to load module script: Expected a JavaScript-or-Wasm module script
but the server responded with a MIME type of "text/html".
```
**Root Cause:** The `vercel.json` file had a routing rule:
```json
"routes": [{ "src": "/(.*)", "dest": "/index.html" }]
```
This route is meant to support React Router (so refreshing `/dashboard` doesn't 404). However, `routes` in Vercel is too aggressive — it intercepts **every** request, including the request for the bundled JavaScript file (`/assets/index-XYZ.js`). Vercel was returning an HTML page instead of a JavaScript file. The browser received HTML, tried to execute it as JS, and crashed.  
**Fix:** Replaced `routes` with `rewrites`:
```json
"rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
```
`rewrites` is non-destructive. It only kicks in when the requested file does not physically exist on disk. Static assets like `.js` and `.css` files exist in the `dist/assets/` folder, so Vercel serves them directly. Routes like `/dashboard` don't exist as files, so Vercel falls back to `index.html` correctly.

---

## Error 3 — CORS Blocked: Registration/Login Failing on Live Site
**When:** After both Vercel and Render were successfully deployed  
**Error Message (browser console):**
```
Access to XMLHttpRequest at 'https://crashcostv2-1.onrender.com/api/auth/register'
from origin 'https://crash-cost-v2.vercel.app' has been blocked by CORS policy.
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```
**Root Cause:** During the pre-deployment audit, the backend's CORS policy was tightened from open (`app.use(cors())`) to a strict allowlist that reads the `FRONTEND_URL` environment variable. In production, that env variable wasn't set yet on Render, so the allowlist was essentially empty — blocking the Vercel domain.  
**Fix:** Added the following environment variable in the Render dashboard:
- **Key:** `FRONTEND_URL`
- **Value:** `https://crash-cost-v2.vercel.app`

After Render restarted, the backend's CORS middleware allowed traffic from the live Vercel domain.

---

## Error 4 — Git Commit Failed: Pathspec Error
**When:** Running `git commit "message"` without the `-m` flag  
**Error Message:**
```
error: pathspec 'Deleted some files' did not match any file(s) known to git
```
**Root Cause:** The correct syntax for a git commit message is `git commit -m "message"`. Without `-m`, git tries to interpret the message as a file path.  
**Fix:** Use the correct syntax: `git commit -m "your message here"`

---

## Error 5 — Duplicate Detections Inflating Repair Cost
**When:** Testing the AI pipeline  
**Issue:** YOLO instance segmentation was outputting multiple overlapping bounding boxes for the same piece of damage. Since each detection was priced independently, duplicate detections were added multiple times to the total estimate, causing the cost to be 2-3x the real value.  
**Root Cause:** YOLO's Non-Maximum Suppression (NMS) wasn't aggressive enough for this use case. It's designed to remove boxes that are nearly identical, but boxes with slight offsets on the same damage region would survive NMS as separate detections.  
**Fix:** Implemented a post-processing deduplication filter in `backend/controllers/claimController.js` using Intersection over Union (IoU) and containment checks. Any two detections of the same label where IoU > 30% or one box is >50% inside the other are considered the same damage. Only the first one is kept; the rest are discarded. The total estimate is then recalculated from the unique detections only.

---

## Error 6 — Frontend API Calls Failing in Production (Vite Proxy)
**When:** Pre-deployment audit  
**Issue:** All 5 API calls in the React frontend used relative paths like `/api/segment-car`. This worked in development because Vite's dev server has a proxy that rewrites `/api/*` to `http://localhost:5000/*`. But Vercel doesn't run Vite's dev server — it just hosts the compiled static files. There is no proxy, so `/api/segment-car` would just try to hit `https://crash-cost-v2.vercel.app/api/segment-car`, which doesn't exist.  
**Fix:** Replaced every hardcoded relative path with:
```javascript
const API_URL = import.meta.env.VITE_API_URL || '';
const response = await fetch(`${API_URL}/api/segment-car`, ...);
```
And set `VITE_API_URL=https://crashcostv2-1.onrender.com` as an environment variable in Vercel. At build time, Vite replaces `import.meta.env.VITE_API_URL` with the real URL string.

---

## Error 7 — node_modules and Large Files Tracked in Git
**When:** Initial GitHub push  
**Issue:** The GitHub repository contained `node_modules/`, the YOLO model file `best.pt` (60MB), the CatBoost model `crashcost_pricing_model.cbm` (4MB), and the training dataset `insurevision_pricing_data_v4.csv` (1.3MB). These should never be in version control — they slow down cloning and can expose training data.  
**Root Cause:** No `.gitignore` files existed at the project root or in the individual `frontend/` and `backend/` subdirectories.  
**Fix:**
1. Created `.gitignore` files in root, `frontend/`, and `backend/` directories.
2. Ran `git rm -r --cached .` to remove everything from the git index (does NOT delete files locally).
3. Ran `git add .` so git re-staged everything, this time respecting `.gitignore`.
4. Committed and pushed. The large binary files were deleted from GitHub automatically.
