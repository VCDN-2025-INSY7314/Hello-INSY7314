# Adding Admin Functionality on the frontend

To allow for secure adding admin, you will need to adjust how admins are created on the backend.
Code changes and the postman collection are included directly here for brevity.

## Backend changes

#### Step 1: Updating authController
Update `authController.js` to include an initialise an admin separately, amend existing register admins and sanitise input.

```js
const jwt = require("jsonwebtoken");
const User = require("../models/User");
const { validationResult } = require("express-validator");
const logger = require("../utils/logger");

const generateToken = (user) =>
  jwt.sign(
    { id: user._id, email: user.email, roles: user.roles },
    process.env.JWT_SECRET,
    { expiresIn: "1h" }
  );

  exports.initAdmin = async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty())
      return res.status(400).json({ message: "Invalid input", errors: errors.array() });

    const email = req.body.email.toString();
    const password = req.body.password.toString();

    try {
      const adminExists = await User.exists({ "roles.role": "admin" });
      if (adminExists) {
        return res.status(403).json({ message: "Admin already exists" });
      }

      const existing = await User.findOne({ email: { $eq: email } });
      if (existing) return res.status(400).json({ message: "Email already exists" });
   
      const newAdmin = await User.create({
        email,
        password,
        roles: [{ organisationId: null, role: "admin" }],
      });
  
      const token = generateToken(newAdmin);
      return res.status(201).json({
        message: "Initial admin created successfully",
        token,
      });
    } catch (err) {
      return res.status(500).json({ error: "Server error: " + err.message });
    }
  };
  
exports.registerUser = async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty())
    return res.status(400).json({ message: "Invalid input", errors: errors.array() });

  const email = req.body.email.toString();
  const password = req.body.password.toString();

  try {
    const existing = await User.findOne({ email: { $eq: email } });
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
    const requestingUser = await User.findById(req.user.id);

    const isAuthorized =
      requestingUser &&
      requestingUser.roles.some(r =>
        r.role === "admin" || r.role === "manager"
      );

    if (!isAuthorized) {
      return res.status(403).json({ message: "Only admins or managers can create managers" });
    }

    const email = req.body.email.toString();
    const password = req.body.password.toString();

    const existing = await User.findOne({ email: { $eq: email } });
    if (existing)
      return res.status(400).json({ message: "Email already exists" });

    const managerUser = await User.create({
      email,
      password,
      roles: [{ organisationId: null, role: "manager" }],
    });

    const token = generateToken(managerUser);
    return res.status(201).json({ message: "Manager registered", token });
  } catch (err) {
    return res.status(500).json({ error: "Server error: " + err.message });
  }
};

exports.registerAdmin = async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty())
    return res.status(400).json({ message: "Invalid input", errors: errors.array() });

  try {
    const requestingUser = await User.findById(req.user.id);

    if (!requestingUser || !requestingUser.roles.some(r => r.role === "admin")) {
      return res.status(403).json({ message: "Only admins can create admins" });
    }

    const email = req.body.email.toString();
    const password = req.body.password.toString();

    const existing = await User.findOne({ email: { $eq: email } });
    if (existing)
      return res.status(400).json({ message: "Email already exists" });

    const adminUser = await User.create({
      email,
      password,
      roles: [{ organisationId: null, role: "admin" }],
    });

    const token = generateToken(adminUser);
    return res.status(201).json({ message: "Admin registered", token });
  } catch (err) {
    return res.status(500).json({ error: "Server error: " + err.message });
  }
};

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

  const email = req.body.email.toString();
  const password = req.body.password.toString();

  try {
    const user = await User.findOne({ email: { $eq: email } });

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

#### Step 2: Updating authRoutes
We need to add the `/init-admin` endpoint and protect the `/register-admin` endpoint.

```js
const express = require("express");
const { body } = require("express-validator");
const { registerUser, registerManager, registerAdmin, login, initAdmin } = require("../controllers/authController");
const { protect } = require("../middleware/authMiddleware");
const { requireRole } = require("../middleware/roleMiddleware");
const { registerLimiter, loginLimiter} = require("../middleware/rateLimiter")

const router = express.Router();

const emailValidator = body("email")
  .isEmail().withMessage("Email must be valid")
  .normalizeEmail();

const passwordValidator = body("password")
  .isLength({ min: 8 }).withMessage("Password must be at least 8 characters")
  .matches(/[A-Za-z]/).withMessage("Password must include a letter")
  .matches(/\d/).withMessage("Password must include a number")
  .trim().escape();

  router.post("/init-admin", initAdmin);

  router.post("/register-user", registerLimiter, [emailValidator, passwordValidator], registerUser);
  router.post("/register-manager", protect, requireRole("admin"), registerLimiter, [emailValidator, passwordValidator], registerManager);
  router.post("/register-admin", protect, requireRole("admin"), registerLimiter, [emailValidator, passwordValidator], registerAdmin);
  
  router.post("/login", loginLimiter, [emailValidator, body("password").notEmpty().trim().escape()], login);

module.exports = router;
```

#### Step 3: Unit tests
Don't forget to update your unit tests.

#### Step 4: Updating collection.json
Since have made changes to the endpoints, we need to amend our tests in the pipeline.

```json
{
	"info": {
	  "_postman_id": "a1089c93-9521-4b67-a505-d13ef9a7db99",
	  "name": "PulseVote RBAC Test",
	  "description": "Full RBAC test flow for PulseVote.",
	  "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
	},
	"item": [
		{
			"name": "0a. Register First Admin",
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{ADMIN_EMAIL}}\",\n  \"password\": \"{{ADMIN_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/init-admin",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"init-admin"
					]
				}
			},
			"response": []
		},
		{
			"name": "0b. Login First Admin",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('ADMIN_TOKEN', res.token);",
							"// Optional: pm.collectionVariables.set('ADMIN_ID', res.userId || res.id);"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{ADMIN_EMAIL}}\",\n  \"password\": \"{{ADMIN_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/login",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"login"
					]
				}
			},
			"response": []
		},
		{
			"name": "1a. Register Second Admin",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{ADMIN_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{SECOND_ADMIN_EMAIL}}\",\n  \"password\": \"{{SECOND_ADMIN_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/register-admin",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"register-admin"
					]
				}
			},
			"response": []
		},
		{
			"name": "1b. Login Second Admin",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('SECOND_ADMIN_TOKEN', res.token);"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{SECOND_ADMIN_EMAIL}}\",\n  \"password\": \"{{SECOND_ADMIN_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/login",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"login"
					]
				}
			},
			"response": []
		},
		{
			"name": "2. Register Manager",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{ADMIN_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{MANAGER_EMAIL}}\",\n  \"password\": \"{{MANAGER_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/register-manager",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"register-manager"
					]
				}
			},
			"response": []
		},
		{
			"name": "3. Login Manager",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('MANAGER_TOKEN', res.token);",
							"// Optional: pm.collectionVariables.set('MANAGER_ID', res.userId || res.id);"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{MANAGER_EMAIL}}\",\n  \"password\": \"{{MANAGER_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/login",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"login"
					]
				}
			},
			"response": []
		},
		{
			"name": "4. Create Organisation",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"if (res.organisation && res.organisation._id) {",
							"    pm.collectionVariables.set('ORG_ID', res.organisation._id);",
							"}",
							"if (res.organisation && res.organisation.joinCode) {",
							"    pm.collectionVariables.set('JOIN_CODE', res.organisation.joinCode);",
							"}",
							"console.log('Organisation ID:', pm.collectionVariables.get('ORG_ID'));",
							"console.log('Join Code:', pm.collectionVariables.get('JOIN_CODE'));"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"name\": \"{{ORG_NAME}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/organisations/create-organisation",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"organisations",
						"create-organisation"
					]
				}
			},
			"response": []
		},
		{
			"name": "5. Register User",
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{USER_EMAIL}}\",\n  \"password\": \"{{USER_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/register-user",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"register-user"
					]
				}
			},
			"response": []
		},
		{
			"name": "6. Login User",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('USER_TOKEN', res.token);",
							"// Optional: pm.collectionVariables.set('USER_ID', res.userId || res.id);"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{USER_EMAIL}}\",\n  \"password\": \"{{USER_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/login",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"login"
					]
				}
			},
			"response": []
		},
		{
			"name": "7. Join Organisation",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{USER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"joinCode\": \"{{JOIN_CODE}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/organisations/join-organisation",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"organisations",
						"join-organisation"
					]
				}
			},
			"response": []
		},
		{
			"name": "8. Generate Join Code",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('JOIN_CODE', res.joinCode || res.code || (res.data && res.data.joinCode));"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": ""
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/organisations/generate-join-code/{{ORG_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"organisations",
						"generate-join-code",
						"{{ORG_ID}}"
					]
				}
			},
			"response": []
		},
		{
			"name": "9. Create Poll",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('POLL_ID', res._id || res.id);"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"organisationId\": \"{{ORG_ID}}\",\n  \"question\": \"{{POLL_QUESTION}}\",\n  \"options\": {{POLL_OPTIONS}}\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/create-poll",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"create-poll"
					]
				}
			},
			"response": []
		},
		{
			"name": "10. Get Organisation Polls",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"if (Array.isArray(res) && res.length) { pm.collectionVariables.set('POLL_ID', res[0]._id || res[0].id); }",
							"if (res && res.polls && res.polls.length) { pm.collectionVariables.set('POLL_ID', res.polls[0]._id || res.polls[0].id); }"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "GET",
				"header": [],
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/get-polls/{{ORG_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"get-polls",
						"{{ORG_ID}}"
					]
				}
			},
			"response": []
		},
		{
			"name": "11. Vote in Poll",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{USER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"optionIndex\": {{OPTION_INDEX}}\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/vote/{{POLL_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"vote",
						"{{POLL_ID}}"
					]
				}
			},
			"response": []
		},
		{
			"name": "12. Get Poll Results",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "GET",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/get-poll-results/{{POLL_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"get-poll-results",
						"{{POLL_ID}}"
					]
				}
			},
			"response": []
		},
		{
			"name": "13. Close Poll",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"organisationId\": \"{{ORG_ID}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/close/{{POLL_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"close",
						"{{POLL_ID}}"
					]
				}
			},
			"response": []
		},
		{
			"name": "14. Re-open Poll",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"organisationId\": \"{{ORG_ID}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/open/{{POLL_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"open",
						"{{POLL_ID}}"
					]
				}
			},
			"response": []
		}
	],
	"event": [
		{
			"listen": "prerequest",
			"script": {
				"type": "text/javascript",
				"packages": {},
				"exec": [
					""
				]
			}
		},
		{
			"listen": "test",
			"script": {
				"type": "text/javascript",
				"packages": {},
				"exec": [
					""
				]
			}
		}
	],
	"variable": [
	  { "key": "PROTOCOL", "value": "http" },
	  { "key": "HOST", "value": "localhost" },
	  { "key": "ADMIN_EMAIL", "value": "admin@pulsevote.com" },
	  { "key": "ADMIN_PASSWORD", "value": "{{TEST_PASSWORD}}" },
	  { "key": "SECOND_ADMIN_EMAIL", "value": "admin2@pulsevote.com" },
	  { "key": "SECOND_ADMIN_PASSWORD", "value": "{{TEST_PASSWORD}}" },
	  { "key": "MANAGER_EMAIL", "value": "manager@pulsevote.com" },
	  { "key": "MANAGER_PASSWORD", "value": "{{TEST_PASSWORD}}" },
	  { "key": "USER_EMAIL", "value": "user@pulsevote.com" },
	  { "key": "USER_PASSWORD", "value": "{{TEST_PASSWORD}}" },
	  { "key": "ORG_NAME", "value": "INSY7314" },
	  { "key": "POLL_QUESTION", "value": "What's your favourite programming language?" },
	  { "key": "POLL_OPTIONS", "value": "[\"JavaScript\", \"Python\", \"C#\"]" },
	  { "key": "OPTION_INDEX", "value": "1" },
	  { "key": "ADMIN_TOKEN", "value": "" },
	  { "key": "SECOND_ADMIN_TOKEN", "value": "" },
	  { "key": "MANAGER_TOKEN", "value": "" },
	  { "key": "USER_TOKEN", "value": "" },
	  { "key": "ORG_ID", "value": "" },
	  { "key": "JOIN_CODE", "value": "" },
	  { "key": "POLL_ID", "value": "" }
	]
  }
  
```