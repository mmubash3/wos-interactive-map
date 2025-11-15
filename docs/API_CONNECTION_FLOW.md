# Whiteout API Connection Flow Diagrams

This document provides visual representations of how the application connects to and interacts with the Whiteout API.

## Table of Contents

1. [User Authentication Flow](#user-authentication-flow)
2. [Gift Code Redemption Flow](#gift-code-redemption-flow)
3. [Component Architecture](#component-architecture)
4. [Request Signing Process](#request-signing-process)

---

## User Authentication Flow

This diagram shows how a user authenticates with their Whiteout Furnace ID (FID).

```
┌─────────────┐
│   Browser   │
│  (Frontend) │
└──────┬──────┘
       │
       │ 1. User enters FID
       │    POST /validate-id
       │    { "id": "123456" }
       ↓
┌──────────────────────────────────────┐
│   Express Backend                     │
│   wos-interactive-map-back/          │
│   src/index.js                       │
└──────┬───────────────────────────────┘
       │
       │ 2. Generate signature
       │    time = Date.now()
       │    params = "fid=123456&time={time}"
       │    sign = MD5(params + SECRET)
       │
       │ 3. POST to Whiteout API
       │    URL: https://wos-giftcode-api.centurygame.com/api/player
       │    Body: sign={sign}&fid=123456&time={time}
       ↓
┌──────────────────────────────────────┐
│   Whiteout API                        │
│   wos-giftcode-api.centurygame.com   │
└──────┬───────────────────────────────┘
       │
       │ 4. Validates signature & FID
       │    Returns player data:
       │    {
       │      "code": 0,
       │      "msg": "success",
       │      "data": {
       │        "fid": "123456",
       │        "nickname": "PlayerName",
       │        "stove_lv": "25",
       │        "avatar_image": "...",
       │        ...
       │      }
       │    }
       ↓
┌──────────────────────────────────────┐
│   Express Backend                     │
└──────┬───────────────────────────────┘
       │
       │ 5. Generate auth token
       │    token = crypto.randomBytes(16).toString('hex')
       │    tokens[token] = { valid: true, expiry: now + 1hr }
       │
       │ 6. Save/Update user in MongoDB
       │    User.findOneAndUpdate({
       │      id: fid,
       │      nom: nickname,
       │      lvl: stove_lv,
       │      ...
       │    })
       │
       │ 7. Return to frontend
       │    {
       │      "success": true,
       │      "token": "abc123...",
       │      "playerData": {...}
       │    }
       ↓
┌──────────────┐
│   Browser    │
│              │
│ - Stores     │
│   token in   │
│   cookie     │
│ - Shows user │
│   profile    │
└──────────────┘
```

---

## Gift Code Redemption Flow

This diagram shows the complete process of redeeming a gift code, including CAPTCHA solving.

```
┌─────────────┐
│   Browser   │
└──────┬──────┘
       │
       │ 1. User submits gift code
       │    POST /api/redeem
       │    { "giftcode": "ABCD1234" }
       │    Authorization: Bearer {token}
       ↓
┌────────────────────────────────────────────────┐
│   Express Backend                               │
│   wos-interactive-map-back/src/routes/         │
│   redeemRoutes.js                              │
└──────┬─────────────────────────────────────────┘
       │
       │ 2. Create background job
       │    jobId = createJob(giftcode, userCount)
       │
       │ 3. Return immediately
       │    { "success": true, "jobId": "..." }
       ↓
┌─────────────┐
│   Browser   │
│             │
│ Shows       │
│ "Processing"│
└─────────────┘

┌────────────────────────────────────────────────┐
│   Background Process                            │
│   wos-interactive-map-back/src/services/       │
│   redeem.js                                    │
└──────┬─────────────────────────────────────────┘
       │
       │ For each user in database:
       │
       ├──────────────────────────────────────────┐
       │                                           │
       │ 4. LOGIN PHASE                           │
       │    ─────────────                         │
       │    POST to Whiteout player API           │
       │    params = { fid, time }                │
       │    sign = MD5(sorted_params + SECRET)    │
       ↓                                           │
┌──────────────────────────────────────┐          │
│   Whiteout API                        │          │
│   /api/player                         │          │
└──────┬───────────────────────────────┘          │
       │                                           │
       │ Returns: { "msg": "success" }            │
       ↓                                           │
┌────────────────────────────────────────────────┤
│   Background Process                            │
│                                                 │
│ 5. CAPTCHA PHASE                               │
│    ─────────────                               │
│    POST to Whiteout captcha API                │
│    params = { fid, time }                      │
│    sign = MD5(sorted_params + SECRET)          │
└──────┬──────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────┐
│   Whiteout API                        │
│   /api/captcha                        │
└──────┬───────────────────────────────┘
       │
       │ Returns: {
       │   "data": {
       │     "img": "data:image/png;base64,..."
       │   }
       │ }
       ↓
┌────────────────────────────────────────────────┐
│   Background Process                            │
│                                                 │
│ 6. SOLVE CAPTCHA                               │
│    ─────────────                               │
│    Extract base64 image                        │
│    POST to solver service                      │
│    URL: http://solver:5001/solve              │
│    Body: FormData with image file              │
└──────┬──────────────────────────────────────────┘
       ↓
┌────────────────────────────────────────────────┐
│   Python CAPTCHA Solver                        │
│   wos-gift-code-service/                       │
│   solver_api.py                                │
│                                                 │
│ - Receives image                               │
│ - Runs ML model                                │
│ - Recognizes text                              │
└──────┬──────────────────────────────────────────┘
       │
       │ Returns: {
       │   "text": "A3K9",
       │   "success": true,
       │   "confidence": 0.95
       │ }
       ↓
┌────────────────────────────────────────────────┐
│   Background Process                            │
│                                                 │
│ 7. REDEEM GIFT CODE                            │
│    ────────────────                            │
│    POST to Whiteout gift_code API              │
│    params = {                                  │
│      fid,                                      │
│      cdk: "ABCD1234",                          │
│      captcha_code: "A3K9",                     │
│      time                                      │
│    }                                           │
│    sign = MD5(sorted_params + SECRET)          │
└──────┬──────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────┐
│   Whiteout API                        │
│   /api/gift_code                      │
└──────┬───────────────────────────────┘
       │
       │ Returns one of:
       │ - "success" - Code redeemed
       │ - "CAPTCHA CHECK ERROR." - Retry
       │ - "CAPTCHA EXPIRED." - Retry
       │ - "TIME_ERROR" - Code expired
       │ - "CDK_NOT_FOUND" - Invalid code
       │ - "USAGE_LIMIT" - Already used
       ↓
┌────────────────────────────────────────────────┐
│   Background Process                            │
│                                                 │
│ 8. HANDLE RESPONSE                             │
│    ───────────────                             │
│    If retriable error (CAPTCHA errors):        │
│      - Sleep with exponential backoff          │
│      - Retry from step 4 (up to 50 times)      │
│                                                 │
│    If success or non-retriable error:          │
│      - Record result                           │
│      - Update job progress                     │
│      - Move to next user                       │
│                                                 │
│ 9. After all users processed:                  │
│    - Update job status to "completed"          │
│    - Store final results                       │
└─────────────────────────────────────────────────┘
```

---

## Component Architecture

This diagram shows how the different components of the system interact.

```
┌───────────────────────────────────────────────────────────────┐
│                         Docker Environment                     │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Frontend Container (React + Vite)                    │   │
│  │  wos-interactive-map-front/                          │   │
│  │                                                        │   │
│  │  - User Interface                                     │   │
│  │  - Authentication forms                               │   │
│  │  - Map visualization                                  │   │
│  │  - Gift code management UI                            │   │
│  └─────────────────────┬──────────────────────────────────┘   │
│                        │                                       │
│                        │ HTTP Requests                         │
│                        │ (API calls)                           │
│                        ↓                                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Backend Container (Node.js + Express)               │   │
│  │  wos-interactive-map-back/                          │   │
│  │                                                        │   │
│  │  Components:                                          │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  src/index.js                               │   │   │
│  │  │  - Main server                              │   │   │
│  │  │  - Authentication middleware                │   │   │
│  │  │  - Token management                         │   │   │
│  │  │  - User validation endpoint                 │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  src/routes/                                │   │   │
│  │  │  - buildings.js                             │   │   │
│  │  │  - guilds.js                                │   │   │
│  │  │  - user.js                                  │   │   │
│  │  │  - redeemRoutes.js ← Gift code endpoint    │   │   │
│  │  │  - jobs.js                                  │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  src/services/                              │   │   │
│  │  │  - redeem.js ← Gift code logic              │   │   │
│  │  │  - jobStore.js ← Job tracking               │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  src/models/                                │   │   │
│  │  │  - User.js ← MongoDB user schema            │   │   │
│  │  │  - Building.js                              │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └──────┬───────────────────────┬────────────────────┘   │
│         │                       │                         │
│         │                       │ HTTP Requests           │
│         │                       │ (Docker network)        │
│         │                       ↓                         │
│         │  ┌──────────────────────────────────────────┐  │
│         │  │  CAPTCHA Solver Container (Python)       │  │
│         │  │  wos-gift-code-service/                 │  │
│         │  │                                          │  │
│         │  │  - solver_api.py (FastAPI server)       │  │
│         │  │  - gift_captchasolver.py (ML models)    │  │
│         │  │  - Port: 5001                           │  │
│         │  │                                          │  │
│         │  │  Receives: CAPTCHA images               │  │
│         │  │  Returns: Solved text                   │  │
│         │  └──────────────────────────────────────────┘  │
│         │                                                 │
│         │ MongoDB Connection                              │
│         ↓                                                 │
│  ┌──────────────────────────────────────────────────┐   │
│  │  MongoDB Container                                │   │
│  │                                                    │   │
│  │  Database: mapBuildings                          │   │
│  │  Collections:                                     │   │
│  │  - users (player data)                           │   │
│  │  - buildings (map data)                          │   │
│  │  - guilds (guild data)                           │   │
│  └──────────────────────────────────────────────────┘   │
│                                                            │
└───────────────────────────────────────────────────────────┘
                           │
                           │ HTTPS Requests
                           │ (External API calls)
                           ↓
         ┌──────────────────────────────────────┐
         │  Whiteout API (External)              │
         │  wos-giftcode-api.centurygame.com    │
         │                                       │
         │  Endpoints:                           │
         │  - /api/player (authentication)       │
         │  - /api/captcha (get captcha)         │
         │  - /api/gift_code (redeem codes)      │
         └───────────────────────────────────────┘
```

---

## Request Signing Process

This diagram explains how requests are signed for authentication with the Whiteout API.

```
┌─────────────────────────────────────────────────────────────┐
│  Request Signing Algorithm                                   │
│  (Used for all Whiteout API calls)                          │
└─────────────────────────────────────────────────────────────┘

Step 1: Prepare Parameters
──────────────────────────
┌────────────────────────┐
│  Request Parameters:   │
│  {                     │
│    fid: "123456",      │
│    time: 1700000000,   │
│    cdk: "ABCD1234"     │ (optional, for gift codes)
│  }                     │
└────────┬───────────────┘
         │
         ↓

Step 2: Sort Keys Alphabetically
─────────────────────────────────
┌────────────────────────┐
│  Sorted keys:          │
│  ["cdk", "fid", "time"]│
└────────┬───────────────┘
         │
         ↓

Step 3: Create Parameter String
────────────────────────────────
┌────────────────────────────────────┐
│  params = keys.map(k =>            │
│    `${k}=${data[k]}`).join("&")    │
│                                    │
│  Result:                           │
│  "cdk=ABCD1234&fid=123456&time=..."│
└────────┬───────────────────────────┘
         │
         ↓

Step 4: Append Secret Key
──────────────────────────
┌────────────────────────────────────┐
│  signString = params + SECRET      │
│                                    │
│  Result:                           │
│  "cdk=ABCD1234&fid=123456&..."     │
│  + "tB87#kPtkxqOS2"                │
│                                    │
│  = "cdk=ABCD1234&fid=123456&..."   │
│    "time=1700000000tB87#kPtkxqOS2" │
└────────┬───────────────────────────┘
         │
         ↓

Step 5: Generate MD5 Hash
──────────────────────────
┌────────────────────────────────────┐
│  sign = MD5(signString)            │
│                                    │
│  Example result:                   │
│  "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"│
└────────┬───────────────────────────┘
         │
         ↓

Step 6: Build Request Body
───────────────────────────
┌────────────────────────────────────┐
│  Final request body:               │
│  {                                 │
│    sign: "a1b2c3d4...",            │
│    cdk: "ABCD1234",                │
│    fid: "123456",                  │
│    time: 1700000000                │
│  }                                 │
│                                    │
│  As URL-encoded:                   │
│  "sign=a1b2c3d4...&cdk=ABCD1234&"  │
│  "fid=123456&time=1700000000"      │
└────────┬───────────────────────────┘
         │
         ↓

Step 7: Send to Whiteout API
─────────────────────────────
┌────────────────────────────────────┐
│  POST https://wos-giftcode-api...  │
│  Content-Type:                     │
│    application/x-www-form-urlencoded│
│  Body: sign=...&cdk=...&fid=...   │
└────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────┐
│  Implementation Examples                                     │
└─────────────────────────────────────────────────────────────┘

Backend (Node.js):
──────────────────
function encodeData(data) {
  const sortedKeys = Object.keys(data).sort();
  const encoded = sortedKeys
    .map(k => `${k}=${data[k]}`)
    .join("&");
  const sign = crypto
    .createHash("md5")
    .update(encoded + SECRET)
    .digest("hex");
  return { sign, ...data };
}

Frontend (JavaScript with CryptoJS):
─────────────────────────────────────
const params = `fid=${fid}&time=${time}`;
const signString = params + secretKey;
const sign = CryptoJS.MD5(signString)
  .toString(CryptoJS.enc.Hex);


┌─────────────────────────────────────────────────────────────┐
│  Security Notes                                              │
└─────────────────────────────────────────────────────────────┘

1. The SECRET key must be kept confidential
2. The timestamp (time) prevents replay attacks
3. The signature proves the request hasn't been tampered with
4. All parameters must be included in the signature calculation
5. Parameters must be sorted alphabetically for consistent hashing
```

---

## Data Flow Summary

### Quick Reference Table

| Step | Component | Action | Data Flow |
|------|-----------|--------|-----------|
| 1 | Frontend | User enters FID | → Backend |
| 2 | Backend | Generate signature | Internal |
| 3 | Backend | Validate FID | → Whiteout API |
| 4 | Whiteout API | Return player data | → Backend |
| 5 | Backend | Generate token | Internal |
| 6 | Backend | Save user to DB | → MongoDB |
| 7 | Backend | Return token | → Frontend |
| 8 | Frontend | Store token | Internal |

### Gift Code Redemption Table

| Step | Component | Action | Data Flow |
|------|-----------|--------|-----------|
| 1 | Frontend | Submit gift code | → Backend |
| 2 | Backend | Create job | Internal |
| 3 | Backend | Login to Whiteout | → Whiteout API |
| 4 | Backend | Fetch CAPTCHA | → Whiteout API |
| 5 | Backend | Send to solver | → Python Solver |
| 6 | Python Solver | Solve CAPTCHA | → Backend |
| 7 | Backend | Redeem code | → Whiteout API |
| 8 | Backend | Handle response | Internal |
| 9 | Backend | Retry if needed | Loop to step 3 |
| 10 | Backend | Complete job | → Frontend (polling) |

---

## Connection Endpoints Summary

### Backend API Endpoints

```
Authentication:
  POST /validate-id
    Body: { "id": "123456" }
    Response: { "success": true, "token": "...", "playerData": {...} }

Gift Code Management:
  POST /api/redeem
    Headers: Authorization: Bearer {token}
    Body: { "giftcode": "ABCD1234" }
    Response: { "success": true, "jobId": "..." }

  GET /api/jobs/:jobId
    Headers: Authorization: Bearer {token}
    Response: { "status": "completed", "results": [...] }

User Management:
  GET /api/users
    Headers: Authorization: Bearer {token}
    Response: { "users": [...] }

  PUT /api/users/:id
    Headers: Authorization: Bearer {token}
    Body: { "nom": "...", "power": 123, ... }
    Response: { "user": {...} }
```

### Whiteout API Endpoints

```
Player Authentication:
  POST https://wos-giftcode-api.centurygame.com/api/player
    Content-Type: application/x-www-form-urlencoded
    Body: sign={md5}&fid={id}&time={timestamp}
    Response: { "code": 0, "msg": "success", "data": {...} }

CAPTCHA Retrieval:
  POST https://wos-giftcode-api.centurygame.com/api/captcha
    Content-Type: application/x-www-form-urlencoded
    Body: sign={md5}&fid={id}&time={timestamp}
    Response: { "data": { "img": "data:image/png;base64,..." } }

Gift Code Redemption:
  POST https://wos-giftcode-api.centurygame.com/api/gift_code
    Content-Type: application/x-www-form-urlencoded
    Body: sign={md5}&fid={id}&cdk={code}&captcha_code={text}&time={timestamp}
    Response: { "msg": "success" | "error_type" }
```

### CAPTCHA Solver API

```
Solve CAPTCHA:
  POST http://solver:5001/solve
    Content-Type: multipart/form-data
    Body: FormData { file: <image_bytes> }
    Response: {
      "text": "A3K9",
      "success": true,
      "method": "ml_model",
      "confidence": 0.95
    }
```

---

## Troubleshooting Guide

### Connection Issues

| Problem | Possible Cause | Solution |
|---------|---------------|----------|
| "Invalid or missing token" | Token expired or not provided | Re-authenticate with FID |
| "Sign Error" | Incorrect secret key or parameter order | Check SECRET env variable |
| "CAPTCHA CHECK ERROR" | CAPTCHA solution incorrect | Retry with new CAPTCHA |
| "CAPTCHA_FETCH_ERROR" | Network issue or rate limit | Check solver service status |
| "LOGIN_FAILED" | Invalid FID or API down | Verify FID is correct |
| "SOLVER_FAILED" | Python service unreachable | Check Docker network/container |

### Verification Steps

1. **Check Backend Logs**
   ```bash
   docker logs wos-interactive-map-back
   ```

2. **Check Solver Logs**
   ```bash
   docker logs solver
   ```

3. **Verify MongoDB Connection**
   ```bash
   docker exec -it mongodb mongosh
   use mapBuildings
   db.users.find()
   ```

4. **Test Whiteout API Directly**
   ```bash
   curl -X POST https://wos-giftcode-api.centurygame.com/api/player \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "sign={calculated_md5}&fid={your_fid}&time={timestamp}"
   ```

---

## Additional Resources

- **Main Documentation:** [README.md](../README.md)
- **Technical Details:** [WHITEOUT_API_INTEGRATION.md](./WHITEOUT_API_INTEGRATION.md)
- **Presentation:** [presentation.md](./presentation.md)

For more information, see the source code:
- Backend: `wos-interactive-map-back/src/`
- Frontend: `wos-interactive-map-front/src/`
- Solver: `wos-gift-code-service/`
