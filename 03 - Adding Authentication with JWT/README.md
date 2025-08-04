# Adding Authentication with JWT

## Research
Before you start, research what JWT is, why it is necessary, and the impact it has on real-world web applications.
Spend 10–15 minutes researching the following questions:

- What is JWT?  
- Why is JWT necessary in web applications?
- How does JWT work?
- What could happen if a web application does not use SSL?
- Find one real-world example (news or blog) of a security incident involving JWT being omitted or misconfigured

Write a short summary (4–6 sentences) in your own words. Add this write up into your repo as well.

Suggested links:
- https://auth0.com/docs/secure/tokens/json-web-tokens
- https://www.jwt.io/

## Code Integraton
This guide explains how to apply auth to your PulseVote app and store profiles in MongoDB.
You will need to:
1. Configure MongoDB
2. Make API changes
3. Make frontend changes (see next activity)

Remember, you need to research and understand as you go...

## 1. MongoDB
1. Sign up for MongoDB and create a cluster for your project - https://www.mongodb.com/cloud/atlas/register.
2. Make sure to choose the free tier.
3. When you configure it, you will get a connection string that needs to be added to `.env` in API as follows:
    `
    MONGO_URI=<your string here>
    `
4. Update server.js to connect to the DB on launch:
    ```js
    const mongoose = require('mongoose');
    mongoose.connect(process.env.MONGO_URI)
    .then(() => {
        https.createServer(options, app).listen(PORT, () => {
        console.log('Server running at https://localhost:' + PORT);
        });
    })
    .catch((err) => {
        console.error('MongoDB connection error:', err);
    });
    ```
5. Test your app

## 2. API changes

### Getting started
Run npm install
```
npm install express mongoose dotenv cors bcrypt jsonwebtoken
```
Research what each offers you!

### Models
In your backend project folder, we will need to make the model class which allow for writing users to the database. We will also ned to encrypt passwords at the very minimum.

In the models folder, create a `models/User.js` model class:
```js
const mongoose = require("mongoose");
const bcrypt = require("bcrypt");

const userSchema = new mongoose.Schema({
  email: { type: String, unique: true, required: true },
  password: { type: String, required: true }
});

userSchema.pre("save", async function (next) {
  if (!this.isModified("password")) return next();
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

userSchema.methods.comparePassword = function (candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model("User", userSchema);
```
### Controller
We need to setup controllers to manage DB interaction for login and register attempts. Create a controller under `controllers/authController.js`

Before you do this, add a secret key in `.env` called JWT_SECRET.

If a user is successful, this controller generates and sends back a JWT.

```js
const jwt = require("jsonwebtoken");
const User = require("../models/User");

const generateToken = (userId) =>
  jwt.sign({ id: userId }, process.env.JWT_SECRET, { expiresIn: "1h" });

exports.register = async (req, res) => {
  const { email, password } = req.body;
  try {
    const existing = await User.findOne({ email });
    if (existing) return res.status(400).json({ message: "Email already exists" });

    const user = await User.create({ email, password });
    const token = generateToken(user._id);
    res.status(201).json({ token });
  } catch (err) {
    res.status(500).json({ error: "Server error" });
  }
};

exports.login = async (req, res) => {
  const { email, password } = req.body;
  try {
    const user = await User.findOne({ email });
    if (!user || !(await user.comparePassword(password))) {
      return res.status(400).json({ message: "Invalid credentials" });
    }

    const token = generateToken(user._id);
    res.json({ token });
  } catch (err) {
    res.status(500).json({ error: "Server error" });
  }
};

```
### Routes
To make the controller methods accessible, we need to setup the routes in `routes/authRoutes.js`
```js
const express = require("express");
const { register, login } = require("../controllers/authController");
const router = express.Router();

router.post("/register", register);
router.post("/login", login);

module.exports = router;
```
### Add routes to `app.js`
Add these to app.js

```js
const authRoutes = require("./routes/authRoutes");

app.use("/api/auth", authRoutes);
```

### Add CORS to `app.js`
Add these to app.js
```js
const cors = require('cors');
app.use(cors({
  origin: "https://localhost:5173",
  credentials: true
}));
```

### Add a protected endpoint
Add the protected endpoint to `app.js`

```js

const { protect } = require("./middleware/authMiddleware");

app.get("/api/protected", protect, (req, res) => {
  res.json({
    message: `Welcome, user ${req.user.id}! You have accessed protected data.`,
    timestamp: new Date()
  });
});
```
For this to work, we need to define the protection in `middleware/authMiddleware.js`

Here, we check if there is a token in the request, and if there is, we check validity.

```js
const jwt = require("jsonwebtoken");

const protect = (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith("Bearer "))
    return res.status(401).json({ message: "Unauthorized" });

  const token = authHeader.split(" ")[1];
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    res.status(403).json({ message: "Token invalid or expired" });
  }
};

module.exports = { protect };
```

### Test in Postman
1. Register a user at the relevant endpoint, check it shows up in Mongo.
2. Login and get the token
3. Make a request to the protected endpoint using the token, and without the token.

