---
title: Auth
description: Authentication and authorization patterns for React + Vite + Tailwind apps using Cloud Backend (Supabase). Use when implementing login, signup, password reset, session management, protected routes, Row Level Security, role-based access control, or user data tables.
---

# Authentication Skill for Sticklight Apps

This skill helps you implement user authentication using Cloud Backend.

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
    return null; // Or a loading component
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

### 4.4 Reset Password Page (Set New Password)

```typescript
import { useState } from 'react';
import { useNavigate } from 'react-router';
import { supabase } from '@/integrations/supabase/client';

export default function ResetPassword() {
  const [password, setPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const navigate = useNavigate();

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

    const { error } = await supabase.auth.updateUser({ password });

    if (error) {
      setError(error.message);
    } else {
      navigate('/login');
    }
    setLoading(false);
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
  .select('*')
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
- [ ] Create Reset Password page
- [ ] Add Logout functionality
- [ ] Create first user via signup or Settings -> Users
- [ ] Promote user to admin via Data tab (if roles needed)
- [ ] Add `user_id` to all user-owned tables
- [ ] Enable RLS on all user tables
