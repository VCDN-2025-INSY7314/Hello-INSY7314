# Building PulseVote -  Project Foundations

## Background 
You are beginning work on PulseVote - a secure, real-time polling web app. Over this module, you’ll build it using the MERN stack (MongoDB, Express, React, Node.js), applying professional software engineering and security practices throughout.

This week’s focus is on project setup: scaffolding a secure backend and frontend, establishing good coding hygiene, and getting ready for full-stack development.  

You will also reflect on the importance of security in real-world polling and voting applications.

## Activity Steps

### 1. Understand the Problem
- Research: Spend 10 minutes reviewing live polling/voting platforms (e.g., Kahoot, Mentimeter, Google Forms, Strawpoll).
    - What are their key features?
    - Can you find any news or blog posts about security issues with these apps (e.g., cheating, bot attacks, privacy breaches)?
- Write (3-4 sentences): In your own words, summarize why security matters for live polling apps.

### 2. Build the Backend

- Create a new project folder (e.g., pulsevote-backend) and open it in VS Code.
- Run:
    ```
    npm init -y
    npm install express cors dotenv helmet mongoose express-rate-limit
    npm install --save-dev nodemon
    ```
- Add:
    - app.js (Express setup)
    - server.js (entry point, connects to MongoDB)
- Create folders: models, routes, controllers, middleware
- Create a simple route in app.js:
    ```js
    app.get('/', (req, res) => res.send('PulseVote API running!'));
    ```
- Test your server:
    ```
    npx nodemon server.js
    ```
- Visit http://localhost:5000/ in your browser—you should see a confirmation message.

### 3. Build the Frontend

- In a new folder (e.g., pulsevote-frontend), run:
    ```
    npm create vite@latest pulsevote-frontend -- --template react
    cd pulsevote-frontend
    npm install
    npm install axios react-router-dom
    ```
- Run the frontend:
    ```
    npm run dev
    ```
- Visit the provided local URL. Edit src/App.jsx to display “Welcome to PulseVote”.

### 4. Secure Project Hygiene

- Add a .gitignore file to both backend and frontend folders. Include at least:
    ```
    node_modules
    .env
    dist
    ```
- Create a .env file for backend:
    ```
    MONGO_URI=mongodb://localhost:27017/pulsevote
    JWT_SECRET=changeMeToSomethingStrong
    PORT=5000
    ```
    (Don’t commit this file to Git!)
- Initialize Git repositories in both folders:
    ```
    git init
    git add .
    git commit -m "Initial scaffold"
    ```
    (You may push to your own GitHub if required.)

### 5. Dependency Audit

- Run in the backend:
    ```
    npm audit
    ```
    If any high/critical vulnerabilities are listed, try `npm audit fix` and rerun.
- Do the same for the frontend.

### 6. Deliverables

- For class discussion: 
    - A short written reflection (3–4 sentences):
        - Why is security important for polling apps?
        - List one potential threat to these apps.

- Backend
    - A working backend (Express) that responds at / with a welcome message.
    - Documentation within the README.md file
    - Commit history with at least two commits.
    - .gitignore and .env set up correctly in both projects.
    - Claim backend repo: https://classroom.github.com/a/wib6AI8Z
- Frontend
    - A working frontend (React) that displays “Welcome to PulseVote”.
    - Documentation within the README.md file
    - Commit history with at least two commits.
    - .gitignore and .env set up correctly in both projects.
    - Claim frontend repo: https://classroom.github.com/a/59Pt9kw6    

