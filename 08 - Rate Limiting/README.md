# PulseVote – Rate Limiting (Login & Register)

## Research
Before you code, spend 10–15 minutes exploring what rate limiting is and why it matters.

Answer briefly:
- What is rate limiting?
- Why is it critical for authentication endpoints?
- What is the difference between per‑IP and per‑identifier (e.g., email) limits?
- How can reverse proxies/load balancers affect `req.ip` and rate limiting accuracy?
- What are safe defaults vs. production-ready settings, and why?

Write a 4–6 sentence summary in your own words and commit it to your repo.

---

## Requirements
After this activity, your PulseVote backend will:
- Limit register attempts per client to reduce scripted account creation.
- Limit login attempts per client (per-IP+email) to throttle credential stuffing/brute force.
- Return consistent `429 Too Many Requests` JSON messages.
- Expose standard `RateLimit-*` headers for observability.
- Be proxy‑aware (`trust proxy`) so limits key on real client IPs.

---

## Code Integration

### 1. Install
```bash
npm i express-rate-limit
```

### 2. Create a reusable limiter
Create `middleware/rateLimiter.js`:

```js
onst rateLimit = require('express-rate-limit');

const keyByIp = (req) =>
  req.ip ||
  req.headers['x-forwarded-for']?.split(',')[0]?.trim() ||
  req.connection?.remoteAddress ||
  'unknown';

const registerLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  standardHeaders: true,
  legacyHeaders: false,
  keyGenerator: keyByIp,
  handler: (req, res) => {
    return res.status(429).json({
      message: 'Too many registration attempts. Please try again later.'
    });
  },
});

const loginLimiter = rateLimit({
    windowMs: 10 * 60 * 1000,
    max: 5,
    standardHeaders: true,
    legacyHeaders: false,
    skipSuccessfulRequests: true,
    keyGenerator: (req) => `${keyByIp(req)}:${req.body?.email || ''}`,
    handler: (req, res) => {
      return res.status(429).json({
        message: 'Too many login attempts. Please try again later.'
      });
    },
});

module.exports = { registerLimiter, loginLimiter };

```

### 3. Apply to routes
Update `routes/authRoutes.js`:

```js
const { registerLimiter, loginLimiter} = require("../middleware/rateLimiter")

router.post("/register-user", registerLimiter, [emailValidator, passwordValidator], registerUser);
router.post("/register-manager", protect, requireRole("admin"), registerLimiter, [emailValidator, passwordValidator], registerManager);
router.post("/register-admin", registerLimiter, [emailValidator, passwordValidator], registerAdmin);

router.post("/login", loginLimiter, [emailValidator, body("password").notEmpty().trim().escape()], login);

```

### 4. Trust proxy (if applicable) and ensure JSON body parsing
Update `app.js` to include:

```js
const express = require('express');
const app = express();

app.set('trust proxy', 1);

app.use(express.json());
```
## Postman Testing

1. I have shared [Postman Test Script](/08%20-%20Rate%20Limiting/PulseVite%20Rate%20Limit%20Test.postman_collection). This script will test the API using some predefined values. It will ping your login 5 times and register 5 times which should result in an error 429.
3. Open it in Postman and run the test scripts separately. If you have configured your API correctly, it should pass all tests. If not, either adjust your API or tests.