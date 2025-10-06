# Applying DaisyUI on the frontend

## Frontend changes
Here, we’ll restyle the existing authentication-based React frontend (Login, Register, Dashboard, etc.) using Tailwind CSS and DaisyUI for a modern, consistent UI.

### Research
Your will need to read up on at least these:
* DaisyUI components and themes
* Tailwind utility classes

### Key requirements
1. Migrate UI to DaisyUI components. Replace plain HTML inputs/buttons with DaisyUI’s input, btn, alert, card, and dropdown.
1. Consistent Layout. Use a global layout (Layout.jsx) with a DaisyUI navbar.
1. Feedback & Validation: Show DaisyUI alert components for success/error, inputs have real-time validation for email and password strength, buttons show loading state with btn loading.
1. Reusable Components

> Note: do not just copy and paste as you may break your app nicely! Use the tutorial carefully.

### Step 1. Install dependencies:
```
npm install -D tailwindcss@3 postcss autoprefixer
npx tailwindcss init -p
```

If the previous step doesn't work, create the files manually.

`postcss.config.js`
```js 
export default {
    plugins: {
      tailwindcss: {},
      autoprefixer: {},
    },
}
```

`tailwind.config.js`
```js
/** @type {import('tailwindcss').Config} */
export default {
    content: [
      "./index.html",
      "./src/**/*.{js,ts,jsx,tsx}",
    ],
    theme: {
      extend: {},
    },
    plugins: [require("daisyui")],
    daisyui: {
        themes: ["dark"],
      }
}
```

Then, install DaisyUI

```
npm install daisyui
```

### Step 2: Add Tailwind to your CSS

Replace src/App.css with 

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

```

Now run your app to test that it has been installed successfully.
You app may look really bad and simple now but we will fix that by replacing all CSS with DaisyUI components.

### Step 3: Update pages

Here, you need update all pages. Here are some snippets to use and extend.

Update `Layout.jsx` - note how we use the classes from DaisyUI now.
```js
<header className="navbar bg-base-200 px-4">
  <div className="flex-1">
    <span className="text-xl font-bold">PulseVote</span>
  </div>
  <nav className="flex gap-2">
    <NavLink to="/" className="btn btn-ghost">Home</NavLink>
    {!loggedIn && (
      <>
        <NavLink to="/login" className="btn btn-ghost">Login</NavLink>
        <NavLink to="/register" className="btn btn-ghost">Register</NavLink>
      </>
    )}
    {loggedIn && (
      <NavLink to="/logout" className="btn btn-ghost">Logout</NavLink>
    )}
  </nav>
</header>
```

Update `DashboardPage.jsx`
```js
return (
  <div className="card bg-base-100 shadow-md w-full max-w-md mx-auto mt-10">
    <div className="card-body">
      <h2 className="card-title">Dashboard</h2>

      {authorized === false && (
        <div className="alert alert-error mt-3">
          <span>You are not authorized to access this content.</span>
        </div>
      )}

      {authorized === true && (
        <div className="alert alert-success mt-3">
          <span>You are authorized and can see this protected content.</span>
        </div>
      )}
    </div>
  </div>
);
```

Update `HomePage.jsx`
```js
export default function HomePage() {
  return (
    <div className="card bg-base-100 shadow-md w-full max-w-md mx-auto mt-10">
      <div className="card-body">
        <h2 className="card-title">Welcome to PulseVote</h2>
        <p>You can login or register using the menu.</p>
      </div>
    </div>
  );
}
```

Update `LoginPage.jsx`
```js
export default function LoginPage() {
  return (
    <div className="card bg-base-100 shadow-md w-full max-w-md mx-auto mt-10">
      <div className="card-body">
        <h2 className="card-title">Login</h2>
        <Login />
      </div>
    </div>
  );
}
```
Reminder: You need to also go ahead an update the RegisterPage and LogoutPage as well.

### Step 4: Update components

Update the Layout component so we can show links applying DaisyUI:
```js
  return (
    <div className="app-shell">
      <header>
        <h1>PulseVote</h1>
        <nav>
          <NavLink to="/" className={({ isActive }) => isActive ? "active" : ""}>Home</NavLink>

          {!loggedIn && (
            <>
              <NavLink to="/login" className={({ isActive }) => isActive ? "active" : ""}>Login</NavLink>
              <NavLink to="/register" className={({ isActive }) => isActive ? "active" : ""}>Register</NavLink>
            </>
          )}

          {loggedIn && (
            <>
              <NavLink to="/dashboard" className={({ isActive }) => isActive ? "active" : ""}>Dashboard</NavLink>
              <NavLink to="/organisations" className={({ isActive }) => isActive ? "active" : ""}>Organisations</NavLink>
              <NavLink to="/manager/create-poll" className={({ isActive }) => isActive ? "active" : ""}>Create Poll</NavLink>
              <NavLink to="/admin" className={({ isActive }) => isActive ? "active" : ""}>Admin</NavLink>
              <NavLink to="/logout" className={({ isActive }) => isActive ? "active" : ""}>Logout</NavLink>
            </>
          )}
        </nav>
      </header>
      <main>
        <Outlet />
      </main>
    </div>
  );
```

Also, update the the `Login` component to use DaisyUI.

```js
import { useState } from "react";
import axios from "axios";
import { useNavigate } from "react-router-dom";
import { jwtDecode } from "jwt-decode";

export default function Login() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [showPassword, setShowPassword] = useState(false);
  const [error, setError] = useState("");
  const [success, setSuccess] = useState("");
  const [loading, setLoading] = useState(false);
  const navigate = useNavigate();

  const isValidEmail = (email) =>
    /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);

  const isStrongPassword = (password) =>
    /^(?=.*[A-Za-z])(?=.*\d)[A-Za-z\d!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]{8,}$/.test(password);

  const login = async () => {
    setError('');
    setSuccess('');
    setLoading(true); // we are adding this for the loading on the button

    if (!email || !password) {
      setError("Email and password are required.");
      setLoading(false);
      return;
    }

    if (!isValidEmail(email)) {
      setError("Invalid email format.");
      setLoading(false);
      return;
    }

    if (!isStrongPassword(password)) {
      setError("Password must be at least 8 characters long, include letters and numbers, and optionally, a special character.");
      setLoading(false);
      return;
    }

    try {
      const res = await axios.post("https://localhost:5000/api/auth/login", { email, password });

      const token = res.data.token;
      localStorage.setItem("token", token);

      const decoded = jwtDecode(token);
 
      const userRole = decoded.roles?.[0]?.role || null;

      if (userRole) {
        localStorage.setItem("role", userRole);
      }
          
      setSuccess("Successfully logged in.");
      setTimeout(() => navigate("/dashboard"), 1000);
    } catch (err) {
      setError(err.response?.data?.message || "Login failed.");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="form-control w-full">
      {error && <div className="alert alert-error mb-3">{error}</div>}
      {success && <div className="alert alert-success mb-3">{success}</div>}

      <label className="label">
        <span className="label-text">Email</span>
      </label>
      <input
        type="email"
        placeholder="Enter your email"
        className="input input-bordered w-full mb-3"
        onChange={(e) => setEmail(e.target.value)}
      />

      <label className="label">
        <span className="label-text">Password</span>
      </label>
      <div className="input-group mb-3">
        <input
          type={showPassword ? "text" : "password"}
          placeholder="Enter your password"
          className="input input-bordered w-full"
          onChange={(e) => setPassword(e.target.value)}
        />
        <button
          type="button"
          className="btn btn-outline"
          onClick={() => setShowPassword(!showPassword)}
        >
          {showPassword ? "Hide" : "Show"}
        </button>
      </div>

      <button
        onClick={login}
        className={`btn btn-primary w-full ${loading ? "loading" : ""}`}
        disabled={loading}
      >
        {loading ? "Logging in..." : "Login"}
      </button>
    </div>
  );
}
```
Remember to apply similar changes to the register component.

### Step 5: Update endpoints
Remember, you have now containerised, deployed etc.
You have also updated endpoints since you last worked on frontend.
You may or may not use HTTPS

Update these in your frontend code:
* Login might be: https://localhost:5000/api/auth/login
* Registering a user might be: "https://localhost:5000/api/auth/register-user"
* Registering an admin or manager will have its endpoint which you will build next.

Once this is done, then you can go ahead and test your application and add more frontend features.