# Firebase-React App Auth Process

To implement user authentication in a React app with Firebase using email/password, Google, and GitHub, I'll break down the steps and explain them in a way that's easy to follo

## The Process:

Firebase Configuration: Set up Firebase project, enable email/password, Google, and GitHub sign-in methods.
AuthContext Setup: Create a context to handle authentication and provide methods for sign-in and sign-out.
UI Components: Create login forms for email/password and buttons for Google and GitHub logins.
Protected Routes: Use ProtectedRoute to guard sensitive routes like the dashboard.
Route Setup: Use react-router-dom to set up navigation and protect specific pages.
This approach adheres to Firebase’s recommended practices and provides a clear, scalable structure for handling multiple authentication methods in a React application.

## Step 1: Setting Up Firebase

[1] Create a Firebase Project:

- Go to the Firebase Console.
- Click on "Add Project" and follow the steps to create a new project.

[2] Enable Authentication Providers:

- Navigate to the "Authentication" section in your Firebase console.
- Under the "Sign-in method" tab, enable the following providers:
  - Email/Password: Toggle this on.
  - Google: Enable this and configure the necessary fields like the project’s OAuth client ID and secret.
  - GitHub: Enable this, and you’ll need to provide the OAuth client ID and secret from GitHub, which can be obtained from the GitHub Developer Settings.

[3] Add Firebase SDK to Your Project:

- Install Firebase in your React project:
  `npm install firebase`
- Set up your Firebase configuration in a `src/firebase/config.js` file:

```js
// src/firebase/config.js
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';

const firebaseConfig = {
	apiKey: 'YOUR_API_KEY',
	authDomain: 'YOUR_PROJECT_ID.firebaseapp.com',
	projectId: 'YOUR_PROJECT_ID',
	storageBucket: 'YOUR_PROJECT_ID.appspot.com',
	messagingSenderId: 'YOUR_MESSAGING_SENDER_ID',
	appId: 'YOUR_APP_ID',
};

const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);
```

# Step 2: Set Up Authentication in React

[1] <strong>Create an Authentication Context</strong>: We'll create a context to manage user authentication state and provide methods for signing in with different providers.

```js
//src/context/AuthContext.js
import React, { createContext, useContext, useEffect, useState } from 'react';
import { auth } from './firebase'; // Import auth from firebase.js
import {
	onAuthStateChanged,
	signInWithEmailAndPassword,
	signOut,
	GoogleAuthProvider,
	GithubAuthProvider,
	signInWithPopup,
} from 'firebase/auth';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
	const [currentUser, setCurrentUser] = useState(null);
	const [loading, setLoading] = useState(true);

	useEffect(() => {
		const unsubscribe = onAuthStateChanged(auth, (user) => {
			setCurrentUser(user);
			setLoading(false);
		});

		return unsubscribe;
	}, []);

	// Sign in with email and password
	const loginWithEmail = (email, password) => {
		return signInWithEmailAndPassword(auth, email, password);
	};

	// Sign in with Google
	const loginWithGoogle = () => {
		const provider = new GoogleAuthProvider();
		return signInWithPopup(auth, provider);
	};

	// Sign in with GitHub
	const loginWithGithub = () => {
		const provider = new GithubAuthProvider();
		return signInWithPopup(auth, provider);
	};

	// Sign out
	const logout = () => {
		return signOut(auth);
	};

	return (
		<AuthContext.Provider
			value={{
				currentUser,
				loginWithEmail,
				loginWithGoogle,
				loginWithGithub,
				logout,
			}}
		>
			{!loading && children}
		</AuthContext.Provider>
	);
};

export const useAuth = () => useContext(AuthContext);
```

[2] <strong>Protect Routes with Context</strong>: Use the AuthContext to protect certain routes in your app. If the user is not authenticated, redirect them to the login page.

```js
// components/ProtectedRoute.js

import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from './AuthContext';

const ProtectedRoute = ({ children }) => {
	const { currentUser } = useAuth();

	if (!currentUser) {
		return <Navigate to='/login' />;
	}

	return children;
};

export default ProtectedRoute;
```

# Step 3: Implement Authentication in the UI

    [1] <strong>Sign-Up and Login Components</strong>: Create separate components for sign-up, login, and social logins.

```js
//src/Pages/Login.js
import React, { useState } from 'react';
import { useAuth } from './AuthContext';
import { useNavigate } from 'react-router-dom';

const Login = () => {
	const [email, setEmail] = useState('');
	const [password, setPassword] = useState('');
	const { loginWithEmail, loginWithGoogle, loginWithGithub } = useAuth();
	const [error, setError] = useState('');
	const navigate = useNavigate();

	const handleLogin = async (e) => {
		e.preventDefault();
		try {
			await loginWithEmail(email, password);
			navigate('/dashboard');
		} catch (err) {
			setError(err.message);
		}
	};

	const handleGoogleLogin = async () => {
		try {
			await loginWithGoogle();
			navigate('/dashboard');
		} catch (err) {
			setError(err.message);
		}
	};

	const handleGithubLogin = async () => {
		try {
			await loginWithGithub();
			navigate('/dashboard');
		} catch (err) {
			setError(err.message);
		}
	};

	return (
		<div>
			<h2>Login</h2>
			{error && <p style={{ color: 'red' }}>{error}</p>}
			<form onSubmit={handleLogin}>
				<input
					type='email'
					placeholder='Email'
					value={email}
					onChange={(e) => setEmail(e.target.value)}
					required
				/>
				<input
					type='password'
					placeholder='Password'
					value={password}
					onChange={(e) => setPassword(e.target.value)}
					required
				/>
				<button type='submit'>Login</button>
			</form>
			<button onClick={handleGoogleLogin}>Login with Google</button>
			<button onClick={handleGithubLogin}>Login with GitHub</button>
		</div>
	);
};

export default Login;
```

```js
// src/Page/SignUp.js
import React, { useState } from 'react';
import { useAuth } from './AuthContext';
import { auth } from './firebase';
import { createUserWithEmailAndPassword, updateProfile } from 'firebase/auth';
import { useNavigate } from 'react-router-dom';

const SignUp = () => {
	const [name, setName] = useState('');
	const [email, setEmail] = useState('');
	const [password, setPassword] = useState('');
	const [confirmPassword, setConfirmPassword] = useState('');
	const [error, setError] = useState('');
	const { currentUser } = useAuth();
	const navigate = useNavigate();

	const handleSignUp = async (e) => {
		e.preventDefault();
		setError('');

		if (password !== confirmPassword) {
			setError('Passwords do not match');
			return;
		}

		try {
			// Create user with email and password
			const userCredential = await createUserWithEmailAndPassword(
				auth,
				email,
				password
			);

			// Set the display name
			await updateProfile(userCredential.user, {
				displayName: name,
			});

			// Send email verification
			await sendEmailVerification(userCredential.user);

			// Navigate to dashboard or any other page after sign-up
			navigate('/dashboard');
		} catch (err) {
			setError(err.message);
		}
	};

	return (
		<div>
			<h2>Sign Up</h2>
			{error && <p style={{ color: 'red' }}>{error}</p>}
			<form onSubmit={handleSignUp}>
				<input
					type='text'
					placeholder='Name'
					value={name}
					onChange={(e) => setName(e.target.value)}
					required
				/>
				<input
					type='email'
					placeholder='Email'
					value={email}
					onChange={(e) => setEmail(e.target.value)}
					required
				/>
				<input
					type='password'
					placeholder='Password'
					value={password}
					onChange={(e) => setPassword(e.target.value)}
					required
				/>
				<input
					type='password'
					placeholder='Confirm Password'
					value={confirmPassword}
					onChange={(e) => setConfirmPassword(e.target.value)}
					required
				/>
				<button type='submit'>Sign Up</button>
			</form>
			<p>
				Already have an account? <a href='/login'>Log In</a>
			</p>
		</div>
	);
};

export default SignUp;
```

[2] <strong>Dashboard Component:</strong> Create a component for the dashboard where authenticated users will be redirected.

```js
// src/Pages/Dashboard.js
import React from 'react';
import { useAuth } from './AuthContext';

const Dashboard = () => {
	const { currentUser, logout } = useAuth();

	return (
		<div>
			<h1>Welcome, {currentUser.displayName || currentUser.email}</h1>
			<button onClick={logout}>Logout</button>
		</div>
	);
};

export default Dashboard;
```

# Step 4: Setting Up Routes

Use react-router-dom to set up your routes and protect the dashboard route.

```js
// App.js
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import { AuthProvider } from './AuthContext';
import Login from './Login';
import SignUp from './SignUp'; // Import the SignUp component
import Dashboard from './Dashboard';
import ProtectedRoute from './ProtectedRoute';

function App() {
	return (
		<AuthProvider>
			<Router>
				<Routes>
					<Route path='/login' element={<Login />} />
					<Route path='/signup' element={<SignUp />} /> {/* Add SignUp route */}
					<Route
						path='/dashboard'
						element={
							<ProtectedRoute>
								<Dashboard />
							</ProtectedRoute>
						}
					/>
				</Routes>
			</Router>
		</AuthProvider>
	);
}

export default App;
```
