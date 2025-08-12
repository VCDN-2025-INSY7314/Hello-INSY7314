# PulseVote – Organisations, Roles & Polls

## Research
Before you start coding, research Role-Based Access Control (RBAC) and why it’s important in real-world web apps.  

Spend 10–15 minutes answering:

- What is RBAC?
- Why is RBAC important in web applications?
- How could  different user roles (admin, manager, user) affect access in PulseVote?
- What could go wrong if RBAC is not implemented correctly?
- Find a real-world example where a lack of RBAC caused a security breach.

Write a short summary (4–6 sentences) in your own words and commit it to your repo.

Suggested links:
- https://auth0.com/docs/manage-users/access-control/rbac

---

## Requirements
After this activity, your PulseVote backend will allow:
* JWT login/registration with three roles: Admin, Manager, User.
* Organisations: create, join via code, belong to multiple.
* Polls: create (manager/admin), vote (one per user), close polls, view polls.
* Security: role checks, membership checks, no duplicate joins/votes.

## Code Integration
This guide explains how to add:
- Roles including admin, manager, user
- Organisations that users can join, and managers can manage.
- Polls within organisations which users can vote in.
- Secure voting and permissions.

- Remember to test in Postman as you go!

- I have included a postman test script right at the end.

### 1. Adding Role Support

Here, you need to update the API to include support for different roles.

1. Add a new schema for a role and update the user schema to include a role - do this in `models/User.js`
    ```js
    const roleSchema = new mongoose.Schema({
    organisationId: { type: mongoose.Schema.Types.ObjectId, ref: "Organisation" },
    role: { type: String, enum: ["admin", "manager", "user"], required: true }
    }, { _id: false });

    const userSchema = new mongoose.Schema({
    email: { type: String, unique: true, required: true },
    password: { type: String, required: true },
    roles: [roleSchema]
    });
    ```

2. Update the `controllers/authController.js` to cater for roles and return a JWT with user and role. Also, we need to cater for different level of registration.

    ```js
    const jwt = require("jsonwebtoken");
    const User = require("../models/User");
    const { validationResult } = require("express-validator");

    const generateToken = (user) =>
    jwt.sign(
        { id: user._id, email: user.email, roles: user.roles },
        process.env.JWT_SECRET,
        { expiresIn: "1h" }
    );

    exports.registerUser = async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty())
        return res.status(400).json({ message: "Invalid input", errors: errors.array() });

    const { email, password } = req.body;
    try {
        const existing = await User.findOne({ email });
        if (existing) return res.status(400).json({ message: "Email already exists" });

        const user = await User.create({
        email,
        password,
        roles: [{ organisationId: null, role: "user" }]
        });

        const token = generateToken(user);
        res.status(201).json({ message: "User registered", token });
    } catch (err) {
        res.status(500).json({ error: "Server error" + err});
    }
    };

    exports.registerManager = async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty())
        return res.status(400).json({ message: "Invalid input", errors: errors.array() });

    try {
        const adminUser = await User.findById(req.user.id);
        if (!adminUser || !adminUser.roles.some(r => r.role === "admin")) {
        return res.status(403).json({ message: "Only admins can create managers" });
        }

        const { email, password } = req.body;

        const existing = await User.findOne({ email });
        if (existing) return res.status(400).json({ message: "Email already exists" });

        const managerUser = await User.create({
        email,
        password,
        roles: [{ organisationId: null, role: "manager" }]
        });

        const token = generateToken(managerUser);
        res.status(201).json({ message: "Manager registered", token });
    } catch (err) {
        res.status(500).json({ error: "Server error: " + err });
    }
    };

    exports.registerAdmin = async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty())
        return res.status(400).json({ message: "Invalid input", errors: errors.array() });

    try {
        const { email, password } = req.body;

        const adminExists = await User.exists({ "roles.role": "admin" });

        if (adminExists) {
        const requestingUser = await User.findById(req.user.id);
        const isAdmin = requestingUser?.roles?.some(r => r.role === "admin");
        if (!isAdmin) {
            return res.status(403).json({ message: "Only admins can create admins" });
        }
        }

        const existing = await User.findOne({ email });
        if (existing) return res.status(400).json({ message: "Email already exists" });

        const adminUser = await User.create({
        email,
        password,
        roles: [{ organisationId: null, role: "admin" }]
        });

        const token = generateToken(adminUser);
        return res.status(201).json({ message: "Admin registered", token });
    } catch (err) {
        return res.status(500).json({ error: "Server error: " + err });
    }
    };

    exports.login = async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty())
        return res.status(400).json({ message: "Invalid input", errors: errors.array() });

    const { email, password } = req.body;
    try {
        const user = await User.findOne({ email });
        if (!user || !(await user.comparePassword(password))) {
        return res.status(400).json({ message: "Invalid credentials" });
        }

        const token = generateToken(user);
        res.json({ token });
    } catch (err) {
        res.status(500).json({ error: "Server error" });
    }
    };
    ```

3. Like we did for the auth, we need add middleware that checks roles and ensures that an admin is an admin, a manager is a manager, and a user is a user.
    ```js
    const User = require("../models/User");

    const requireRole = (role) => {
    return async (req, res, next) => {
        try {
        const user = await User.findById(req.user.id);
        if (!user) return res.status(401).json({ message: "User not found" });

        if (role === "admin") {
            const isAdmin = user.roles.some(r => r.role === "admin");
            if (!isAdmin) return res.status(403).json({ message: "Forbidden" });
            return next();
        }

        const orgId = req.params.organisationId || req.body.organisationId;

        const hasRole = user.roles.some(r =>
            r.role === role && (!orgId || r.organisationId?.toString() === orgId)
        );

        if (!hasRole && !user.roles.some(r => r.role === "admin")) {
            return res.status(403).json({ message: "Forbidden" });
        }

        next();
        } catch (err) {
        res.status(500).json({ error: "Server error"});
        }
    };
    };

    module.exports = { requireRole };
    ```

4. Update the `authRoutes.js` to cater for the new routes and to use the new roleMiddleware you added.

    ```js
    const { registerUser, registerManager, registerAdmin, login } = require("../controllers/authController");

    const { requireRole } = require("../middleware/roleMiddleware");

    // remove /register endpoint and include these ones.
    router.post("/register-user", [emailValidator, passwordValidator], registerUser);
    router.post("/register-manager", protect, requireRole("admin"), [emailValidator, passwordValidator], registerManager);
    router.post("/register-admin", [emailValidator, passwordValidator], registerAdmin);
    router.post("/login", [emailValidator, body("password").notEmpty().trim().escape()], login);
    ```
    Note: We are not protecting `/register-admin` here, but have the check in the controller. This is to cater for no admins in the first place.

### 2. Adding Organisation Support

Now that we can cater for different roles, let's cater for organisations which managers can manage, and users can join.

1. Add a model so we can store organisation info. Add this to `models/Organisation.js`
    ```js
    const mongoose = require("mongoose");
    const crypto = require("crypto");

    const organisationSchema = new mongoose.Schema({
    name: { type: String, required: true, unique: true },
    joinCode: { type: String, unique: true },
    createdBy: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    members: [{ type: mongoose.Schema.Types.ObjectId, ref: "User" }]
    });

    organisationSchema.methods.generateJoinCode = function () {
    this.joinCode = crypto.randomBytes(4).toString("hex");
    return this.joinCode;
    };

    module.exports = mongoose.model("Organisation", organisationSchema);
    ```

2. Next, add a controller that allows users to interact with an organisation. This is functionality, but we will implement the RBAC in the next step.

    ```js
    const { validationResult } = require("express-validator");

    const Organisation = require("../models/Organisation");
    const User = require("../models/User");

    exports.createOrganisation = async (req, res) => {
    try {
        const { name } = req.body;

        const org = new Organisation({
        name,
        createdBy: req.user.id,
        members: [req.user.id]
        });

        const user = await User.findById(req.user.id);
        user.roles.push({ organisationId: org._id, role: "manager" });
        await user.save();

        org.generateJoinCode();
        await org.save();

        res.status(201).json({ message: "Organisation created", organisation: org });
    } catch (err) {
        res.status(500).json({ error: "Server error" });
    }
    };

    exports.generateJoinCode = async (req, res) => {
    try {
        const { organisationId } = req.params;
        const org = await Organisation.findById(organisationId);
        if (!org) return res.status(404).json({ message: "Organisation not found" });

        org.generateJoinCode();
        await org.save();

        res.json({ message: "Join code regenerated", joinCode: org.joinCode });
    } catch (err) {
        res.status(500).json({ error: "Server error" });
    }
    };

    exports.joinOrganisation = async (req, res) => {
    try {
        const { joinCode } = req.body;

        const org = await Organisation.findOne({ joinCode });
        if (!org) return res.status(404).json({ message: "Invalid join code" });

        if (!org.members.includes(req.user.id)) {
        org.members.push(req.user.id);
        await org.save();
        }

        const user = await User.findById(req.user.id);
        const alreadyInRole = user.roles.some(r => 
        r.organisationId?.toString() === org._id.toString()
        );

        if (!alreadyInRole) {
        user.roles.push({ organisationId: org._id, role: "user" });
        await user.save();
        }

        res.json({ message: "Joined organisation", organisation: org });
    } catch (err) {
        res.status(500).json({ error: "Server error" });
    }
    };
    ```
3. Now, we add in the different routes with the following restrictions in place:
    * Only managers can create organisations
    * Only managers can generate join codes
    * Registered users can join an organisation

    ```js
    const express = require("express");
    const { protect } = require("../middleware/authMiddleware");
    const { requireRole } = require("../middleware/roleMiddleware");
    const {
    createOrganisation,
    generateJoinCode,
    joinOrganisation
    } = require("../controllers/organisationController");

    const router = express.Router();

    router.post("/create-organisation", protect, requireRole("manager"), createOrganisation);
    router.post("/generate-join-code/:organisationId/", protect, requireRole("manager"), generateJoinCode);
    router.post("/join-organisation", protect, joinOrganisation);

    module.exports = router;
    ```

4. Add the route to `app.js`
    ```js
    const organisationRoutes = require("./routes/organisationRoutes");

    app.use("/api/organisations", organisationRoutes);
    ```

### 3. Adding Poll Functionality

Now that we cater for different roles and organisations, we can add functionality to support polls which are tied to organisations.

1. Add in `models/Poll.js` which caters for a poll and its question. We assume a poll has just one question.
    ```js
    const mongoose = require("mongoose");

    const voteSchema = new mongoose.Schema({
    userId: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    optionIndex: { type: Number, required: true }
    }, { _id: false });

    const pollSchema = new mongoose.Schema({
    organisationId: { type: mongoose.Schema.Types.ObjectId, ref: "Organisation", required: true },
    question: { type: String, required: true },
    options: [{ type: String, required: true }],
    createdBy: { type: mongoose.Schema.Types.ObjectId, ref: "User", required: true },
    status: { type: String, enum: ["open", "closed"], default: "open" },
    votes: [voteSchema]
    });

    pollSchema.index({ organisationId: 1 });

    module.exports = mongoose.model("Poll", pollSchema);
    ```

2. We need a controller to allow for creation, voting, results, viewing polls within an organisation, opening and closing.

    ```js
    const Poll = require("../models/Poll");
    const Organisation = require("../models/Organisation");
    const User = require("../models/User")

    exports.createPoll = async (req, res) => {
    try {
        const { organisationId, question, options } = req.body;

        if (!options || options.length < 2) {
        return res.status(400).json({ message: "A poll must have at least two options" });
        }

        const org = await Organisation.findById(organisationId);
        if (!org) return res.status(404).json({ message: "Organisation not found" });

        const poll = await Poll.create({
        organisationId,
        question,
        options,
        createdBy: req.user.id
        });

        res.status(201).json({ message: "Poll created", poll });
    } catch (err) {
        res.status(500).json({ error: "Server error" });
    }
    };

    exports.votePoll = async (req, res) => {
    try {
        const { pollId } = req.params;
        const { optionIndex } = req.body;

        const poll = await Poll.findById(pollId);
        if (!poll) return res.status(404).json({ message: "Poll not found" });
        if (poll.status !== "open") return res.status(400).json({ message: "Poll is closed" });

        const alreadyVoted = poll.votes.some(v => v.userId.toString() === req.user.id);
        if (alreadyVoted) {
        return res.status(409).json({ message: "You have already voted" });
        }

        poll.votes.push({ userId: req.user.id, optionIndex });
        await poll.save();

        res.json({ message: "Vote recorded", poll });
    } catch (err) {
        res.status(500).json({ error: "Server error" });
    }
    };

    exports.getPollResults = async (req, res) => {
    try {
        const { pollId } = req.params;

        const poll = await Poll.findById(pollId).lean();
        if (!poll) return res.status(404).json({ message: "Poll not found" });

        const user = await User.findById(req.user.id).lean();
        const isAdmin = user?.roles?.some(r => r.role === "admin");
        const isMember = user?.roles?.some(
        r => r.organisationId?.toString() === poll.organisationId.toString()
        );
        if (!isAdmin && !isMember) {
        return res.status(403).json({ message: "Not a member of this organisation" });
        }

        const optionCount = poll.options.length;
        const counts = Array(optionCount).fill(0);

        for (const v of (poll.votes || [])) {
        if (typeof v.optionIndex === "number" && v.optionIndex >= 0 && v.optionIndex < optionCount) {
            counts[v.optionIndex]++;
        }
        }

        const totalVotes = counts.reduce((a, b) => a + b, 0);
        const percentages = counts.map(c => (totalVotes ? +( (c / totalVotes) * 100 ).toFixed(2) : 0));

        let userVoteIndex = null;
        if (poll.votes?.length) {
        const mine = poll.votes.find(v => v.userId?.toString() === req.user.id);
        if (mine) userVoteIndex = mine.optionIndex;
        }

        return res.json({
        poll: {
            _id: poll._id,
            organisationId: poll.organisationId,
            question: poll.question,
            options: poll.options,
            status: poll.status
        },
        results: {
            counts,
            percentages,
            totalVotes,
            userVoteIndex
        }
        });
    } catch (err) {
        return res.status(500).json({ error: "Server error"});
    }
    };

    exports.getOrgPolls = async (req, res) => {
    try {
        const { organisationId } = req.params;
        const polls = await Poll.find({ organisationId });
        res.json(polls);
    } catch (err) {
        res.status(500).json({ error: "Server error" });
    }
    };

    exports.closePoll = async (req, res) => {
    try {
        const {pollId } = req.params;
        
        const poll = await Poll.findById(pollId);
        if (!poll) return res.status(404).json({ message: "Poll not found" });

        poll.status = "closed";
        await poll.save();

        res.json({ message: "Poll closed", poll });
    } catch (err) {
        res.status(500).json({ error: "Server error" });
    }
    };

    exports.openPoll = async (req, res) => {
    try {
        const {pollId } = req.params;
        
        const poll = await Poll.findById(pollId);
        if (!poll) return res.status(404).json({ message: "Poll not found" });

        poll.status = "open";
        await poll.save();

        res.json({ message: "Poll opened", poll });
    } catch (err) {
        res.status(500).json({ error: "Server error" });
    }
    };
    ```
3. Now, let's add it route with the following rules:
    * Managers can create  polls
    * Only Users can complete polls.
    * Registered users can view polls or results
    * Managers can open or close polls

    ```js
    const express = require("express");
    const { protect } = require("../middleware/authMiddleware");
    const { requireRole } = require("../middleware/roleMiddleware");
    const {
        createPoll,
        votePoll,
        getPollResults,
        getOrgPolls,
        closePoll,
        openPoll
    } = require("../controllers/pollController");

    const router = express.Router();

    router.post("/create-poll", protect, requireRole("manager"), createPoll);
    router.post("/vote/:pollId", protect, requireRole("user"), votePoll);
    router.get("/get-poll-results/:pollId", protect, getPollResults);
    router.get("/get-polls/:organisationId", protect, getOrgPolls);
    router.post("/close/:pollId", protect, requireRole("manager"), closePoll);
    router.post("/open/:pollId", protect, requireRole("manager"), openPoll);

    module.exports = router;
    ```
4. Add the route to `app.js`
    ```js
    const pollRoutes = require("./routes/pollRoutes");

    app.use("/api/polls", pollRoutes);
    ```

### 4. Postman Testing

1. I have shared [Postman Test Script](/07%20-%20Adding%20RBAC/PulseVote%20RBAC%20Test.postman_collection.json). This script will test the API using some predefined values. Where it gets tokens, organisation and poll IDs, it will save them for use in subsequent requests.
2. Download and edit it in VS Code to add the variables at the bottom. Add passwords only. The rest of the blank values are added when you run it. 
4. Clear your MongoDB collections or update the values in the json config file to ensure no duplicates.
3. Open it in Postman and run the tests. If you have configured your API correctly, it should pass all tests. If not, either adjust your API or tests.