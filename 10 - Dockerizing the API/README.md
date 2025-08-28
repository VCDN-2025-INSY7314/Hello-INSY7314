# PulseVote – Docker and docker-compose

## Research
Before you code, spend 10–15 minutes exploring what linting and unit tests are and why it matters.

Answer briefly:
- What is docker?
- What is docker-compose?
- Why are they an essential skill for developers?
- Can building and running of docker containers automated in any way?
- How do you dockerize a nodejs api?

Write a 4–6 sentence summary in your own words and commit it to your repo.

---

## Requirements
After this activity, your PulseVote backend will:
- Be dockerised using a `Dockerfile`
- Incorporate `docker-compose.yml`
- Incorporate `.dockerignore`

Note: you will run the container locally with a command here, but when we get to pipelines, we will run these automatically.

## Code changes


### 1. Update .gitignore
Update `.gitignore` to not push ssl. This is likely already there, but it would mean that there will be no certificates in your container. This is ok for now.

```.gitignore
node_modules
.env
ssl
.env.*.local
```
### 2. Update the server.js and app.js code.
Since we will be running our app in a container, some of the code we learnt locally need to be adjusted to cater for a containerized environment.

Notice that HTTPS is not applied in containerized environments
Research the changes and why they are needed when Dockerizing

1. Update `app.js`
    ```js
    const express = require('express');
    const cors = require('cors');
    const helmet = require('helmet');
    const dotenv = require('dotenv');
    const authRoutes = require("./routes/authRoutes");
    const organisationRoutes = require("./routes/organisationRoutes");
    const pollRoutes = require("./routes/pollRoutes");
    const { protect } = require("./middleware/authMiddleware");

    dotenv.config();
    const app = express();

    app.use(helmet());

    const CSP_CONNECT = (process.env.CSP_CONNECT || '').split(',').filter(Boolean);
    const defaultConnect = [
    "'self'",
    "http://localhost:5000", "https://localhost:5000",
    "http://localhost:5173", "https://localhost:5173",
    "ws://localhost:5173", "wss://localhost:5173"
    ];

    app.use(
    helmet.contentSecurityPolicy({
        useDefaults: true,
        directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "https://apis.google.com"],
        styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        imgSrc: ["'self'", "data:"],
        connectSrc: CSP_CONNECT.length ? CSP_CONNECT : defaultConnect,
        },
    })
    );

    const allowed = (process.env.CORS_ORIGINS || "http://localhost:5173,https://localhost:5173")
    .split(',')
    .map(s => s.trim());

    app.use(cors({
    origin: (origin, cb) => {
        if (!origin) return cb(null, true);
        if (allowed.includes(origin)) return cb(null, true);
        cb(new Error(`CORS blocked: ${origin}`));
    },
    credentials: true
    }));

    app.use(express.json());
    app.set('trust proxy', 1);

    app.use("/api/auth", authRoutes);
    app.use("/api/organisations", organisationRoutes);
    app.use("/api/polls", pollRoutes);

    app.get('/health', (req, res) => 
    res.status(200).json({
        ok: true,
        ts: Date.now()
    }));

    app.get('/', (req, res) => 
    res.send('PulseVote API running!'));

    app.get('/test', (req, res) => {
    res.json({
        message: 'This is a test endpoint from PulseVote API!',
        status: 'success',
        timestamp: new Date()
    });
    });

    app.get("/api/protected", protect, (req, res) => {
    res.json({
        message: `Welcome, user ${req.user.id}! You have accessed protected data.`,
        timestamp: new Date()
    });
    });

    module.exports = app;
    ```

1. Update `server.js`
    ```js
    const mongoose = require('mongoose');
    const app = require('./app');
    const https = require('https');
    const http = require('http');
    const fs = require('fs');
    require('dotenv').config();

    const PORT = process.env.PORT || 5000;
    const HOST = '0.0.0.0';
    const useHttps = String(process.env.USE_HTTPS || '').toLowerCase() === 'true';
    const mongo = process.env.MONGO_URI;

    if (!mongo) {
    console.error('Missing MONGO_URI');
    process.exit(1);
    }

    mongoose.connect(mongo)
    .then(() => {
        if (useHttps) {
        const keyPath = process.env.SSL_KEY_PATH || 'ssl/key.pem';
        const certPath = process.env.SSL_CERT_PATH || 'ssl/cert.pem';
        const haveFiles = fs.existsSync(keyPath) && fs.existsSync(certPath);

        if (!haveFiles) {
            console.warn('SSL files not found, falling back to HTTP');
        } else {
            const options = { key: fs.readFileSync(keyPath), cert: fs.readFileSync(certPath) };
            https.createServer(options, app).listen(PORT, HOST, () => {
            console.log(`HTTPS server running at https://localhost:${PORT}`);
            });
            return;
        }
        }

        http.createServer(app).listen(PORT, HOST, () => {
        console.log(`HTTP server running at http://localhost:${PORT}`);
        });
    })
    .catch((err) => {
        console.error('MongoDB connection error:', err);
        process.exit(1);
    });
    ```

3. Run your app to make sure it all still runs. Clear your test db in Mongo, then run the tests. They should all run successfully.

### 3. Dockerfile and docker-compose.yml
Research this in detail - do not blindly copy-paste please...

1. Add the `Dockerfile`
    ```Dockerfile
    FROM node:22-alpine3.20 AS deps
    WORKDIR /app
    COPY package*.json ./
    RUN npm ci --omit=dev

    FROM node:22-alpine3.20 AS runner
    WORKDIR /app
    ENV NODE_ENV=production
    ENV PORT=5000
    EXPOSE 5000

    HEALTHCHECK CMD node -e "require('http').get('http://localhost:'+ (process.env.PORT||5000) +'/health',r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))"

    RUN addgroup -S nodegrp && adduser -S nodeuser -G nodegrp
    USER nodeuser

    COPY --from=deps /app/node_modules ./node_modules
    COPY . .
    CMD ["npm","start"]
    ```
2. Add the `.dockerignore` file - what does this do?
    ```
    node_modules
    npm-debug.log
    Dockerfile*
    .dockerignore
    .git
    .gitignore
    coverage
    .vscode
    .circleci
    ssl
    .env
    .env.*.local
    test
    jest.config.js
    eslint.config.cjs
    ```
3. Add a `docker-compose.yml` - indenting matters!
    ```
    services:
        api:
            container_name: api
            build:
            context: .
            dockerfile: Dockerfile
            ports:
            - "5000:5000"
            env_file:
            - .env
            restart: unless-stopped

    ```

4. Run the nodejs app to make sure it runs. 
    ```
    npm run dev
    ```
    Access at https://localhost:5000

5. Run the container to make sure it runs. 
    ```
    docker-compose up --build
    ```
    Access at http://localhost:5000 (notice no s)

6. Update your postman collection to use http instead of https, clear your test db in Mongo, then run the tests.    
