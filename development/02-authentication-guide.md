# Authentication Implementation Guide

## Overview
This guide covers implementing secure authentication using Supabase Auth with OAuth providers (Google and GitHub), session management, and protected routes.

## Step 1: Supabase Client Configuration

### Create Supabase Client

Create `src/lib/supabase/client.ts`:

```typescript
import { createClient } from '@supabase/supabase-js'
import { Database } from './types'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient<Database>(supabaseUrl, supabaseAnonKey, {
  auth: {
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: true
  }
})
```

### Create Server-Side Client

Create `src/lib/supabase/server.ts`:

```typescript
import { createServerComponentClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'
import { Database } from './types'

export const createServerSupabaseClient = () => {
  return createServerComponentClient<Database>({ cookies })
}
```

### Create Auth Utilities

Create `src/lib/supabase/auth.ts`:

```typescript
import { supabase } from './client'
import { User } from '@supabase/supabase-js'

export interface AuthUser extends User {
  user_metadata?: {
    full_name?: string
    avatar_url?: string
  }
}

export async function signInWithOAuth(provider: 'google' | 'github') {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider,
    options: {
      redirectTo: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard`,
    },
  })
  return { data, error }
}

export async function signOut() {
  const { error } = await supabase.auth.signOut()
  return { error }
}

export async function getCurrentUser(): Promise<AuthUser | null> {
  const { data: { user } } = await supabase.auth.getUser()
  return user as AuthUser | null
}

export async function getSession() {
  const { data: { session } } = await supabase.auth.getSession()
  return session
}
```

## Step 2: TypeScript Types

Create `src/lib/supabase/types.ts`:

```typescript
export type Json =
  | string
  | number
  | boolean
  | null
  | { [key: string]: Json | undefined }
  | Json[]

export interface Database {
  public: {
    Tables: {
      documents: {
        Row: {
          id: string
          user_id: string
          title: string
          content: string | null
          content_html: string | null
          type: string
          status: string
          word_count: number
          is_template: boolean
          template_category: string | null
          metadata: Json
          created_at: string
          updated_at: string
          last_edited_at: string
        }
        Insert: {
          id?: string
          user_id: string
          title: string
          content?: string | null
          content_html?: string | null
          type?: string
          status?: string
          word_count?: number
          is_template?: boolean
          template_category?: string | null
          metadata?: Json
          created_at?: string
          updated_at?: string
          last_edited_at?: string
        }
        Update: {
          id?: string
          user_id?: string
          title?: string
          content?: string | null
          content_html?: string | null
          type?: string
          status?: string
          word_count?: number
          is_template?: boolean
          template_category?: string | null
          metadata?: Json
          created_at?: string
          updated_at?: string
          last_edited_at?: string
        }
      }
      // Add other table types as needed
    }
  }
}
```

## Step 3: Authentication Components

### Create Sign-In Form

Create `src/components/auth/sign-in-form.tsx`:

```typescript
'use client'

import { useState } from 'react'
import { Button } from '@/components/ui/button'
import { signInWithOAuth } from '@/lib/supabase/auth'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { Loader2 } from 'lucide-react'

export function SignInForm() {
  const [isLoading, setIsLoading] = useState<string | null>(null)

  const handleSignIn = async (provider: 'google' | 'github') => {
    try {
      setIsLoading(provider)
      const { error } = await signInWithOAuth(provider)
      
      if (error) {
        console.error('Sign in error:', error)
        alert('Failed to sign in. Please try again.')
      }
    } catch (error) {
      console.error('Sign in error:', error)
      alert('An unexpected error occurred. Please try again.')
    } finally {
      setIsLoading(null)
    }
  }

  return (
    <Card className="w-full max-w-md mx-auto">
      <CardHeader className="space-y-1">
        <CardTitle className="text-2xl text-center">Sign in to AI Document Creator</CardTitle>
        <CardDescription className="text-center">
          Choose your preferred authentication method
        </CardDescription>
      </CardHeader>
      <CardContent className="space-y-4">
        <Button
          variant="outline"
          className="w-full"
          onClick={() => handleSignIn('google')}
          disabled={isLoading !== null}
        >
          {isLoading === 'google' ? (
            <Loader2 className="mr-2 h-4 w-4 animate-spin" />
          ) : null}
          Continue with Google
        </Button>
        
        <Button
          variant="outline"
          className="w-full"
          onClick={() => handleSignIn('github')}
          disabled={isLoading !== null}
        >
          {isLoading === 'github' ? (
            <Loader2 className="mr-2 h-4 w-4 animate-spin" />
          ) : null}
          Continue with GitHub
        </Button>
      </CardContent>
    </Card>
  )
}
```

### Create Protected Route Component

Create `src/components/auth/protected-route.tsx`:

```typescript
'use client'

import { useEffect, useState } from 'react'
import { useRouter } from 'next/navigation'
import { supabase } from '@/lib/supabase/client'
import { Loader2 } from 'lucide-react'

interface ProtectedRouteProps {
  children: React.ReactNode
}

export function ProtectedRoute({ children }: ProtectedRouteProps) {
  const [loading, setLoading] = useState(true)
  const [authenticated, setAuthenticated] = useState(false)
  const router = useRouter()

  useEffect(() => {
    // Check initial session
    const checkSession = async () => {
      const { data: { session } } = await supabase.auth.getSession()
      
      if (session) {
        setAuthenticated(true)
      } else {
        router.push('/sign-in')
      }
      setLoading(false)
    }

    checkSession()

    // Listen for auth changes
    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      async (event, session) => {
        if (event === 'SIGNED_IN' && session) {
          setAuthenticated(true)
        } else if (event === 'SIGNED_OUT') {
          setAuthenticated(false)
          router.push('/sign-in')
        }
      }
    )

    return () => subscription.unsubscribe()
  }, [router])

  if (loading) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <Loader2 className="h-8 w-8 animate-spin" />
      </div>
    )
  }

  if (!authenticated) {
    return null
  }

  return <>{children}</>
}
```

## Step 4: Authentication Pages

### Create Sign-In Page

Create `src/app/(auth)/sign-in/page.tsx`:

```typescript
import { Metadata } from 'next'
import { SignInForm } from '@/components/auth/sign-in-form'

export const metadata: Metadata = {
  title: 'Sign In - AI Document Creator',
  description: 'Sign in to your AI Document Creator account',
}

export default function SignInPage() {
  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50 py-12 px-4 sm:px-6 lg:px-8">
      <SignInForm />
    </div>
  )
}
```

### Create Callback Page

Create `src/app/(auth)/callback/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'

export async function GET(request: NextRequest) {
  const requestUrl = new URL(request.url)
  const code = requestUrl.searchParams.get('code')
  const next = requestUrl.searchParams.get('next') ?? '/dashboard'

  if (code) {
    const supabase = createRouteHandlerClient({ cookies })
    
    await supabase.auth.exchangeCodeForSession(code)
  }

  return NextResponse.redirect(new URL(next, requestUrl.origin))
}
```

## Step 5: Middleware Configuration

Create `src/middleware.ts`:

```typescript
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export async function middleware(req: NextRequest) {
  const res = NextResponse.next()
  const supabase = createMiddlewareClient({ req, res })

  const {
    data: { session },
  } = await supabase.auth.getSession()

  // Protect dashboard routes
  if (req.nextUrl.pathname.startsWith('/dashboard') && !session) {
    const redirectUrl = req.nextUrl.clone()
    redirectUrl.pathname = '/sign-in'
    return NextResponse.redirect(redirectUrl)
  }

  // Redirect authenticated users away from auth pages
  if (req.nextUrl.pathname.startsWith('/sign-in') && session) {
    const redirectUrl = req.nextUrl.clone()
    redirectUrl.pathname = '/dashboard'
    return NextResponse.redirect(redirectUrl)
  }

  // Protect API routes
  if (req.nextUrl.pathname.startsWith('/api') && !session) {
    return new NextResponse('Unauthorized', { status: 401 })
  }

  return res
}

export const config = {
  matcher: ['/dashboard/:path*', '/sign-in', '/api/:path*'],
}
```

## Step 6: Auth Hook

Create `src/hooks/use-auth.ts`:

```typescript
'use client'

import { useState, useEffect } from 'react'
import { User } from '@supabase/supabase-js'
import { supabase } from '@/lib/supabase/client'
import { AuthUser } from '@/lib/supabase/auth'

export function useAuth() {
  const [user, setUser] = useState<AuthUser | null>(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // Get initial session
    const getSession = async () => {
      const { data: { session } } = await supabase.auth.getSession()
      setUser(session?.user as AuthUser ?? null)
      setLoading(false)
    }

    getSession()

    // Listen for auth changes
    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      async (event, session) => {
        setUser(session?.user as AuthUser ?? null)
        setLoading(false)
      }
    )

    return () => subscription.unsubscribe()
  }, [])

  const signOut = async () => {
    const { error } = await supabase.auth.signOut()
    if (error) {
      console.error('Error signing out:', error)
      throw error
    }
  }

  return {
    user,
    loading,
    signOut,
    isAuthenticated: !!user,
  }
}
```

## Step 7: Authentication API Route

Create `src/app/api/auth/supabase/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'

export async function POST(request: NextRequest) {
  const supabase = createRouteHandlerClient({ cookies })
  const body = await request.json()
  const { action, ...data } = body

  try {
    switch (action) {
      case 'sign_out':
        const { error: signOutError } = await supabase.auth.signOut()
        if (signOutError) throw signOutError
        return NextResponse.json({ success: true })

      case 'get_session':
        const { data: session } = await supabase.auth.getSession()
        return NextResponse.json({ session })

      default:
        return NextResponse.json({ error: 'Invalid action' }, { status: 400 })
    }
  } catch (error) {
    console.error('Auth API error:', error)
    return NextResponse.json(
      { error: 'Authentication error' },
      { status: 500 }
    )
  }
}
```

## Step 8: Update Layout for Authentication

Update `src/app/layout.tsx`:

```typescript
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import './globals.css'

const inter = Inter({ subsets: ['latin'] })

export const metadata: Metadata = {
  title: 'AI Document Creator',
  description: 'Create professional documents with AI assistance',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        {children}
      </body>
    </html>
  )
}
```

## Testing Authentication

### 1. Test Sign-In Flow
- Navigate to `/sign-in`
- Click on Google or GitHub button
- Complete OAuth flow
- Verify redirect to `/dashboard`

### 2. Test Protected Routes
- Try to access `/dashboard` without authentication
- Should redirect to `/sign-in`

### 3. Test Sign-Out
- Click sign-out button in dashboard
- Should redirect to `/sign-in`

## Security Considerations

1. **Environment Variables**: Ensure sensitive keys are properly configured
2. **HTTPS**: Always use HTTPS in production
3. **Session Security**: Supabase handles secure session management
4. **CSRF Protection**: Built into Supabase Auth
5. **Rate Limiting**: Consider implementing rate limiting for auth endpoints

## Common Issues and Solutions

### 1. OAuth Redirect Issues
- Ensure `NEXT_PUBLIC_APP_URL` is correctly set
- Check Supabase project settings for redirect URLs
- Verify OAuth providers are properly configured

### 2. Session Persistence Issues
- Check browser storage for session data
- Ensure cookies are properly set
- Verify Supabase client configuration

### 3. Middleware Issues
- Ensure middleware is properly configured
- Check route patterns in middleware
- Verify middleware execution order

## Next Steps

Now that authentication is implemented, proceed to:
1. [Database Schema Setup Guide](./03-database-setup-guide.md)
2. [Document CRUD Operations Guide](./04-document-crud-guide.md)

This completes the authentication implementation. Users can now securely sign in and access protected routes in your AI Document SaaS application.