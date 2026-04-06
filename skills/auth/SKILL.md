---
title: Auth
version: "1.0"
description: |
  Authentication and authorization patterns for React + Vite + Tailwind
  apps using Cloud Backend (Supabase). Use when implementing login,
  signup, password reset, session management, protected routes, Row
  Level Security (RLS), role-based access control, or user data tables.
  Covers: profiles table with auto-creation trigger, user_roles table,
  AuthContext with getUser() (not getSession()), ProtectedRoute
  component, full auth pages (Login, Signup, Forgot Password, Reset
  Password with expired link handling), logout, user-owned data
  with RLS policies, and security best practices.
---

# Authentication Skill for Sticklight Apps

This skill helps you implement user authentication using Cloud Backend.

## Your Task

When this skill is active, follow these steps:

1. **Assess the current state** — Check if the project already has auth setup (AuthContext, ProtectedRoute, auth pages, database tables)
2. **Set up the database** — Create the `profiles` table with auto-creation trigger, and optionally the `user_roles` table
3. **Create AuthContext** — Build `src/contexts/AuthContext.tsx` with `getUser()`, `signUp`, `signIn`, `signOut`, and `resetPassword`
4. **Wrap the app** — Add `<AuthProvider>` to `src/main.tsx`
5. **Create ProtectedRoute** — Build `src/components/ProtectedRoute.tsx` to guard authenticated pages
6. **Build auth pages** — Create Login, Signup, Forgot Password, and Reset Password pages with proper error handling
7. **Add logout** — Add a sign-out button/flow that clears the session and redirects to login
8. **Secure data tables** — Enable RLS on every user-facing table, add `user_id` column, write proper policies
9. **Verify** — Run through the Auth Checklist (section 10) before shipping

## Overview

Cloud Backend provides built-in authentication with:
- Email/password authentication
- Session management
- Secure token handling
- Row Level Security (RLS) integration

---

## 1. Database Setup

### 1.1 User Profiles Table

Every auth setup should include a `profiles` table to store additional user data.

```sql
CREATE TABLE public.profiles (
  id UUID NOT NULL PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT,
  full_name TEXT,
  avatar_url TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read own profile"
  ON public.profiles FOR SELECT
  USING ((SELECT auth.uid()) = id);

CREATE POLICY "Users can update own profile"
  ON public.profiles FOR UPDATE
  USING ((SELECT auth.uid()) = id);

CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER SET search_path = public
AS $$
BEGIN
  INSERT INTO public.profiles (id, email, full_name)
  VALUES (
    NEW.id,
    NEW.email,
    COALESCE(NEW.raw_user_meta_data->>'full_name', '')
  );
  RETURN NEW;
END;
$$;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

### 1.2 User Roles Table (Optional)

For admin/editor roles, create a separate table. Never store roles in `user_metadata`.

```sql
CREATE TABLE public.user_roles (
  id UUID NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  role TEXT NOT NULL DEFAULT 'user' CHECK (role IN ('user', 'editor', 'admin')),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  UNIQUE(user_id)
);

ALTER TABLE public.user_roles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read own role"
  ON public.user_roles FOR SELECT
  USING ((SELECT auth.uid()) = user_id);
```

---

## 2. Auth Context

Create `src/contexts/AuthContext.tsx`:

```typescript
import { createContext, useContext, useEffect, useState, useCallback, ReactNode, useRef } from 'react';
import { supabase } from '@/integrations/supabase/client';
import type { User } from '@supabase/supabase-js';

interface AuthContextType {
  user: User | null;
  loading: boolean;
  signUp: (email: string, password: string, fullName?: string) => Promise<{ success: boolean; error?: string }>;
  signIn: (email: string, password: string) => Promise<{ success: boolean; error?: string }>;
  signOut: () => Promise<void>;
  resetPassword: (email: string) => Promise<{ success: boolean; error?: string }>;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const initialized = useRef(false);

  useEffect(() => {
    if (!supabase || initialized.current) {
      setLoading(false);
      return;
    }
    initialized.current = true;

    // Get initial user (verified from server)
    supabase.auth.getUser().then(({ data: { user } }) => {
      setUser(user);
      setLoading(false);
    });

    // Listen for auth changes
    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      async (event, session) => {
        if (event === 'SIGNED_OUT') {
          setUser(null);
        } else if (session?.user) {
          const { data: { user } } = await supabase.auth.getUser();
          setUser(user);
        }
      }
    );

    return () => subscription.unsubscribe();
  }, []);

  const signUp = useCallback(async (
    email: string, 
    password: string, 
    fullName?: string
  ): Promise<{ success: boolean; error?: string }> => {
    if (!supabase) return { success: false, error: 'Database not connected' };

    const { data, error } = await supabase.auth.signUp({
      email,
      password,
      options: {
        data: { full_name: fullName }
      }
    });

    if (error) return { success: false, error: error.message };
    if (!data.user) return { success: false, error: 'Signup failed' };

    return { success: true };
  }, []);

  const signIn = useCallback(async (
    email: string, 
    password: string
  ): Promise<{ success: boolean; error?: string }> => {
    if (!supabase) return { success: false, error: 'Database not connected' };

    const { data, error } = await supabase.auth.signInWithPassword({
      email,
      password
    });

    if (error) return { success: false, error: error.message };
    if (!data.user) return { success: false, error: 'Login failed' };

    setUser(data.user);
    return { success: true };
  }, []);

  const signOut = useCallback(async () => {
    if (!supabase) return;
    await supabase.auth.signOut();
    setUser(null);
  }, []);

  const resetPassword = useCallback(async (
    email: string
  ): Promise<{ success: boolean; error?: string }> => {
    if (!supabase) return { success: false, error: 'Database not connected' };

    const { error } = await supabase.auth.resetPasswordForEmail(email, {
      redirectTo: `${window.location.origin}/reset-password`
    });

    if (error) return { success: false, error: error.message };
    return { success: true };
  }, []);

  return (
    <AuthContext.Provider value={{ user, loading, signUp, signIn, signOut, resetPassword }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

Wrap your app in `src/main.tsx`:

```typescript
import { AuthProvider } from '@/contexts/AuthContext';

<AuthProvider>
  <App />
</AuthProvider>
```

---

## 3. Protected Routes

Create `src/components/ProtectedRoute.tsx`:

```typescript
import { useEffect } from 'react';
import { useNavigate } from 'react-router';
import { useAuth } from '@/contexts/AuthContext';

interface ProtectedRouteProps {
  children: React.ReactNode;
  fallbackPath?: string;
}

export function ProtectedRoute({ children, fallbackPath = '/login' }: ProtectedRouteProps) {
  const { user, loading } = useAuth();
  const navigate = useNavigate();

  useEffect(() => {
    if (!loading && !user) {
      navigate(fallbackPath, { replace: true });
    }
  }, [loading, user, navigate, fallbackPath]);

  if (loading) {
    return (
      <div className="flex min-h-screen items-center justify-center">
        <div className="h-8 w-8 animate-spin rounded-full border-4 border-gray-200 border-t-blue-500" />
      </div>
    );
  }

  if (!user) {
    return null;
  }

  return <>{children}</>;
}
```

Usage in routes:

```typescript
<Route path="/dashboard" element={
  <ProtectedRoute>
    <Dashboard />
  </ProtectedRoute>
} />
```

---

## 4. Auth Pages

### 4.1 Login Page

```typescript
import { useState } from 'react';
import { useNavigate } from 'react-router';
import { useAuth } from '@/contexts/AuthContext';

export default function Login() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const navigate = useNavigate();
  const { signIn } = useAuth();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    setLoading(true);

    const result = await signIn(email, password);

    if (result.success) {
      navigate('/dashboard');
    } else {
      setError(result.error || 'Login failed');
    }
    setLoading(false);
  }

  return (
    <form onSubmit={handleSubmit}>
      {error && <div>{error}</div>}
      
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
        required
      />
      
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
        required
      />
      
      <button type="submit" disabled={loading}>
        {loading ? 'Signing in...' : 'Sign In'}
      </button>
    </form>
  );
}
```

### 4.2 Signup Page

```typescript
import { useState } from 'react';
import { useNavigate } from 'react-router';
import { useAuth } from '@/contexts/AuthContext';

export default function Signup() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [fullName, setFullName] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [success, setSuccess] = useState(false);
  const { signUp } = useAuth();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    setLoading(true);

    const result = await signUp(email, password, fullName);

    if (result.success) {
      setSuccess(true);
    } else {
      setError(result.error || 'Signup failed');
    }
    setLoading(false);
  }

  if (success) {
    return <div>Check your email for a confirmation link.</div>;
  }

  return (
    <form onSubmit={handleSubmit}>
      {error && <div>{error}</div>}
      
      <input
        type="text"
        value={fullName}
        onChange={(e) => setFullName(e.target.value)}
        placeholder="Full Name"
      />
      
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
        required
      />
      
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
        minLength={6}
        required
      />
      
      <button type="submit" disabled={loading}>
        {loading ? 'Creating account...' : 'Sign Up'}
      </button>
    </form>
  );
}
```

### 4.3 Forgot Password Page

```typescript
import { useState } from 'react';
import { useAuth } from '@/contexts/AuthContext';

export default function ForgotPassword() {
  const [email, setEmail] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [success, setSuccess] = useState(false);
  const { resetPassword } = useAuth();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    setLoading(true);

    const result = await resetPassword(email);

    if (result.success) {
      setSuccess(true);
    } else {
      setError(result.error || 'Failed to send reset email');
    }
    setLoading(false);
  }

  if (success) {
    return <div>Check your email for a password reset link.</div>;
  }

  return (
    <form onSubmit={handleSubmit}>
      {error && <div>{error}</div>}
      
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
        required
      />
      
      <button type="submit" disabled={loading}>
        {loading ? 'Sending...' : 'Send Reset Link'}
      </button>
    </form>
  );
}
```

### 4.4 Reset Password Page (with Error Handling)

This page handles expired links, invalid tokens, and shows appropriate error messages.

#### Default Email Service

Cloud Backend includes a built-in email service that works out of the box. No SMTP configuration needed.

| Aspect | Default Service |
|--------|-----------------|
| Sender | `noreply@mail.app.supabase.io` |
| Rate limit | ~4 emails/hour per user |
| Link expiration | 1 hour |

For production, consider configuring a custom SMTP for branded emails and higher limits.

#### Error Handling

When a reset link is invalid or expired, Cloud Backend redirects with error parameters in the URL hash:

```
/reset-password#error=access_denied&error_code=otp_expired&error_description=Email+link+is+invalid+or+has+expired
```

**Common error codes:**

| Error Code | Meaning | User Action |
|------------|---------|-------------|
| `otp_expired` | Link expired (default: 1 hour) | Request new link |
| `access_denied` | Link already used or invalid | Request new link |

Always parse the URL hash on page load and show a helpful message with option to request a new link.

#### Component

```typescript
import { useState, useEffect } from 'react';
import { useNavigate, Link } from 'react-router';
import { supabase } from '@/integrations/supabase/client';

export default function ResetPassword() {
  const [password, setPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [success, setSuccess] = useState(false);
  const [sessionReady, setSessionReady] = useState(false);
  const [linkError, setLinkError] = useState<string | null>(null);
  const navigate = useNavigate();

  useEffect(() => {
    // Check for error in URL hash (e.g., expired link)
    const hash = window.location.hash;
    if (hash) {
      const params = new URLSearchParams(hash.substring(1));
      const errorCode = params.get('error_code');
      const errorDescription = params.get('error_description');
      
      if (errorCode || errorDescription) {
        if (errorCode === 'otp_expired') {
          setLinkError('This reset link has expired. Please request a new one.');
        } else {
          setLinkError(errorDescription?.replace(/\+/g, ' ') || 'Invalid reset link.');
        }
        return;
      }
    }

    if (!supabase) return;

    const { data: { subscription } } = supabase.auth.onAuthStateChange((event) => {
      if (event === 'PASSWORD_RECOVERY') {
        setSessionReady(true);
      }
    });

    supabase.auth.getSession().then(({ data: { session } }) => {
      if (session) {
        setSessionReady(true);
      }
    });

    // Timeout if session doesn't become ready
    const timeout = setTimeout(() => {
      if (!sessionReady) {
        setLinkError('Unable to verify reset link. It may have expired.');
      }
    }, 5000);

    return () => {
      subscription.unsubscribe();
      clearTimeout(timeout);
    };
  }, [sessionReady]);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);

    if (password !== confirmPassword) {
      setError('Passwords do not match');
      return;
    }

    if (password.length < 6) {
      setError('Password must be at least 6 characters');
      return;
    }

    setLoading(true);

    if (!supabase) {
      setError('Database not connected');
      setLoading(false);
      return;
    }

    const { error: updateError } = await supabase.auth.updateUser({ password });

    if (updateError) {
      setError(updateError.message);
      setLoading(false);
    } else {
      setSuccess(true);
      await supabase.auth.signOut();
      setTimeout(() => navigate('/login'), 2000);
    }
  }

  // Show error if link is expired/invalid
  if (linkError) {
    return (
      <div>
        <p>{linkError}</p>
        <Link to="/forgot-password">Request New Link</Link>
      </div>
    );
  }

  if (!sessionReady) {
    return <div>Verifying reset link...</div>;
  }

  if (success) {
    return <div>Password updated! Redirecting to login...</div>;
  }

  return (
    <form onSubmit={handleSubmit}>
      {error && <div>{error}</div>}
      
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="New Password"
        minLength={6}
        required
      />
      
      <input
        type="password"
        value={confirmPassword}
        onChange={(e) => setConfirmPassword(e.target.value)}
        placeholder="Confirm Password"
        minLength={6}
        required
      />
      
      <button type="submit" disabled={loading}>
        {loading ? 'Updating...' : 'Update Password'}
      </button>
    </form>
  );
}
```

---

## 5. Logout

```typescript
import { useAuth } from '@/contexts/AuthContext';
import { useNavigate } from 'react-router';

function LogoutButton() {
  const { signOut } = useAuth();
  const navigate = useNavigate();

  async function handleLogout() {
    await signOut();
    navigate('/login');
  }

  return <button onClick={handleLogout}>Sign Out</button>;
}
```

---

## 6. Creating the First User

1. Build the signup page and navigate to `/signup`
2. Create an account with email and password
3. The user is created automatically

Alternatively:
1. Go to **Settings** (gear icon in top menu)
2. Click **Users** in the secondary menu
3. Click **Add User**
4. Enter email and password

---

## 7. Promoting a User to Admin

1. Go to **Data** (cylinder icon in top menu)
2. Select the `user_roles` table
3. Click **Add Row**
4. Set `user_id` (copy from Settings -> Users -> click user -> copy ID)
5. Set `role` to `admin`
6. Save

---

## 8. User Data with RLS

When creating tables that store user-specific data, always include `user_id` and proper RLS policies:

```sql
CREATE TABLE public.user_items (
  id UUID NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

ALTER TABLE public.user_items ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can manage own items"
  ON public.user_items FOR ALL
  USING ((SELECT auth.uid()) = user_id)
  WITH CHECK ((SELECT auth.uid()) = user_id);
```

Query with user filter:

```typescript
const { data } = await supabase
  .from('user_items')
  .select('id, title, created_at')
  .eq('user_id', user.id);
```

---

## 9. Security Rules

1. **Always use `getUser()` for authorization** — it verifies with the server. Never use `getSession()` for security decisions.

2. **Never store roles in user_metadata** — users can modify their own metadata. Use a database table.

3. **Always enable RLS** — every user table must have Row Level Security enabled with proper policies.

4. **Password requirements** — enforce minimum 6 characters (Supabase default). Add additional validation as needed.

5. **Protect routes** — wrap authenticated pages with `ProtectedRoute` component.

---

## Process

When adding auth to a project, follow this workflow:

1. Check if Cloud Backend is enabled — if not, instruct the user to enable it
2. Run the SQL to create `profiles` table, trigger, and RLS policies
3. If roles are needed, run the SQL to create `user_roles` table with RLS
4. Check if `AuthContext` exists — if not, create `src/contexts/AuthContext.tsx` with all auth functions
5. Verify the app is wrapped with `<AuthProvider>` in `src/main.tsx`
6. Check if `ProtectedRoute` exists — if not, create `src/components/ProtectedRoute.tsx`
7. Create auth pages that are missing: Login, Signup, Forgot Password, Reset Password
8. Ensure Reset Password handles expired/invalid links by parsing URL hash errors
9. Add a logout button or flow to the authenticated layout
10. For every new table with user-owned data, add `user_id` column, enable RLS, and write policies
11. Verify: use `getUser()` (not `getSession()`) for all authorization checks
12. Verify: roles are stored in `user_roles` table, never in `user_metadata`
13. Run through the Auth Checklist (section 10)

---

## Output Format

When generating auth setup for a project, include:

1. **SQL migrations** — `profiles` table, trigger, RLS policies, and optionally `user_roles` table
2. **`src/contexts/AuthContext.tsx`** — full context with `useAuth()` hook
3. **`src/components/ProtectedRoute.tsx`** — route guard component
4. **Auth pages** — Login, Signup, Forgot Password, Reset Password (each with loading state, error display, and success handling)
5. **Logout component** — button or menu item that calls `signOut()` and redirects

When generating a new user-owned data table, include:

1. **SQL** — table definition with `user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE`
2. **RLS** — `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` and policies using `(SELECT auth.uid()) = user_id`
3. **Query example** — TypeScript code showing how to query with the user filter

When modifying existing auth code, preserve:

1. The `{ success: boolean; error?: string }` return pattern for all auth operations
2. The `useCallback` wrapping on auth functions
3. The `initialized` ref guard to prevent double-mount issues
4. Null-safe `if (!supabase)` checks

---

## 10. Auth Checklist

- [ ] Enable Cloud Backend
- [ ] Create `profiles` table with auto-creation trigger
- [ ] Create `user_roles` table (if roles needed)
- [ ] Create `AuthContext` with all auth functions
- [ ] Wrap app with `AuthProvider`
- [ ] Create `ProtectedRoute` component
- [ ] Create Login page
- [ ] Create Signup page
- [ ] Create Forgot Password page
- [ ] Create Reset Password page (with expired link handling)
- [ ] Add Logout functionality
- [ ] Create first user via signup or Settings -> Users
- [ ] Promote user to admin via Data tab (if roles needed)
- [ ] Add `user_id` to all user-owned tables
- [ ] Enable RLS on all user tables
