# Adding Authentication on the front-end 

## Frontend changes
Here, we will build the ability to login, register, logout, and view a dashboard if successfully logged in.

### Getting started

Suggested structure:
```
src/
├── components/
│   ├── Layout.jsx
│   ├── Login.jsx
│   ├── ProtectedRoute.jsx
│   ├── Register.jsx
├── pages/
│   ├── DashboardPage.jsx
│   ├── HomePage.jsx
│   ├── LoginPage.jsx
│   ├── LogoutPage.jsx
│   ├── RegisterPage.jsx
```

Install dependencies:
```
npm install react-router-dom axios
```
### Research
Your will need to read up on at least these:
* Axios
* localStorage
* Using ReactRouter
* UseEffect and useState
* Input sanitization

### Key requirements
Now, you need to continue building your front-end - here are the key requirements:
1. Include a register page that allows simple registration with email and password.
1. Let a user login with their username and password.
1. Have a Dashboard page that only authenticated users may access.
1. Update the menu. When logged in - show Home, Dashboard and Logout. When logged out - show Home, Register and Login.
1. Show the appropriate error and confirmation messages. e.g. successful/unsuccessful login, registered, need to login, etc.
1. Use consistent layout and styling across pages.
1. Your front-end must not interact with MongoDB directly.
1. Use Axios to make requests to the API.
