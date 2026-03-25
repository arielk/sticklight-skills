---
name: auth
description: Authentication and authorization patterns for React + Vite + Tailwind apps using Supabase Auth. Use when implementing login, signup, password reset, OAuth providers, session management, protected routes, Row Level Security, or role-based access control with Supabase.
---

# Auth with Supabase + React

Implement authentication and authorization using Supabase Auth in React + Vite + Tailwind CSS applications.

## Supabase Client Setup

Create a single shared Supabase client instance:

```ts
// src/lib/supabase.ts
import { createClient } from '@supabase/supabase-js';

export const supabase = createClient(
  import.meta.env.VITE_SUPABASE_URL,
  import.meta.env.VITE_SUPABASE_ANON_KEY,
);
```

Environment variables in `.env`:

```
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key
```

- Never expose the `service_role` key in client-side code — it bypasses RLS
- The `anon` key is safe to use client-side because Row Level Security enforces access control
- Prefix env vars with `VITE_` so Vite exposes them to the client bundle

## Auth Context & Provider

Wrap the app in an auth context so any component can access the current user and session:

```tsx
// src/contexts/AuthContext.tsx
import { createContext, useContext, useEffect, useState } from 'react';
import type { User, Session } from '@supabase/supabase-js';
import { supabase } from '../lib/supabase';

interface AuthState {
  user: User | null;
  session: Session | null;
  loading: boolean;
}

const AuthContext = createContext<AuthState>({
  user: null,
  session: null,
  loading: true,
});

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [state, setState] = useState<AuthState>({
    user: null,
    session: null,
    loading: true,
  });

  useEffect(() => {
    supabase.auth.getSession().then(({ data: { session } }) => {
      setState({ user: session?.user ?? null, session, loading: false });
    });

    const {
      data: { subscription },
    } = supabase.auth.onAuthStateChange((_event, session) => {
      setState({ user: session?.user ?? null, session, loading: false });
    });

    return () => subscription.unsubscribe();
  }, []);

  return <AuthContext.Provider value={state}>{children}</AuthContext.Provider>;
}

export const useAuth = () => useContext(AuthContext);
```

Mount it at the app root:

```tsx
// main.tsx
import { AuthProvider } from './contexts/AuthContext';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <AuthProvider>
    <App />
  </AuthProvider>
);
```

## Auth Operations

### Email/Password Sign Up

```tsx
async function signUp(email: string, password: string) {
  const { data, error } = await supabase.auth.signUp({ email, password });
  if (error) throw error;
  return data;
}
```

### Email/Password Sign In

```tsx
async function signIn(email: string, password: string) {
  const { data, error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });
  if (error) throw error;
  return data;
}
```

### OAuth (Google, GitHub, etc.)

```tsx
async function signInWithProvider(provider: 'google' | 'github' | 'discord') {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider,
    options: {
      redirectTo: `${window.location.origin}/auth/callback`,
    },
  });
  if (error) throw error;
  return data;
}
```

Handle the callback route:

```tsx
// src/pages/AuthCallback.tsx
import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { supabase } from '../lib/supabase';

export function AuthCallback() {
  const navigate = useNavigate();

  useEffect(() => {
    supabase.auth.onAuthStateChange((event) => {
      if (event === 'SIGNED_IN') {
        navigate('/dashboard', { replace: true });
      }
    });
  }, [navigate]);

  return (
    <div className="flex min-h-screen items-center justify-center">
      <p className="text-gray-500">Signing you in...</p>
    </div>
  );
}
```

### Magic Link (Passwordless)

```tsx
async function signInWithMagicLink(email: string) {
  const { error } = await supabase.auth.signInWithOtp({
    email,
    options: { emailRedirectTo: `${window.location.origin}/auth/callback` },
  });
  if (error) throw error;
}
```

### Password Reset

```tsx
async function resetPassword(email: string) {
  const { error } = await supabase.auth.resetPasswordForEmail(email, {
    redirectTo: `${window.location.origin}/auth/update-password`,
  });
  if (error) throw error;
}

async function updatePassword(newPassword: string) {
  const { error } = await supabase.auth.updateUser({ password: newPassword });
  if (error) throw error;
}
```

### Sign Out

```tsx
async function signOut() {
  const { error } = await supabase.auth.signOut();
  if (error) throw error;
}
```

## Protected Routes

Use a wrapper component with React Router:

```tsx
// src/components/ProtectedRoute.tsx
import { Navigate, Outlet } from 'react-router-dom';
import { useAuth } from '../contexts/AuthContext';

export function ProtectedRoute() {
  const { user, loading } = useAuth();

  if (loading) {
    return (
      <div className="flex min-h-screen items-center justify-center">
        <div className="h-8 w-8 animate-spin rounded-full border-4 border-blue-500 border-t-transparent" />
      </div>
    );
  }

  return user ? <Outlet /> : <Navigate to="/login" replace />;
}
```

Wire it up in the router:

```tsx
import { createBrowserRouter } from 'react-router-dom';
import { ProtectedRoute } from './components/ProtectedRoute';

const router = createBrowserRouter([
  { path: '/login', element: <LoginPage /> },
  { path: '/signup', element: <SignupPage /> },
  { path: '/auth/callback', element: <AuthCallback /> },
  {
    element: <ProtectedRoute />,
    children: [
      { path: '/dashboard', element: <DashboardPage /> },
      { path: '/settings', element: <SettingsPage /> },
    ],
  },
]);
```

## Row Level Security (RLS)

RLS is the backbone of Supabase authorization. Always enable it on every table and write policies that enforce access.

### Enable RLS

```sql
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
```

### Common Policy Patterns

**Users can read their own data:**

```sql
CREATE POLICY "Users read own data"
  ON posts FOR SELECT
  USING (auth.uid() = user_id);
```

**Users can insert their own data:**

```sql
CREATE POLICY "Users insert own data"
  ON posts FOR INSERT
  WITH CHECK (auth.uid() = user_id);
```

**Users can update their own data:**

```sql
CREATE POLICY "Users update own data"
  ON posts FOR UPDATE
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);
```

**Users can delete their own data:**

```sql
CREATE POLICY "Users delete own data"
  ON posts FOR DELETE
  USING (auth.uid() = user_id);
```

**Public read access (e.g. published blog posts):**

```sql
CREATE POLICY "Public read published posts"
  ON posts FOR SELECT
  USING (published = true);
```

### Role-Based Access with Custom Claims

Store roles in a `profiles` table and reference them in policies:

```sql
CREATE TABLE profiles (
  id UUID REFERENCES auth.users(id) PRIMARY KEY,
  role TEXT NOT NULL DEFAULT 'user' CHECK (role IN ('user', 'editor', 'admin'))
);

ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users read own profile"
  ON profiles FOR SELECT
  USING (auth.uid() = id);
```

Use in other table policies:

```sql
CREATE POLICY "Admins can do anything on posts"
  ON posts FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM profiles
      WHERE profiles.id = auth.uid()
      AND profiles.role = 'admin'
    )
  );
```

### Auto-Create Profile on Signup

Use a database trigger to create a profile row when a new user signs up:

```sql
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO profiles (id, role)
  VALUES (NEW.id, 'user');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW
  EXECUTE FUNCTION handle_new_user();
```

## Role-Based UI

Check the user's role in React components to conditionally render UI:

```tsx
import { useEffect, useState } from 'react';
import { useAuth } from '../contexts/AuthContext';
import { supabase } from '../lib/supabase';

export function useUserRole() {
  const { user } = useAuth();
  const [role, setRole] = useState<string | null>(null);

  useEffect(() => {
    if (!user) { setRole(null); return; }

    supabase
      .from('profiles')
      .select('role')
      .eq('id', user.id)
      .single()
      .then(({ data }) => setRole(data?.role ?? 'user'));
  }, [user]);

  return role;
}
```

```tsx
function AdminPanel() {
  const role = useUserRole();
  if (role !== 'admin') return null;

  return (
    <div className="rounded-lg border border-red-200 bg-red-50 p-4">
      <h2 className="text-lg font-semibold text-red-800">Admin Panel</h2>
      {/* admin-only controls */}
    </div>
  );
}
```

**Never rely on client-side role checks alone.** RLS policies on the database are the real enforcement layer. Client-side checks are only for UX (hiding buttons, showing admin UI).

## Security Checklist

- [ ] `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` are the only Supabase credentials in client code
- [ ] `service_role` key is only used in server-side code (Edge Functions, API routes)
- [ ] RLS is enabled on **every** table
- [ ] Every table has at least one policy (a table with RLS enabled and no policies denies all access)
- [ ] OAuth redirect URLs are configured in Supabase dashboard under Authentication > URL Configuration
- [ ] Email templates are customized in Supabase dashboard
- [ ] Password requirements are configured in Supabase dashboard
- [ ] Session handling listens to `onAuthStateChange` to react to token refresh and sign-out events
- [ ] Protected routes redirect unauthenticated users to login
- [ ] Auth errors are displayed to the user with clear, non-technical messages
