# Documentation Index

This directory contains comprehensive documentation for the WOS Interactive Map application.

## Available Documentation

### 📘 [WHITEOUT_API_INTEGRATION.md](./WHITEOUT_API_INTEGRATION.md)
**Comprehensive Technical Documentation**

Complete guide covering the technical implementation of Whiteout API integration:
- Architecture overview
- Authentication flow with code examples
- API endpoints (Whiteout and backend)
- Security implementation details
- Gift code redemption process
- CAPTCHA solving integration
- Configuration guides
- Error handling and troubleshooting

**Best for:** Developers who need to understand the codebase, modify the integration, or debug issues.

---

### 📊 [API_CONNECTION_FLOW.md](./API_CONNECTION_FLOW.md)
**Visual Flow Diagrams and Quick Reference**

Visual representations and diagrams of the system:
- User authentication flow diagram
- Gift code redemption flow diagram
- Component architecture diagram
- Request signing process explanation
- Quick reference tables for endpoints
- Troubleshooting guide

**Best for:** Understanding the system at a glance, onboarding new team members, or explaining to non-technical stakeholders.

---

### 🎨 [presentation.md](./presentation.md)
**Application Presentation**

Screenshots and visual presentation of the application features and user interface.

**Best for:** Understanding what the application looks like and its user-facing features.

---

## Quick Start Guide

### For Understanding the Whiteout Integration

1. **Quick Overview:** Start with the "How It Connects to Whiteout" section in [../README.md](../README.md)
2. **Visual Understanding:** Read [API_CONNECTION_FLOW.md](./API_CONNECTION_FLOW.md) for flow diagrams
3. **Deep Dive:** Refer to [WHITEOUT_API_INTEGRATION.md](./WHITEOUT_API_INTEGRATION.md) for technical details

### For Developers

If you're working on the codebase:

1. **Authentication:**
   - Backend: `wos-interactive-map-back/src/index.js` (lines 90-143)
   - Frontend: `wos-interactive-map-front/src/crud/users.js`
   - See: [WHITEOUT_API_INTEGRATION.md#authentication-flow](./WHITEOUT_API_INTEGRATION.md#authentication-flow)

2. **Gift Code Redemption:**
   - Service: `wos-interactive-map-back/src/services/redeem.js`
   - Routes: `wos-interactive-map-back/src/routes/redeemRoutes.js`
   - See: [WHITEOUT_API_INTEGRATION.md#gift-code-redemption](./WHITEOUT_API_INTEGRATION.md#gift-code-redemption)

3. **CAPTCHA Solver:**
   - API: `wos-gift-code-service/solver_api.py`
   - Solver: `wos-gift-code-service/gift_captchasolver.py`
   - See: [WHITEOUT_API_INTEGRATION.md#captcha-solving](./WHITEOUT_API_INTEGRATION.md#captcha-solving)

### For System Administrators

If you're deploying or configuring the application:

1. **Configuration:** See [WHITEOUT_API_INTEGRATION.md#configuration](./WHITEOUT_API_INTEGRATION.md#configuration)
2. **Security:** See [WHITEOUT_API_INTEGRATION.md#security-implementation](./WHITEOUT_API_INTEGRATION.md#security-implementation)
3. **Troubleshooting:** See [API_CONNECTION_FLOW.md#troubleshooting-guide](./API_CONNECTION_FLOW.md#troubleshooting-guide)

---

## Key Concepts

### What is Whiteout?

Whiteout (Whiteout Survival) is a mobile game. This application integrates with the game's official gift code API to allow players to:
- Authenticate using their in-game Furnace ID (FID)
- Automatically redeem gift codes across multiple accounts
- Manage their alliance/guild information

### How Does the Integration Work?

The application acts as an intermediary between users and the Whiteout API:

```
User → Frontend → Backend → Whiteout API
                    ↓
              CAPTCHA Solver
                    ↓
                 MongoDB
```

All requests to the Whiteout API are signed with MD5 signatures for security. The CAPTCHA solver uses machine learning to automatically solve CAPTCHAs required for gift code redemption.

### Key Features

1. **Secure Authentication:** MD5-signed requests, token-based sessions
2. **Automated Redemption:** Bulk gift code redemption with retry logic
3. **CAPTCHA Solving:** ML-based automatic CAPTCHA solving
4. **Multi-User Support:** Manage and redeem codes for multiple accounts
5. **Interactive Map:** Visualize and manage in-game locations

---

## API Endpoints Quick Reference

### Backend Endpoints

- `POST /validate-id` - Authenticate user with FID
- `POST /api/redeem` - Start gift code redemption
- `GET /api/users` - List all users
- `PUT /api/users/:id` - Update user information

### Whiteout API Endpoints

- `POST https://wos-giftcode-api.centurygame.com/api/player` - Authenticate/get player info
- `POST https://wos-giftcode-api.centurygame.com/api/captcha` - Get CAPTCHA image
- `POST https://wos-giftcode-api.centurygame.com/api/gift_code` - Redeem gift code

### CAPTCHA Solver

- `POST http://solver:5001/solve` - Solve CAPTCHA image

---

## Environment Variables

Key environment variables for configuration:

```env
# MongoDB
MONGO_URI=mongodb://localhost:27017/mapBuildings

# Server
PORT=5000
ALLOWED_HOST=localhost

# Whiteout API
API_URL=https://wos-giftcode-api.centurygame.com/api/player
SECRET=tB87#kPtkxqOS2

# CAPTCHA Solver (Docker)
SOLVER_URL=http://solver:5001
```

See [WHITEOUT_API_INTEGRATION.md#configuration](./WHITEOUT_API_INTEGRATION.md#configuration) for complete details.

---

## Common Issues and Solutions

| Issue | Solution |
|-------|----------|
| Invalid token | Re-authenticate with FID |
| Sign Error | Check SECRET environment variable |
| CAPTCHA errors | Verify solver service is running |
| Rate limiting | Built-in retry logic handles this automatically |

For detailed troubleshooting, see [API_CONNECTION_FLOW.md#troubleshooting-guide](./API_CONNECTION_FLOW.md#troubleshooting-guide).

---

## Contributing to Documentation

When updating documentation:

1. **Technical changes:** Update [WHITEOUT_API_INTEGRATION.md](./WHITEOUT_API_INTEGRATION.md)
2. **Flow changes:** Update diagrams in [API_CONNECTION_FLOW.md](./API_CONNECTION_FLOW.md)
3. **Quick reference:** Update tables and summaries in both documents
4. **Main README:** Update [../README.md](../README.md) if high-level changes occur

---

## Additional Resources

- **Main Repository:** [https://github.com/mmubash3/wos-interactive-map](https://github.com/mmubash3/wos-interactive-map)
- **Lite Version:** [https://github.com/Krozac/wos-interactive-map-lite](https://github.com/Krozac/wos-interactive-map-lite)
- **Live Demo:** [https://whiteout-planner.krozac.fr](https://whiteout-planner.krozac.fr)

---

## Document History

- **Initial Creation:** November 2024
- **Purpose:** Document Whiteout API integration in response to user request
- **Coverage:** Complete authentication flow, gift code redemption, and CAPTCHA solving
