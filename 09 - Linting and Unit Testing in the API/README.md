# PulseVote – Linting and Unit Testing

## Research
Before you code, spend 10–15 minutes exploring what linting and unit tests are and why it matters.

Answer briefly:
- What is linting?
- What are unit tests?
- Why are they critical for code?
- Can these be automated in any way?
- How do you perform linting in a nodejs api?
- How do you perform unit testing in a nodejs api?
- What problems can flaky tests bring to our lives?

Write a 4–6 sentence summary in your own words and commit it to your repo.

---

## Requirements
After this activity, your PulseVote backend will:
- Incorporate linting
- Incorporate unit tests

Note: you will run each script separately but when we get to pipelines, we will run these automatically.

---

## Code Integration Unit Testing

### 1. Install dependencies
```bash
npm i -D jest supertest cross-env @eslint/js globals
```

### 2. Add in basic unit testing

We will add a simple unit test to check that the API is up and running.

1. Add in a health enpoint in `app.js`:
    ```js
    app.get('/health', (req, res) => 
    res.status(200).json({
        ok: true,
        ts: Date.now()
    }));
    ```

1. Add a simple unit test to `test/health.test.js`
    ```js
    const request = require("supertest");
    const app = require("../app");

    describe("Health", () => {
    it("GET /health -> 200", async () => {
        const res = await request(app).get("/health");
        expect(res.statusCode).toBe(200);
        expect(res.body).toHaveProperty("ok", true);
    });
    });
    ```

1. Create `jest.config.js` for test configuration:
    ```js
    module.exports = {
        testEnvironment: "node",
        testMatch: ["**/test/**/*.test.js"],
        verbose: true
    };
    ```
### 3. Add in linting

1. Add in a lint config file called `eslint.config.cjs`:
    ```js
    const js = require("@eslint/js");
    const globals = require("globals");

    module.exports = [
    { ignores: ["node_modules/", "coverage/", "dist/", "build/", ".circleci/", "ssl/"] },

    js.configs.recommended,

    {
        files: ["**/*.js"],
        languageOptions: {
        ecmaVersion: 2023,
        sourceType: "commonjs",
        globals: { ...globals.node, ...globals.es2021 }
        },
        rules: {
        "no-unused-vars": ["warn", { argsIgnorePattern: "^_" }]
        }
    },

    {
        files: ["test/**/*.test.js", "**/__tests__/**/*.js"],
        languageOptions: {
        globals: { ...globals.jest }
        }
    }
    ];
    ```

### 4. Update scripts

1. Update `package.json` to reflect these scripts

    ```json
    "scripts": {
        "dev": "nodemon server.js",
        "start": "node server.js",
        "test": "jest --passWithNoTests",
        "lint": "eslint ."
    },
    ```
### 5. Run tests

1. Run your lint test
    ```
    npm run lint
    ```

1. Run your unit tests using this command
    ```
    npm test
    ```

You are encouraged to add your own unit tests as well!