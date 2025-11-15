# Whiteout API Integration Documentation

## Overview

This document explains how the WOS Interactive Map application connects to the Whiteout gift code API (`wos-giftcode-api.centurygame.com`) to authenticate users and redeem gift codes.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Authentication Flow](#authentication-flow)
3. [API Endpoints](#api-endpoints)
4. [Security Implementation](#security-implementation)
5. [Gift Code Redemption](#gift-code-redemption)
6. [CAPTCHA Solving](#captcha-solving)

---

## Architecture Overview

The application uses a multi-tier architecture to interact with the Whiteout API:

```
Frontend (React) 
    ↓
Backend API (Node.js/Express)
    ↓
Whiteout API (wos-giftcode-api.centurygame.com)
    ↓
Python CAPTCHA Solver Service
```

### Key Components

1. **Frontend** (`wos-interactive-map-front/`)
   - User interface for authentication
   - Gift code management UI
   - Calls backend API endpoints

2. **Backend** (`wos-interactive-map-back/`)
   - Express.js server
   - Authentication middleware
   - API proxy to Whiteout services

3. **CAPTCHA Solver** (`wos-gift-code-service/`)
   - Python FastAPI service
   - ML-based CAPTCHA solving
   - Runs as a separate container

---

## Authentication Flow

### 1. User ID Validation

The authentication process validates a user's Whiteout ID (FID) with the official API:

#### Backend Implementation (`wos-interactive-map-back/src/index.js`)

```javascript
// Line 90-103
async function validateWithWhiteoutAPI(id) {
  const time = Date.now();
  const form = `fid=${id}&time=${time}`;
  const sign = crypto.createHash('md5').update(form + SECRET).digest('hex');
  const body = `sign=${sign}&${form}`;

  const response = await fetch(API_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body,
  });

  return response.json();
}
```

#### Flow Diagram

```
User submits FID → Backend receives request
                          ↓
                   Generate timestamp
                          ↓
                   Create signature: MD5(fid={FID}&time={timestamp}{SECRET})
                          ↓
                   POST to https://wos-giftcode-api.centurygame.com/api/player
                          ↓
                   Receive player data
                          ↓
                   Store/Update user in MongoDB
                          ↓
                   Generate auth token
                          ↓
                   Return token to frontend
```

### 2. Signature Generation

The API requires request signing for security:

**Formula:**
```
sign = MD5(parameters_string + SECRET_KEY)
```

**Example:**
```javascript
const SECRET = 'tB87#kPtkxqOS2';  // Default secret
const params = `fid=123456&time=1700000000`;
const sign = crypto.createHash('md5').update(params + SECRET).digest('hex');
```

### 3. User Data Returned

Upon successful validation, the API returns:

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "fid": "123456",
    "kid": "789",
    "nickname": "PlayerName",
    "stove_lv": "25",
    "avatar_image": "https://...",
    "stove_lv_content": "https://..."
  }
}
```

### 4. Session Management

After validation:
- Backend generates a random token (32 hex characters)
- Token stored in memory with 1-hour expiration
- Token sent to frontend as cookie
- User data saved/updated in MongoDB

---

## API Endpoints

### Whiteout API Endpoints

#### 1. Player Information/Login
- **URL:** `https://wos-giftcode-api.centurygame.com/api/player`
- **Method:** POST
- **Content-Type:** `application/x-www-form-urlencoded`
- **Parameters:**
  - `fid` - Player's Furnace ID
  - `time` - Current timestamp
  - `sign` - MD5 signature

#### 2. CAPTCHA Retrieval
- **URL:** `https://wos-giftcode-api.centurygame.com/api/captcha`
- **Method:** POST/GET
- **Parameters:**
  - `fid` - Player's Furnace ID
  - `time` - Current timestamp
  - `sign` - MD5 signature
  - `init` - Initialization flag (optional)

#### 3. Gift Code Redemption
- **URL:** `https://wos-giftcode-api.centurygame.com/api/gift_code`
- **Method:** POST
- **Parameters:**
  - `fid` - Player's Furnace ID
  - `cdk` - Gift code
  - `captcha_code` - Solved CAPTCHA text
  - `time` - Current timestamp
  - `sign` - MD5 signature

### Backend API Endpoints

#### 1. Validate User ID
- **URL:** `/validate-id`
- **Method:** POST
- **Public:** Yes
- **Request Body:**
  ```json
  {
    "id": "123456"
  }
  ```
- **Response:**
  ```json
  {
    "success": true,
    "token": "abc123...",
    "playerData": { /* user data */ }
  }
  ```

#### 2. Redeem Gift Code
- **URL:** `/api/redeem`
- **Method:** POST
- **Protected:** Yes (requires auth token)
- **Request Body:**
  ```json
  {
    "giftcode": "ABCD1234"
  }
  ```
- **Response:**
  ```json
  {
    "success": true,
    "jobId": "job-uuid",
    "message": "Redemption started"
  }
  ```

---

## Security Implementation

### 1. Request Signing

All requests to Whiteout API must include a valid signature:

```javascript
// Backend: wos-interactive-map-back/src/services/redeem.js
function encodeData(data) {
  const sortedKeys = Object.keys(data).sort();
  const encoded = sortedKeys.map(k => `${k}=${data[k]}`).join("&");
  const sign = crypto.createHash("md5").update(encoded + SECRET).digest("hex");
  return { sign, ...data };
}
```

### 2. Secret Key

The secret key is configurable via environment variable:

```javascript
const SECRET = process.env.SECRET || 'tB87#kPtkxqOS2';
```

**Configuration:**
- Set in `.env` file: `SECRET=your_secret_key`
- Default value if not set: `tB87#kPtkxqOS2`

### 3. Token Authentication

After user validation:
- Backend generates a secure random token
- Token has 1-hour expiration
- Stored in-memory on backend
- Sent to frontend as HTTP-only cookie
- Required for all protected API routes

### 4. CORS Protection

```javascript
const allowedHost = process.env.ALLOWED_HOST || "";

// Middleware checks host header
if (!host || !host.includes(allowedHost)) {
  return res.redirect(301, redirectUrl);
}
```

---

## Gift Code Redemption

The gift code redemption process is more complex due to CAPTCHA requirements.

### High-Level Flow

```
1. User requests redemption
2. For each user in database:
   a. Login to Whiteout API
   b. Fetch CAPTCHA image
   c. Send CAPTCHA to solver service
   d. Receive solved text
   e. Submit gift code with CAPTCHA
   f. Handle retry logic for failures
3. Return results
```

### Detailed Implementation

#### 1. Login Phase

```javascript
// wos-interactive-map-back/src/services/redeem.js (Line 22-34)
const loginPayload = encodeData({ fid, time: Math.floor(Date.now() / 1000) });
const loginResp = await axios.post(WOS_PLAYER_INFO_URL, new URLSearchParams(loginPayload));

if (loginResp.data.msg !== "success") {
  return { fid, status: "LOGIN_FAILED" };
}
```

#### 2. CAPTCHA Fetch

```javascript
// Line 37-44
const captchaPayload = encodeData({ fid, time: Date.now() });
const captchaResp = await axios.post(WOS_CAPTCHA_URL, new URLSearchParams(captchaPayload));
const captchaBase64 = captchaResp.data?.data?.img;

if (!captchaBase64) return { fid, status: "CAPTCHA_FETCH_ERROR" };
```

#### 3. CAPTCHA Solving

```javascript
// Line 47-55
const base64Data = captchaBase64.replace(/^data:image\/\w+;base64,/, "");
const form = new FormData();
form.append("file", Buffer.from(base64Data, "base64"), "captcha.png");

const solverResp = await axios.post(SOLVER_URL, form, { headers: form.getHeaders() });
if (!solverResp.data.success) return { fid, status: "SOLVER_FAILED" };
const captchaCode = solverResp.data.text;
```

#### 4. Gift Code Submission

```javascript
// Line 59-68
const redeemPayload = encodeData({
  fid,
  cdk: giftcode,
  captcha_code: captchaCode,
  time: Date.now()
});
const redeemResp = await axios.post(WOS_GIFTCODE_URL, new URLSearchParams(redeemPayload));
return { fid, status: redeemResp.data.msg, response: redeemResp.data };
```

#### 5. Retry Logic

```javascript
// Line 79-101
async function redeemWithRetry(fid, giftcode, retries = 50) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    const res = await redeemGiftCodeForUser(fid, giftcode);

    // Retry for rate limit OR captcha errors
    if (
      res.status === "ERROR" ||
      res.status === "CAPTCHA CHECK ERROR." ||
      res.status === "CAPTCHA CHECK TOO FREQUENT." ||
      res.status === "SOLVER_FAILED" ||
      res.status === "Sign Error" || 
      res.status === "CAPTCHA EXPIRED."
    ) {
      console.log(`Retrying for ${fid} due to ${res.status} (attempt ${attempt})...`);
      await sleep(attempt * 1000); // exponential backoff
      continue;
    }

    return res; // success or other non-retriable failure
  }

  return { fid, status: "FAILED_AFTER_RETRIES" };
}
```

### Response Status Codes

The Whiteout API can return various status messages:

| Status | Meaning | Action |
|--------|---------|--------|
| `success` | Code redeemed successfully | Complete |
| `TIME_ERROR` | Code expired | Mark as invalid |
| `CDK_NOT_FOUND` | Code doesn't exist | Mark as invalid |
| `USAGE_LIMIT` | Code already used | Mark as invalid |
| `CAPTCHA CHECK ERROR.` | CAPTCHA failed | Retry |
| `CAPTCHA CHECK TOO FREQUENT.` | Rate limited | Retry with backoff |
| `CAPTCHA EXPIRED.` | CAPTCHA timeout | Retry |
| `Sign Error` | Invalid signature | Retry |

---

## CAPTCHA Solving

### Architecture

The application includes a Python-based ML service for solving CAPTCHAs:

**Location:** `wos-gift-code-service/`

### CAPTCHA Solver Service

#### FastAPI Server (`solver_api.py`)

```python
from fastapi import FastAPI, UploadFile, File
from gift_captchasolver import GiftCaptchaSolver

app = FastAPI(title="Gift Captcha Solver API")
solver = GiftCaptchaSolver(save_images=0)

@app.post("/solve")
async def solve(file: UploadFile = File(...)):
    image_bytes = await file.read()
    text, success, method, confidence, _ = await solver.solve_captcha(image_bytes)
    
    return JSONResponse({
        "text": text,
        "success": success,
        "method": method,
        "confidence": confidence
    })
```

**Features:**
- Runs on port 5001
- Accepts image uploads
- Returns solved text with confidence score
- Uses ML models for character recognition

### Integration with Backend

The backend calls the solver via HTTP:

```javascript
const SOLVER_URL = "http://solver:5001/solve";

// Send CAPTCHA image to solver
const form = new FormData();
form.append("file", Buffer.from(base64Data, "base64"), "captcha.png");
const solverResp = await axios.post(SOLVER_URL, form, { headers: form.getHeaders() });
```

**Docker Networking:**
- Backend container: `wos-interactive-map-back`
- Solver container: `solver`
- Connected via Docker network
- Backend accesses solver at `http://solver:5001`

---

## Frontend Integration

### User Authentication

**File:** `wos-interactive-map-front/src/crud/users.js`

The frontend interacts with Whiteout API both directly and through the backend:

#### 1. Backend-Mediated Authentication (Recommended)

This is handled automatically when users log in through the application UI. The frontend sends the FID to the backend, which validates it and returns a token.

#### 2. Direct API Calls (Legacy)

The frontend also contains code for direct Whiteout API calls:

```javascript
// Lines 64-103: Direct player login
async function loginPlayer(user) {
    const loginUrl = "https://wos-giftcode-api.centurygame.com/api/player";
    const fid = user.id;
    const time = Date.now().toString();
    const secretKey = "tB87#kPtkxqOS2";
    const signString = `fid=${fid}&time=${time}${secretKey}`;
    const sign = CryptoJS.MD5(signString).toString(CryptoJS.enc.Hex);
    
    // ... POST request to loginUrl
}
```

**Note:** This approach requires CryptoJS library in the browser and exposes the secret key. The backend-mediated approach is more secure.

### Gift Code Distribution

**File:** `wos-interactive-map-front/src/crud/users.js` (Lines 146-249)

The frontend can send gift codes to multiple users:

```javascript
async function sendGiftCodeToUsers(users, giftCode) {
    for (const user of users) {
        // Login player
        let loginResult = await loginPlayer(user);
        
        // Fetch captcha
        let captchaResult = await getCaptcha(user);
        
        // Send gift code
        const data = new URLSearchParams();
        data.append('sign', sign);
        data.append('fid', fid);
        data.append('cdk', giftCode);
        data.append('time', time);
        
        const response = await fetch(url, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded',
                'Accept': 'application/json, text/plain, */*',
            },
            body: data.toString(),
        });
    }
}
```

**Features:**
- Progress bar showing completion
- Timer display for estimated completion
- Retry logic for rate limiting (429 errors)
- 60-second backoff on rate limits

---

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
# MongoDB Connection
MONGO_URI=mongodb://localhost:27017/mapBuildings

# Server Configuration
PORT=5000
ALLOWED_HOST=localhost

# Whiteout API Configuration
API_URL=https://wos-giftcode-api.centurygame.com/api/player
SECRET=tB87#kPtkxqOS2
```

### Docker Configuration

If using Docker, the CAPTCHA solver service must be accessible:

```yaml
# docker-compose.yml example
services:
  backend:
    # ...
    environment:
      - SOLVER_URL=http://solver:5001
  
  solver:
    # ...
    ports:
      - "5001:5001"
```

---

## Error Handling

### Common Errors

1. **Invalid Signature**
   - Cause: Incorrect secret key or parameter ordering
   - Solution: Verify SECRET environment variable matches expected value

2. **Rate Limiting**
   - Cause: Too many requests to Whiteout API
   - Solution: Implement exponential backoff (already implemented)

3. **CAPTCHA Solver Unreachable**
   - Cause: Solver service not running or network issue
   - Solution: Ensure solver container is running and accessible

4. **Invalid FID**
   - Cause: User ID doesn't exist in Whiteout system
   - Solution: User must provide valid Furnace ID from game

### Debugging

Enable detailed logging:

```javascript
// Backend logs API responses
console.log("login:", loginResp.data.msg);
console.log("captcha img:", captchaBase64 ? 'ok' : 'failed');
console.log("captcha:", captchaCode);
console.log("status:", redeemResp.data.msg);
```

---

## Rate Limiting & Best Practices

### Whiteout API Limits

- The API has rate limiting (returns 429 when exceeded)
- CAPTCHA checks can fail if too frequent
- Recommended delay between requests: 500-2000ms

### Implementation

```javascript
// Small delay to avoid flooding
await sleep(500);

// Exponential backoff for retries
await sleep(attempt * 1000);
```

### Best Practices

1. **Always use retry logic** for CAPTCHA errors
2. **Implement delays** between bulk operations
3. **Cache authentication** tokens (1 hour expiry)
4. **Monitor API responses** for rate limit indicators
5. **Use backend proxy** instead of direct frontend calls

---

## Security Considerations

### Secret Key Management

⚠️ **Important:** The secret key should be kept confidential.

- Store in environment variables, not in code
- Use different keys for development and production
- Never commit `.env` files to version control
- Consider rotating keys periodically

### CORS and Host Validation

The backend validates the request origin:

```javascript
const allowedHost = process.env.ALLOWED_HOST || "";
if (!host || !host.includes(allowedHost)) {
  return res.redirect(301, redirectUrl);
}
```

This prevents unauthorized domains from using your backend API.

### Token Security

- Tokens expire after 1 hour
- Stored in-memory (not persisted)
- Can be sent as cookie or Authorization header
- Protected routes require valid token

---

## Summary

The WOS Interactive Map connects to Whiteout's official gift code API through a three-tier architecture:

1. **Frontend** - User interface for authentication and gift code management
2. **Backend** - Express.js API that proxies requests to Whiteout, handles authentication, and manages user sessions
3. **CAPTCHA Solver** - Python ML service that solves CAPTCHAs required for gift code redemption

Key security features include:
- MD5-signed requests with secret key
- Token-based session management
- Host validation for CORS protection
- Retry logic with exponential backoff

The integration allows users to authenticate with their Whiteout Furnace ID and programmatically redeem gift codes for multiple accounts.
