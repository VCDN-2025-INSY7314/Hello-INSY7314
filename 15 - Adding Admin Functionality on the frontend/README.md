# Adding Admin Functionality on the frontend

## Frontend changes
Here, we’ll add the ability to allow admins to add managers - this guide includes code, menus, tweaking how we use JWT.

Then, you will add functionality of your own for the relevant endpoints including:
* Add an admin
* Add a manager

> Note: do not just copy and paste as you may break your app nicely! Use the tutorial carefully.

### Step 1: Make the AddManagerForm component

Add a `AddManagerForm.jsx` to your components folder.
Make sure to use the correct endpoint (register-manager).

```js 
import { useState } from "react";
import axios from "axios";

export default function AddManagerForm() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [showPassword, setShowPassword] = useState(false);
  const [error, setError] = useState("");
  const [success, setSuccess] = useState("");
  const [loading, setLoading] = useState(false);

  const isValidEmail = (email) =>
    /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);

  const isStrongPassword = (password) =>
    /^(?=.*[A-Za-z])(?=.*\d)[A-Za-z\d!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]{8,}$/.test(password);

  const addManager = async () => {
    setError("");
    setSuccess("");
    setLoading(true);

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
      setError(
        "Password must be at least 8 characters long, include letters and numbers, and optionally, a special character."
      );
      setLoading(false);
      return;
    }

    try {
      await axios.post("https://localhost:5000/api/auth/register-manager", {
          email,
          password 
        },
        {
          headers: { Authorization: `Bearer ${localStorage.getItem("token")}` },
        }
      );
      setSuccess("Manager registered successfully");
      setEmail("");
      setPassword("");
    } catch (err) {
      setError(err.response?.data?.message || "Failed to add manager.");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="form-control w-full">
      {error && <div className="alert alert-error mb-3">{error}</div>}
      {success && <div className="alert alert-success mb-3">{success}</div>}

      <label className="label">
        <span className="label-text">Manager Email</span>
      </label>
      <input
        type="email"
        placeholder="Enter manager email"
        className="input input-bordered w-full mb-3"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />

      <label className="label">
        <span className="label-text">Password</span>
      </label>
      <div className="input-group mb-3">
        <input
          type={showPassword ? "text" : "password"}
          placeholder="Enter manager password"
          className="input input-bordered w-full"
          value={password}
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
        onClick={addManager}
        className={`btn btn-primary w-full ${loading ? "loading" : ""}`}
        disabled={loading}
      >
        {loading ? "Registering..." : "Add Manager"}
      </button>
    </div>
  );
}
```

### Step 2: Add the page

```js
import AddManagerForm from "../components/AddManagerForm";

export default function AddManagerPage() {
  return (
    <div className="card bg-base-100 shadow-md w-full max-w-md mx-auto mt-10">
      <div className="card-body">
        <h2 className="card-title">Add Manager</h2>
        <AddManagerForm />
      </div>
    </div>
  );
}
```

Run your app and test it.

### Step 3: Amend the login to use contents token of the token to get role of the user.
Since we want to show different menus based on roles (manager, admin, etc.), we need to read the role from the token.

Update `Login.js` to use the contents of the token. 

```js
import { jwtDecode } from "jwt-decode";

// other code

// amend API call similar to below...
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
```

Now that the role is in localSotage, we can reuse this role elsewhere.

### Step 4: Amend the ProtectedRoute to also use role

`ProtectedRoute.jsx`
```js
import { Navigate } from "react-router-dom";
import { jwtDecode } from "jwt-decode";

export default function ProtectedRoute({ children, roleRequired }) {
  const token = localStorage.getItem("token");

  if (!token) {
    return <Navigate to="/login" />;
  }

  try {
    const decoded = jwtDecode(token);

    const userRole = decoded.roles?.[0]?.role || null;

    if (roleRequired && userRole !== roleRequired) {
      return <div className="alert alert-error">Unauthorized</div>;
    }

    return children;
  } catch (e) {
    return <Navigate to="/login" />;
  }
}
```

### Step 5: Amend the layout (navigation) use the role
Here, we can show a different menu based on role - some features are not built out (you need to build them)
```js
import { Outlet, NavLink, useNavigate, useLocation } from "react-router-dom";
import { useEffect, useState } from "react";
import { jwtDecode } from "jwt-decode";

export default function Layout() {
  const [loggedIn, setLoggedIn] = useState(false);
  const [role, setRole] = useState(null);
  const [email, setEmail] = useState(null);
  const navigate = useNavigate();
  const location = useLocation();

  useEffect(() => {
    const token = localStorage.getItem("token");
    setLoggedIn(!!token);

    if (token) {
      try {
        const decoded = jwtDecode(token);

        const userRole = decoded.roles?.[0]?.role || null;
        setRole(userRole);

        setEmail(decoded.email || null);
      } catch (e) {
        setRole(null);
        setEmail(null);
      }
    } else {
      setRole(null);
      setEmail(null);
    }
  }, [location]);

  const handleLogout = () => {
    localStorage.removeItem("token");
    setLoggedIn(false);
    setRole(null);
    setEmail(null);
    navigate("/");
  };

  const displayName = email ? email.split("@")[0] : "Account";

  return (
    <div className="app-shell">
      <header className="navbar bg-base-200 px-4">
        <div className="flex-1">
          <NavLink to="/" className="text-xl font-bold">PulseVote</NavLink>
        </div>

        <nav className="flex gap-2 items-center">
          <NavLink to="/" className="btn btn-ghost">Home</NavLink>
          {!loggedIn && (
            <>
              <NavLink to="/login" className="btn btn-ghost">Login</NavLink>
              <NavLink to="/register" className="btn btn-ghost">Register</NavLink>
            </>
          )}
          {loggedIn && (
            <>
              <NavLink to="/dashboard" className="btn btn-ghost">Dashboard</NavLink>

              {role === "admin" && (
                <div className="dropdown dropdown-end">
                  <label tabIndex={0} className="btn btn-ghost">
                    Add Users
                  </label>
                  <ul
                    tabIndex={0}
                    className="dropdown-content menu p-2 shadow bg-base-100 rounded-box w-52"
                  >
                    <li>
                      <NavLink to="/add-manager">Add Manager</NavLink>
                    </li>
                    <li>
                      <NavLink to="/add-admin">Add Admin</NavLink>
                    </li>
                  </ul>
                </div>
              )}

              {role === "manager" && (
                <NavLink to="/manager-tools" className="btn btn-ghost">Manager Tools</NavLink>
              )}

              <div className="dropdown dropdown-end">
                <label tabIndex={0} className="btn btn-ghost">
                  {displayName}
                </label>
                <ul
                  tabIndex={0}
                  className="dropdown-content menu p-2 shadow bg-base-100 rounded-box w-40"
                >
                  <li className="menu-title">{email}</li>
                  <li>
                    <button onClick={handleLogout}>Logout</button>
                  </li>
                </ul>
              </div>
            </>
          )}
        </nav>
      </header>
      <main>
        <Outlet />
      </main>
    </div>
  );
}
```

### Step 6: Adjusting the routes to include a route protected by role
You can update `App.jsx` routes to include a route that uses the role.
Here we are adding the add-manager route

```js
import AddManagerPage from "./pages/AddManagerPage";

// other code
function App() {
  return (
    <Routes>
      <Route path="/" element={<Layout />}>
        <Route index element={<HomePage />} />
        <Route path="login" element={<LoginPage />} />
        <Route path="register" element={<RegisterPage />} />
        <Route path="logout" element={<LogoutPage />} />
        <Route path="dashboard" element={
          <ProtectedRoute>
            <DashboardPage />
          </ProtectedRoute>
        } />
        <Route
          path="add-manager"
          element={
            <ProtectedRoute roleRequired="admin">
              <AddManagerPage />
            </ProtectedRoute>
          }
        />
      </Route>
    </Routes>
    );
}
```

Run your app and build out the rest of the funtionality.