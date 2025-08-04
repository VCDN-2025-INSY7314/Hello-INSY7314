# Securing your login

## Front-end suggestions

1. Basic checks for validity before passing data to the API:
    ```js
    if (!email || !password) {
        setError("Email and password are required.");
        return;
        }

        if (!isValidEmail(email)) {
        setError("Invalid email format.");
        return;
        }

        if (!isStrongPassword(password)) {
        setError("Password must be at least 8 characters long and include letters and numbers.");
        return;
        }
    ```

2. RegEx checks for email and passwords
    ```js
    const isValidEmail = (email) =>
        /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);

    const isStrongPassword = (password) =>
        /^(?=.*[A-Za-z])(?=.*\d)[A-Za-z\d!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]{8,}$/.test(password);

    ```

3. Add this to login and register.

### Back-end suggestions

1. install express-validator
    ```
    npm install express-validator
    ```
2. Add the following to `routes/authRouter.js` - do not just copy-paste
    ```js
    const { body } = require("express-validator");

    const router = express.Router(); // this is already there

    const emailValidator = body("email")
    .isEmail().withMessage("Email must be valid")
    .normalizeEmail();

    const passwordValidator = body("password")
    .isLength({ min: 8 }).withMessage("Password must be at least 8 characters")
    .matches(/[A-Za-z]/).withMessage("Password must include a letter")
    .matches(/\d/).withMessage("Password must include a number")
    .trim().escape();
    ```

    Update routes to use the validators:
    ```js
    router.post("/register", [emailValidator, passwordValidator], register);
    router.post("/login", [emailValidator, body("password").notEmpty().trim().escape()], login);
    ```

3. Add the following to `controllers/authController.js` - do not just copy-paste
    ```js
    const { validationResult } = require("express-validator");
    // other requires
    ```

    Then, in the register function:
    ```js
    exports.register = async (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty())
            return res.status(400).json({ message: "Invalid input", errors: errors.array() });

            // rest of your logic
    }
    ```
    Then, do the same for the login function.