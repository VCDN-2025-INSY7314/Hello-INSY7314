# PulseVote Backend — Logging to Grafana Cloud with Loki

This guide will take you through the process of:

1. Creating a Grafana Cloud account and Loki data source.
2. Setting up environment variables for Loki in your env folder for local and Render for prod.
3. Configuring a `logger.js` file using `pino` and `pino-loki`.
4. Updating your controllers to log meaningful events.
5. Viewing logs in Grafana Explore.

## Grafana Loki Setup

### Step 1. Create a Grafana Cloud Account and setting up a token.

1. Go to [Grafana.com](https://grafana.com) and sign up (free tier works).
2. Once logged in, go to **Home** and under the **Getting Started Dashboard**, select **Logs**.
3. Under infrastructure, choose **Platform as a Service**, then select **Send logs via HTTP**.
4. Under **Use an API Token**, select **Create a new token**.
5. Give it a name (e.g. `pulsevote-api`).
6. Leave the following default scopes `metrics:write, logs:write, traces:write, profiles:write, stacks:read`
4. Create and copy the token — you’ll only see it once.

### Step 2. Get URL and User ID

From the Grafana sample code, choose Node and you’ll see three important values:

- `LOKI_URL` → e.g. your Loki stack url (e.g. `https://logs-prod-033.grafana.net`)
- `LOKI_USER` → your stack ID a numeric sequence (e.g. `1334333`)
- `LOKI_API_KEY` → the API key you created above

Copy and store these too.

## Backend Changes
### Step 1. npm install

Run this in your project directory:
```
npm install pino pino-loki
```
### Step 2. Add Environment Variables

Store the environment variables in `.env` for local testing and in Render environment variables to use in prod. We don't need them in CircleCI unless we use them for testing in our pipeline.

Update `.env` file in your project:

```env
LOKI_URL=your URL from Grafana
LOKI_USER=your stack ID from Grafana
LOKI_API_KEY= your API key from Grafana
```
> Reminder: Never commit `.env` to git.

### Step 3. Create Logger Utility

Add a new file: `utils/logger.js`

```js
const pino = require("pino");
require("dotenv").config();

const logger = pino({
  level: "info",
}, pino.transport({
  target: "pino-loki",
  options: {
    batching: true,
    interval: 5,
    host: process.env.LOKI_URL,
    endpoint: "/loki/api/v1/push",
    basicAuth: {
      username: process.env.LOKI_USER,
      password: process.env.LOKI_API_KEY
    },
    labels: {
      service_name: "pulsevote-api",
      env: process.env.NODE_ENV
    },
  },
}));

module.exports = logger;
```

### Step 4. Update Controller to Log Events

We will only do this, in this tutorial for login. You are required to, once you get it working, to add it for other areas of your app.

Example: `controllers/authController.js`

```js
exports.login = async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    logger.warn({
      msg: "Invalid login input",
      errors: errors.array(),
      ip: req.ip,
      userAgent: req.headers["user-agent"],
    });
    return res.status(400).json({ message: "Invalid input", errors: errors.array() });
  }

  const { email, password } = req.body;
  try {
    const user = await User.findOne({ email });

    if (!user || !(await user.comparePassword(password))) {
      logger.warn({
        msg: "Unauthorized login attempt",
        email,
        ip: req.ip,
        userAgent: req.headers["user-agent"],
      });
      return res.status(400).json({ message: "Invalid credentials" });
    }

    const token = generateToken(user);

    logger.info({
      msg: "User logged in",
      userId: user._id.toString(),
      email,
      ip: req.ip,
      userAgent: req.headers["user-agent"],
    });

    res.json({ token });
  } catch (err) {
    logger.error({
      msg: "Login error",
      error: err.message,
      stack: err.stack,
      ip: req.ip,
      userAgent: req.headers["user-agent"],
    });
    res.status(500).json({ error: "Server error" });
  }
};
```

### Step 5. Run your app and test login in Postman.
Run your API locally and test the login in Postman with a successful and unsuccessful login. 

## Explore in Grafana 
### Step 1. View Logs
1. Go to **Grafana → Explore → Logs**.
2. Query using the label you set:

```logql
{service_name="pulsevote-api"}
```

3. Expand a log line → click **Parse as JSON** to see fields like `msg`, `email`, `ip`, `userAgent`.

### Step 2. Example Queries

- Show only unauthorized login attempts:

```logql
{service_name="pulsevote-api"} | json | msg="Unauthorized login attempt"
```

- Count logins per IP:

```logql
{service_name="pulsevote-api"} | json | count by (ip)
```

## Unit Tests
1. Write unit tests for `logger.js` to confirm it loads env vars correctly. Also, you need this so Sonar step runs in your pipeline.
2. Extend tests in `authController.test.js` to verify that the correct log messages and metadata are sent for:
   - Invalid input
   - Unauthorized login
   - Successful login
   - Server error

## Test in production
Push your changes and test them in Render. Logs should appear in Grafana Cloud within a few seconds of running API requests.