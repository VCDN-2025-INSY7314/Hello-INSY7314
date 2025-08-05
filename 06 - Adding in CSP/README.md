# Adding in CSP with Helmet

Read up on helmet..

1. What does it offer by default? How do they benefit your tech stack?
2. Add in a CSP to `app.js` in your backend app
    ```js
    app.use(
    helmet.contentSecurityPolicy({
        directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "https://apis.google.com"],
        styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        imgSrc: ["'self'", "data:"],
        connectSrc: ["'self'", "http://localhost:5000"], // or whichever port you use
        },
    })
    );
    ```
    What does this do?
3. Make changes to your front-end application to trigger violations of this policy.