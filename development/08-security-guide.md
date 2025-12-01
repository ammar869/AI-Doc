# Security Implementation Guide

## Overview
This guide covers implementing comprehensive security measures for the AI Document SaaS application, including authentication security, data protection, API security, and compliance considerations.

## Step 1: Authentication Security

### Enhanced Authentication Middleware

Update `src/middleware.ts`:

```typescript
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { RateLimiterMemory } from 'rate-limiter-flexible'

// Rate limiters for different routes
const authLimiter = new RateLimiterMemory({
  keyGenerator: (req) => req.ip || 'unknown',
  points: 5, // Number of requests
  duration: 60, // Per 60 seconds
})

const apiLimiter = new RateLimiterMemory({
  keyGenerator: (req) => req.ip || 'unknown',
  points: 100, // Number of requests
  duration: 60, // Per 60 seconds
})

const documentLimiter = new RateLimiterMemory({
  keyGenerator: (req) => req.ip || 'unknown',
  points: 200, // Number of requests
  duration: 60, // Per 60 seconds
})

export async function middleware(request: NextRequest) {
  const res = NextResponse.next()
  const supabase = createMiddlewareClient({ req, res })

  const url = request.nextUrl
  const clientIP = request.ip || '127.0.0.1'

  // Rate limiting for API routes
  if (url.pathname.startsWith('/api/')) {
    try {
      await apiLimiter.consume(clientIP)
    } catch {
      return new NextResponse('Too Many Requests', { status: 429 })
    }
  }

  // Rate limiting for document operations
  if (url.pathname.startsWith('/api/documents/')) {
    try {
      await documentLimiter.consume(clientIP)
    } catch {
      return new NextResponse('Too Many Requests', { status: 429 })
    }
  }

  // Rate limiting for auth routes
  if (url.pathname.startsWith('/(auth)')) {
    try {
      await authLimiter.consume(clientIP)
    } catch {
      return new NextResponse('Too Many Requests', { status: 429 })
    }
  }

  const {
    data: { session },
  } = await supabase.auth.getSession()

  // Enhanced route protection
  const protectedRoutes = ['/dashboard', '/editor', '/api']
  const authRoutes = ['/sign-in', '/sign-up']

  const isProtectedRoute = protectedRoutes.some(route => 
    url.pathname.startsWith(route)
  )

  const isAuthRoute = authRoutes.some(route => 
    url.pathname.startsWith(route)
  )

  // Security headers
  const requestHeaders = new Headers(request.headers)
  const userAgent = requestHeaders.get('user-agent') || ''
  
  // Block suspicious user agents
  const suspiciousAgents = ['bot', 'crawler', 'spider']
  const isSuspiciousAgent = suspiciousAgents.some(agent => 
    userAgent.toLowerCase().includes(agent)
  )

  if (isSuspiciousAgent && isProtectedRoute) {
    return new NextResponse('Access Denied', { status: 403 })
  }

  // Handle authentication logic
  if (isProtectedRoute && !session) {
    // Redirect to sign-in for protected routes
    const redirectUrl = new URL('/sign-in', request.url)
    redirectUrl.searchParams.set('redirectedFrom', url.pathname)
    return NextResponse.redirect(redirectUrl)
  }

  if (isAuthRoute && session) {
    // Redirect authenticated users away from auth pages
    const redirectUrl = new URL('/dashboard', request.url)
    return NextResponse.redirect(redirectUrl)
  }

  // Add security headers
  const response = NextResponse.next()
  
  // Content Security Policy
  response.headers.set(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self' https://*.supabase.co https://api.openai.com;"
  )

  // X-Frame-Options
  response.headers.set('X-Frame-Options', 'DENY')

  // X-Content-Type-Options
  response.headers.set('X-Content-Type-Options', 'nosniff')

  // X-XSS-Protection
  response.headers.set('X-XSS-Protection', '1; mode=block')

  // Referrer Policy
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')

  // Permissions Policy
  response.headers.set(
    'Permissions-Policy',
    'camera=(), microphone=(), geolocation=(), payment=()'
  )

  return response
}

export const config = {
  matcher: [
    /*
     * Match all request paths except for the ones starting with:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     */
    '/((?!_next/static|_next/image|favicon.ico|.*\\..*).*)',
  ],
}
```

## Step 2: Input Validation and Sanitization

### Create Security Utilities

Create `src/lib/security/validation.ts`:

```typescript
import { z } from 'zod'
import DOMPurify from 'dompurify'

// Document validation schema
export const documentSchema = z.object({
  title: z.string()
    .min(1, 'Title is required')
    .max(255, 'Title must be less than 255 characters')
    .regex(/^[a-zA-Z0-9\s\-_.!?,'"]+$/, 'Title contains invalid characters'),
  
  content: z.string()
    .max(50000, 'Content too long')
    .optional(),
  
  type: z.enum(['letter', 'report', 'proposal', 'contract', 'memo', 'other']),
  
  status: z.enum(['draft', 'published', 'archived']),
  
  metadata: z.record(z.any()).optional(),
})

// User profile validation schema
export const userProfileSchema = z.object({
  full_name: z.string()
    .min(1, 'Full name is required')
    .max(100, 'Full name too long')
    .regex(/^[a-zA-Z\s\-']+$/, 'Invalid characters in name'),
  
  email: z.string()
    .email('Invalid email address')
    .toLowerCase(),
})

// AI request validation schema
export const aiRequestSchema = z.object({
  prompt: z.string()
    .min(1, 'Prompt is required')
    .max(2000, 'Prompt too long'),
  
  type: z.enum(['generate', 'improve', 'suggest', 'expand', 'summarize']),
  
  documentType: z.string().optional(),
  
  tone: z.string().optional(),
  
  length: z.enum(['short', 'medium', 'long']).optional(),
  
  context: z.string().optional(),
})

// API key validation
export function validateApiKey(key: string): boolean {
  // Basic API key format validation
  const apiKeyPattern = /^[a-zA-Z0-9_-]{32,}$/
  return apiKeyPattern.test(key)
}

// HTML sanitization
export function sanitizeHtml(html: string): string {
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: [
      'p', 'br', 'strong', 'em', 'u', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
      'ul', 'ol', 'li', 'blockquote', 'code', 'pre', 'a', 'span', 'div'
    ],
    ALLOWED_ATTR: [
      'href', 'title', 'class', 'id', 'data-*'
    ],
    ALLOW_DATA_ATTR: true,
    FORBID_ATTR: ['style', 'onclick', 'onerror'],
    FORBID_TAGS: ['script', 'iframe', 'object', 'embed', 'form'],
  })
}

// SQL injection prevention
export function escapeSqlLike(str: string): string {
  return str.replace(/[%_\\]/g, '\\$&')
}

// XSS prevention
export function escapeHtml(text: string): string {
  const map: Record<string, string> = {
    '&': '&',
    '<': '<',
    '>': '>',
    '"': '"',
    "'": '&#x27;',
    '/': '&#x2F;',
  }
  return text.replace(/[&<>"'/]/g, (m) => map[m])
}

// File upload validation
export function validateFileUpload(file: File, allowedTypes: string[] = []): {
  valid: boolean
  error?: string
} {
  // Check file size (max 10MB)
  const maxSize = 10 * 1024 * 1024 // 10MB
  if (file.size > maxSize) {
    return { valid: false, error: 'File too large' }
  }

  // Check file type if specified
  if (allowedTypes.length > 0 && !allowedTypes.includes(file.type)) {
    return { valid: false, error: 'File type not allowed' }
  }

  // Check for malicious file extensions
  const dangerousExtensions = ['.exe', '.bat', '.cmd', '.com', '.pif', '.scr', '.vbs', '.js']
  const fileName = file.name.toLowerCase()
  if (dangerousExtensions.some(ext => fileName.endsWith(ext))) {
    return { valid: false, error: 'Dangerous file type' }
  }

  return { valid: true }
}

// Password strength validation
export function validatePasswordStrength(password: string): {
  valid: boolean
  score: number
  feedback: string[]
} {
  const feedback: string[] = []
  let score = 0

  // Minimum length
  if (password.length < 8) {
    feedback.push('Password must be at least 8 characters long')
  } else {
    score += 1
  }

  // Uppercase letters
  if (!/[A-Z]/.test(password)) {
    feedback.push('Password must contain at least one uppercase letter')
  } else {
    score += 1
  }

  // Lowercase letters
  if (!/[a-z]/.test(password)) {
    feedback.push('Password must contain at least one lowercase letter')
  } else {
    score += 1
  }

  // Numbers
  if (!/\d/.test(password)) {
    feedback.push('Password must contain at least one number')
  } else {
    score += 1
  }

  // Special characters
  if (!/[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]/.test(password)) {
    feedback.push('Password must contain at least one special character')
  } else {
    score += 1
  }

  // Common passwords check
  const commonPasswords = [
    'password', '123456', '123456789', 'qwerty', 'abc123',
    'password123', 'admin', 'letmein', 'welcome', 'monkey'
  ]
  
  if (commonPasswords.includes(password.toLowerCase())) {
    feedback.push('Password is too common')
    score = 0
  }

  return {
    valid: score >= 3 && feedback.length === 0,
    score,
    feedback,
  }
}

// Session validation
export function validateSession(session: any): boolean {
  if (!session || !session.user) return false
  
  // Check if session is expired
  const now = Math.floor(Date.now() / 1000)
  if (session.expires_at && session.expires_at < now) {
    return false
  }
  
  // Check session format
  if (typeof session.user.id !== 'string') return false
  
  return true
}
```

### Create Security Middleware for API Routes

Create `src/middleware/security.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase/server'
import { RateLimiterMemory } from 'rate-limiter-flexible'

// Rate limiter for AI endpoints
const aiLimiter = new RateLimiterMemory({
  keyGenerator: (req) => req.ip || 'unknown',
  points: 10, // Number of requests
  duration: 60, // Per 60 seconds per IP
})

// Rate limiter for document operations
const documentLimiter = new RateLimiterMemory({
  keyGenerator: (req) => req.ip || 'unknown',
  points: 50, // Number of requests
  duration: 60, // Per 60 seconds per IP
})

export async function withSecurity(
  handler: (req: NextRequest) => Promise<NextResponse>,
  options: {
    requireAuth?: boolean
    rateLimit?: 'ai' | 'document' | 'general'
    allowedMethods?: string[]
  } = {}
) {
  return async (req: NextRequest): Promise<NextResponse> => {
    try {
      // Check method if specified
      if (options.allowedMethods && !options.allowedMethods.includes(req.method)) {
        return NextResponse.json(
          { error: 'Method not allowed' },
          { status: 405 }
        )
      }

      // Rate limiting
      if (options.rateLimit) {
        const clientIP = req.ip || 'unknown'
        const limiter = options.rateLimit === 'ai' ? aiLimiter : 
                       options.rateLimit === 'document' ? documentLimiter : null
        
        if (limiter) {
          try {
            await limiter.consume(clientIP)
          } catch {
            return NextResponse.json(
              { error: 'Rate limit exceeded' },
              { status: 429 }
            )
          }
        }
      }

      // Authentication check
      if (options.requireAuth) {
        const supabase = createServerSupabaseClient()
        const { data: { user } } = await supabase.auth.getUser()
        
        if (!user) {
          return NextResponse.json(
            { error: 'Authentication required' },
            { status: 401 }
          )
        }

        // Add user to request context if needed
        (req as any).user = user
      }

      // Additional security checks
      const userAgent = req.headers.get('user-agent') || ''
      
      // Block requests without user agent
      if (!userAgent) {
        return NextResponse.json(
          { error: 'User agent required' },
          { status: 400 }
        )
      }

      // Check for suspicious patterns
      const url = new URL(req.url)
      const suspiciousPatterns = [
        /\.\./,  // Path traversal
        /<script/i,  // XSS attempts
        /union.*select/i,  // SQL injection attempts
        /javascript:/i,  // JavaScript injection
        /vbscript:/i,  // VBScript injection
      ]

      if (suspiciousPatterns.test(url.pathname + url.search)) {
        return NextResponse.json(
          { error: 'Suspicious request pattern detected' },
          { status: 400 }
        )
      }

      return await handler(req)
    } catch (error) {
      console.error('Security middleware error:', error)
      return NextResponse.json(
        { error: 'Internal server error' },
        { status: 500 }
      )
    }
  }
}
```

## Step 3: Database Security

### Enhanced RLS Policies

Update `supabase/migrations/003_enhanced_security.sql`:

```sql
-- Enhanced Row Level Security Policies

-- Drop existing policies first (if any)
DROP POLICY IF EXISTS "Users can view own documents" ON documents;
DROP POLICY IF EXISTS "Users can create own documents" ON documents;
DROP POLICY IF EXISTS "Users can update own documents" ON documents;
DROP POLICY IF EXISTS "Users can delete own documents" ON documents;

-- Create more restrictive policies for documents
CREATE POLICY "Users can view own documents" ON documents
  FOR SELECT USING (
    auth.uid() = user_id AND 
    auth.role() = 'authenticated'
  );

CREATE POLICY "Users can create own documents" ON documents
  FOR INSERT WITH CHECK (
    auth.uid() = user_id AND 
    auth.role() = 'authenticated' AND
    LENGTH(title) <= 255 AND
    LENGTH(content) <= 50000
  );

CREATE POLICY "Users can update own documents" ON documents
  FOR UPDATE USING (
    auth.uid() = user_id AND 
    auth.role() = 'authenticated'
  ) WITH CHECK (
    auth.uid() = user_id AND 
    auth.role() = 'authenticated' AND
    LENGTH(title) <= 255 AND
    LENGTH(content) <= 50000
  );

CREATE POLICY "Users can delete own documents" ON documents
  FOR DELETE USING (
    auth.uid() = user_id AND 
    auth.role() = 'authenticated'
  );

-- AI interactions policies
CREATE POLICY "Users can manage own AI interactions" ON ai_interactions
  FOR ALL USING (
    auth.uid() = user_id AND 
    auth.role() = 'authenticated'
  );

-- Document versions policies
CREATE POLICY "Users can manage own document versions" ON document_versions
  FOR ALL USING (
    EXISTS (
      SELECT 1 FROM documents 
      WHERE documents.id = document_versions.document_id 
      AND documents.user_id = auth.uid()
    ) AND
    auth.role() = 'authenticated'
  );

-- Templates policies
CREATE POLICY "Users can manage own templates" ON document_templates
  FOR ALL USING (
    (auth.uid() = user_id OR is_public = true) AND
    auth.role() = 'authenticated'
  );

-- User preferences policies
CREATE POLICY "Users can manage own preferences" ON user_preferences
  FOR ALL USING (
    auth.uid() = user_id AND 
    auth.role() = 'authenticated'
  );

-- Create audit log table
CREATE TABLE IF NOT EXISTS audit_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id),
  action VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50) NOT NULL,
  resource_id UUID,
  old_values JSONB,
  new_values JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Enable RLS on audit logs
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

-- Only service role can read audit logs
CREATE POLICY "Service role can read audit logs" ON audit_logs
  FOR SELECT USING (auth.role() = 'service_role');

-- Create function to log actions
CREATE OR REPLACE FUNCTION log_action(
  user_id_param UUID,
  action_param VARCHAR,
  resource_type_param VARCHAR,
  resource_id_param UUID DEFAULT NULL,
  old_values_param JSONB DEFAULT NULL,
  new_values_param JSONB DEFAULT NULL
)
RETURNS UUID AS $$
DECLARE
  audit_id UUID;
  client_ip INET;
  user_agent_param TEXT;
BEGIN
  -- Get client IP from request (this would need to be passed from application)
  client_ip := '127.0.0.1'::inet;
  user_agent_param := 'Unknown';
  
  INSERT INTO audit_logs (
    user_id,
    action,
    resource_type,
    resource_id,
    old_values,
    new_values,
    ip_address,
    user_agent
  ) VALUES (
    user_id_param,
    action_param,
    resource_type_param,
    resource_id_param,
    old_values_param,
    new_values_param,
    client_ip,
    user_agent_param
  ) RETURNING id INTO audit_id;
  
  RETURN audit_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Create triggers for automatic logging
CREATE OR REPLACE FUNCTION log_document_changes()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    PERFORM log_action(
      NEW.user_id,
      'CREATE',
      'document',
      NEW.id,
      NULL,
      to_jsonb(NEW)
    );
    RETURN NEW;
  ELSIF TG_OP = 'UPDATE' THEN
    PERFORM log_action(
      NEW.user_id,
      'UPDATE',
      'document',
      NEW.id,
      to_jsonb(OLD),
      to_jsonb(NEW)
    );
    RETURN NEW;
  ELSIF TG_OP = 'DELETE' THEN
    PERFORM log_action(
      OLD.user_id,
      'DELETE',
      'document',
      OLD.id,
      to_jsonb(OLD),
      NULL
    );
    RETURN OLD;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Create triggers
DROP TRIGGER IF EXISTS document_audit_trigger ON documents;
CREATE TRIGGER document_audit_trigger
  AFTER INSERT OR UPDATE OR DELETE ON documents
  FOR EACH ROW EXECUTE FUNCTION log_document_changes();
```

## Step 4: API Security Enhancements

### Secure API Route Example

Create `src/app/api/documents/[id]/route.ts` with security:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase/server'
import { withSecurity } from '@/middleware/security'
import { documentSchema, sanitizeHtml } from '@/lib/security/validation'
import { z } from 'zod'

const updateSchema = documentSchema.partial()

export const GET = withSecurity(async (request: NextRequest) => {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const url = new URL(request.url)
  const documentId = url.pathname.split('/').pop()

  if (!documentId) {
    return NextResponse.json({ error: 'Document ID required' }, { status: 400 })
  }

  // Validate UUID format
  const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i
  if (!uuidRegex.test(documentId)) {
    return NextResponse.json({ error: 'Invalid document ID' }, { status: 400 })
  }

  const { data, error } = await supabase
    .from('documents')
    .select('*')
    .eq('id', documentId)
    .eq('user_id', user.id)
    .single()

  if (error) {
    if (error.code === 'PGRST116') {
      return NextResponse.json({ error: 'Document not found' }, { status: 404 })
    }
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  // Log access
  await supabase.rpc('log_action', {
    user_id_param: user.id,
    action_param: 'VIEW',
    resource_type_param: 'document',
    resource_id_param: documentId,
  })

  return NextResponse.json({ document: data })
}, {
  requireAuth: true,
  rateLimit: 'document',
  allowedMethods: ['GET']
})

export const PUT = withSecurity(async (request: NextRequest) => {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const url = new URL(request.url)
  const documentId = url.pathname.split('/').pop()

  if (!documentId) {
    return NextResponse.json({ error: 'Document ID required' }, { status: 400 })
  }

  // Validate request body
  const body = await request.json()
  const validation = updateSchema.safeParse(body)
  
  if (!validation.success) {
    return NextResponse.json({
      error: 'Invalid request data',
      details: validation.error.errors
    }, { status: 400 })
  }

  const updateData = validation.data

  // Sanitize HTML content if provided
  if (updateData.content) {
    updateData.content = sanitizeHtml(updateData.content)
  }

  // Get current document for version tracking
  const { data: currentDoc } = await supabase
    .from('documents')
    .select('*')
    .eq('id', documentId)
    .eq('user_id', user.id)
    .single()

  if (!currentDoc) {
    return NextResponse.json({ error: 'Document not found' }, { status: 404 })
  }

  // Create version before update
  await supabase
    .from('document_versions')
    .insert({
      document_id: documentId,
      version_number: await getNextVersionNumber(supabase, documentId),
      title: currentDoc.title,
      content: currentDoc.content,
      content_html: currentDoc.content_html,
      created_by: user.id,
    })

  // Update document
  const { data, error } = await supabase
    .from('documents')
    .update({
      ...updateData,
      last_edited_at: new Date().toISOString(),
    })
    .eq('id', documentId)
    .eq('user_id', user.id)
    .select()
    .single()

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  // Log the update
  await supabase.rpc('log_action', {
    user_id_param: user.id,
    action_param: 'UPDATE',
    resource_type_param: 'document',
    resource_id_param: documentId,
    old_values_param: to_jsonb(currentDoc),
    new_values_param: to_jsonb(data),
  })

  return NextResponse.json({ document: data })
}, {
  requireAuth: true,
  rateLimit: 'document',
  allowedMethods: ['PUT']
})

async function getNextVersionNumber(supabase: any, documentId: string): Promise<number> {
  const { data } = await supabase
    .from('document_versions')
    .select('version_number')
    .eq('document_id', documentId)
    .order('version_number', { ascending: false })
    .limit(1)

  return (data?.[0]?.version_number || 0) + 1
}

function to_jsonb(obj: any): any {
  return obj ? JSON.parse(JSON.stringify(obj)) : null
}
```

## Step 5: Data Protection and Privacy

### Create Privacy Utilities

Create `src/lib/security/privacy.ts`:

```typescript
// Data encryption utilities
import crypto from 'crypto'

const ALGORITHM = 'aes-256-gcm'
const IV_LENGTH = 16
const SALT_LENGTH = 64
const TAG_LENGTH = 16
const KEY_LENGTH = 32

export class DataProtection {
  private static key: Buffer

  static initialize(key: string) {
    this.key = crypto.scryptSync(key, 'salt', KEY_LENGTH)
  }

  static encrypt(text: string): string {
    if (!this.key) {
      throw new Error('Data protection not initialized')
    }

    const iv = crypto.randomBytes(IV_LENGTH)
    const salt = crypto.randomBytes(SALT_LENGTH)
    const key = crypto.scryptSync(this.key, salt, KEY_LENGTH)
    
    const cipher = crypto.createCipher(ALGORITHM, key)
    cipher.setAAD(Buffer.from('additional-data'))
    
    let encrypted = cipher.update(text, 'utf8', 'hex')
    encrypted += cipher.final('hex')
    
    const tag = cipher.getAuthTag()
    
    return [
      salt.toString('hex'),
      iv.toString('hex'),
      tag.toString('hex'),
      encrypted
    ].join(':')
  }

  static decrypt(encryptedText: string): string {
    if (!this.key) {
      throw new Error('Data protection not initialized')
    }

    const [saltHex, ivHex, tagHex, encrypted] = encryptedText.split(':')
    
    const salt = Buffer.from(saltHex, 'hex')
    const iv = Buffer.from(ivHex, 'hex')
    const tag = Buffer.from(tagHex, 'hex')
    
    const key = crypto.scryptSync(this.key, salt, KEY_LENGTH)
    
    const decipher = crypto.createDecipher(ALGORITHM, key)
    decipher.setAAD(Buffer.from('additional-data'))
    decipher.setAuthTag(tag)
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8')
    decrypted += decipher.final('utf8')
    
    return decrypted
  }

  static hash(data: string): string {
    return crypto.createHash('sha256').update(data).digest('hex')
  }

  static generateToken(length: number = 32): string {
    return crypto.randomBytes(length).toString('hex')
  }

  static compare(a: string, b: string): boolean {
    return crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b))
  }
}

// GDPR compliance utilities
export class GDPRCompliance {
  static async anonymizeUserData(userId: string): Promise<void> {
    // This would contain logic to anonymize user data
    // while preserving referential integrity
    console.log(`Anonymizing data for user: ${userId}`)
  }

  static async deleteUserData(userId: string): Promise<void> {
    // This would contain logic to delete all user data
    // in compliance with GDPR right to erasure
    console.log(`Deleting all data for user: ${userId}`)
  }

  static async exportUserData(userId: string): Promise<any> {
    // This would contain logic to export all user data
    // in compliance with GDPR right to data portability
    console.log(`Exporting data for user: ${userId}`)
    return {}
  }

  static async updateConsent(userId: string, consent: any): Promise<void> {
    // This would contain logic to update user consent
    console.log(`Updating consent for user: ${userId}`, consent)
  }
}

// Data retention policies
export class DataRetention {
  static async applyRetentionPolicies(): Promise<void> {
    // Apply data retention policies
    // - Delete old audit logs (keep for 1 year)
    // - Delete old AI interactions (keep for 6 months)
    // - Delete old document versions (keep for 3 months)
    console.log('Applying data retention policies')
  }

  static getRetentionPeriod(table: string): number {
    const retentionPeriods: Record<string, number> = {
      audit_logs: 365, // days
      ai_interactions: 180, // days
      document_versions: 90, // days
      documents: -1, // never delete
    }
    
    return retentionPeriods[table] || -1
  }
}

// PII detection and masking
export class PIIProtection {
  static maskPII(text: string): string {
    let masked = text

    // Mask email addresses
    masked = masked.replace(
      /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g,
      (email) => {
        const [local, domain] = email.split('@')
        const maskedLocal = local.charAt(0) + '*'.repeat(local.length - 2) + local.slice(-1)
        return `${maskedLocal}@${domain}`
      }
    )

    // Mask phone numbers
    masked = masked.replace(
      /\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/g,
      (phone) => phone.replace(/\d(?=\d{4})/g, '*')
    )

    // Mask credit card numbers
    masked = masked.replace(
      /\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b/g,
      (card) => card.replace(/\d(?=\d{4})/g, '*')
    )

    return masked
  }

  static detectPII(text: string): {
    emails: string[]
    phones: string[]
    creditCards: string[]
    ssn: string[]
  } {
    const emails = text.match(/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g) || []
    const phones = text.match(/\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/g) || []
    const creditCards = text.match(/\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b/g) || []
    const ssn = text.match(/\b\d{3}[-\s]?\d{2}[-\s]?\d{4}\b/g) || []

    return { emails, phones, creditCards, ssn }
  }
}
```

## Step 6: Security Monitoring

### Create Security Monitoring Service

Create `src/lib/security/monitoring.ts`:

```typescript
interface SecurityEvent {
  type: 'authentication' | 'authorization' | 'data_access' | 'api_abuse' | 'suspicious_activity'
  severity: 'low' | 'medium' | 'high' | 'critical'
  userId?: string
  ipAddress?: string
  userAgent?: string
  description: string
  metadata?: Record<string, any>
  timestamp: Date
}

export class SecurityMonitor {
  private static events: SecurityEvent[] = []

  static log(event: Omit<SecurityEvent, 'timestamp'>): void {
    const securityEvent: SecurityEvent = {
      ...event,
      timestamp: new Date(),
    }

    this.events.push(securityEvent)

    // Log to console in development
    if (process.env.NODE_ENV === 'development') {
      console.warn('Security Event:', securityEvent)
    }

    // In production, this would send to a monitoring service
    this.sendToMonitoringService(securityEvent)

    // Check for suspicious patterns
    this.checkForSuspiciousActivity(securityEvent)
  }

  static logAuthentication(
    userId: string, 
    success: boolean, 
    ipAddress?: string, 
    userAgent?: string
  ): void {
    this.log({
      type: 'authentication',
      severity: success ? 'low' : 'medium',
      userId,
      ipAddress,
      userAgent,
      description: success ? 'User authenticated successfully' : 'Failed authentication attempt',
      metadata: { success }
    })
  }

  static logAuthorization(
    userId: string, 
    action: string, 
    resource: string, 
    success: boolean,
    ipAddress?: string
  ): void {
    this.log({
      type: 'authorization',
      severity: success ? 'low' : 'high',
      userId,
      ipAddress,
      description: `${success ? 'Authorized' : 'Unauthorized'} ${action} access to ${resource}`,
      metadata: { action, resource, success }
    })
  }

  static logDataAccess(
    userId: string, 
    resource: string, 
    action: string,
    recordCount?: number
  ): void {
    this.log({
      type: 'data_access',
      severity: 'low',
      userId,
      description: `User accessed ${recordCount || 0} ${resource} records via ${action}`,
      metadata: { resource, action, recordCount }
    })
  }

  static logAPI Abuse(
    ipAddress: string, 
    endpoint: string, 
    method: string,
    statusCode: number
  ): void {
    this.log({
      type: 'api_abuse',
      severity: statusCode === 429 ? 'medium' : 'high',
      ipAddress,
      description: `API abuse detected: ${method} ${endpoint} returned ${statusCode}`,
      metadata: { endpoint, method, statusCode }
    })
  }

  static logSuspiciousActivity(
    userId: string, 
    description: string,
    ipAddress?: string
  ): void {
    this.log({
      type: 'suspicious_activity',
      severity: 'high',
      userId,
      ipAddress,
      description: `Suspicious activity: ${description}`,
    })
  }

  private static async sendToMonitoringService(event: SecurityEvent): Promise<void> {
    // In a real implementation, this would send to services like:
    // - Datadog
    // - New Relic
    // - Sentry
    // - CloudWatch
    // - Custom monitoring solution
    
    // For now, we'll just store the events
    // In production, you'd integrate with your monitoring service
  }

  private static checkForSuspiciousActivity(event: SecurityEvent): void {
    const recentEvents = this.events.filter(e => 
      e.timestamp > new Date(Date.now() - 5 * 60 * 1000) // Last 5 minutes
    )

    // Check for multiple failed login attempts
    const failedLogins = recentEvents.filter(e => 
      e.type === 'authentication' && 
      e.severity === 'medium' &&
      e.ipAddress === event.ipAddress
    )

    if (failedLogins.length >= 5) {
      this.logSuspiciousActivity(
        event.userId || 'unknown',
        `Multiple failed login attempts from IP: ${event.ipAddress}`,
        event.ipAddress
      )
    }

    // Check for unusual access patterns
    const accessEvents = recentEvents.filter(e => 
      e.type === 'data_access' && 
      e.userId === event.userId
    )

    if (accessEvents.length >= 100) {
      this.logSuspiciousActivity(
        event.userId!,
        `High frequency data access detected: ${accessEvents.length} events in 5 minutes`,
        event.ipAddress
      )
    }
  }

  static getEvents(limit: number = 100): SecurityEvent[] {
    return this.events
      .sort((a, b) => b.timestamp.getTime() - a.timestamp.getTime())
      .slice(0, limit)
  }

  static getEventsByType(type: SecurityEvent['type']): SecurityEvent[] {
    return this.events.filter(e => e.type === type)
  }

  static getEventsBySeverity(severity: SecurityEvent['severity']): SecurityEvent[] {
    return this.events.filter(e => e.severity === severity)
  }

  static async generateSecurityReport(): Promise<{
    totalEvents: number
    eventsByType: Record<string, number>
    eventsBySeverity: Record<string, number>
    recentThreats: SecurityEvent[]
  }> {
    const recentEvents = this.events.filter(e => 
      e.timestamp > new Date(Date.now() - 24 * 60 * 60 * 1000) // Last 24 hours
    )

    const eventsByType = recentEvents.reduce((acc, event) => {
      acc[event.type] = (acc[event.type] || 0) + 1
      return acc
    }, {} as Record<string, number>)

    const eventsBySeverity = recentEvents.reduce((acc, event) => {
      acc[event.severity] = (acc[event.severity] || 0) + 1
      return acc
    }, {} as Record<string, number>)

    const recentThreats = recentEvents.filter(e => 
      e.severity === 'high' || e.severity === 'critical'
    )

    return {
      totalEvents: recentEvents.length,
      eventsByType,
      eventsBySeverity,
      recentThreats,
    }
  }
}
```

## Step 7: Security Headers and HTTPS

### Security Configuration

Create `next.config.js`:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    serverActions: true,
  },
  images: {
    domains: ['supabase.co'],
  },
  env: {
    NEXT_PUBLIC_SUPABASE_URL: process.env.NEXT_PUBLIC_SUPABASE_URL,
    NEXT_PUBLIC_SUPABASE_ANON_KEY: process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
  },
  
  // Security headers
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'X-XSS-Protection',
            value: '1; mode=block',
          },
          {
            key: 'Referrer-Policy',
            value: 'strict-origin-when-cross-origin',
          },
          {
            key: 'Strict-Transport-Security',
            value: 'max-age=31536000; includeSubDomains; preload',
          },
          {
            key: 'Permissions-Policy',
            value: 'camera=(), microphone=(), geolocation=(), payment=()',
          },
          {
            key: 'Content-Security-Policy',
            value: `
              default-src 'self';
              script-src 'self' 'unsafe-eval' 'unsafe-inline' https://*.supabase.co;
              style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
              img-src 'self' data: https: blob:;
              font-src 'self' https://fonts.gstatic.com;
              connect-src 'self' https://*.supabase.co https://api.openai.com wss://*.supabase.co;
              frame-src 'none';
              object-src 'none';
              base-uri 'self';
              form-action 'self';
            `.replace(/\s{2,}/g, ' ').trim(),
          },
        ],
      },
    ]
  },

  // Redirect HTTP to HTTPS in production
  async redirects() {
    return process.env.NODE_ENV === 'production' ? [
      {
        source: '/(.*)',
        has: [
          {
            type: 'host',
            value: 'localhost:3000',
          },
        ],
        destination: 'https://yourdomain.com/:path*',
        permanent: true,
      },
    ] : []
  },
}

module.exports = nextConfig
```

## Step 8: Security Testing

### Security Test Suite

Create `tests/security.test.ts`:

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import { createMockSupabaseClient } from './mocks/supabase'
import DocumentEditor from '@/components/document/superdoc-editor'
import { describe, it, expect, vi, beforeEach } from 'vitest'

// Mock Supabase client
const mockSupabase = createMockSupabaseClient()

vi.mock('@/lib/supabase/client', () => ({
  supabase: mockSupabase,
}))

describe('Security Tests', () => {
  describe('Authentication Security', () => {
    it('should redirect unauthenticated users from protected routes', async () => {
      // Test middleware protection
      const response = await fetch('/dashboard', {
        headers: {
          'Cookie': '', // No session cookie
        },
      })
      
      expect(response.status).toBe(302)
      expect(response.headers.get('Location')).toContain('/sign-in')
    })

    it('should prevent unauthorized document access', async () => {
      // Mock unauthorized document access
      mockSupabase.from.mockReturnValue({
        select: vi.fn().mockReturnValue({
          eq: vi.fn().mockReturnValue({
            single: vi.fn().mockResolvedValue({
              error: { code: 'PGRST116' }, // Not found
            }),
          }),
        }),
      })

      const response = await fetch('/api/documents/unauthorized-id', {
        headers: {
          'Authorization': 'Bearer invalid-token',
        },
      })

      expect(response.status).toBe(401)
    })
  })

  describe('Input Validation', () => {
    it('should sanitize malicious HTML content', () => {
      const { sanitizeHtml } = require('@/lib/security/validation')
      
      const maliciousHtml = '<script>alert("xss")</script><p>Safe content</p>'
      const sanitized = sanitizeHtml(maliciousHtml)
      
      expect(sanitized).not.toContain('<script>')
      expect(sanitized).toContain('<p>Safe content</p>')
    })

    it('should validate document title length', () => {
      const { documentSchema } = require('@/lib/security/validation')
      
      const invalidTitle = 'a'.repeat(256) // Too long
      const result = documentSchema.safeParse({
        title: invalidTitle,
        content: 'Valid content',
        type: 'letter',
        status: 'draft',
      })
      
      expect(result.success).toBe(false)
      if (!result.success) {
        expect(result.error.errors[0].message).toContain('Title must be less than 255 characters')
      }
    })
  })

  describe('Rate Limiting', () => {
    it('should block requests exceeding rate limits', async () => {
      // Simulate multiple rapid requests
      const requests = Array(101).fill(null).map(() => 
        fetch('/api/ai/generate', {
          method: 'POST',
          body: JSON.stringify({ prompt: 'test' }),
        })
      )

      const responses = await Promise.all(requests)
      
      // Some requests should be rate limited
      const rateLimitedResponses = responses.filter(r => r.status === 429)
      expect(rateLimitedResponses.length).toBeGreaterThan(0)
    })
  })

  describe('Data Protection', () => {
    it('should mask PII in exports', () => {
      const { PIIProtection } = require('@/lib/security/privacy')
      
      const textWithPII = 'Email: user@example.com, Phone: 123-456-7890'
      const masked = PIIProtection.maskPII(textWithPII)
      
      expect(masked).not.toContain('user@example.com')
      expect(masked).not.toContain('123-456-7890')
    })

    it('should detect PII in text', () => {
      const { PIIProtection } = require('@/lib/security/privacy')
      
      const textWithPII = 'Contact us at user@example.com or 123-456-7890'
      const detected = PIIProtection.detectPII(textWithPII)
      
      expect(detected.emails).toContain('user@example.com')
      expect(detected.phones).toContain('123-456-7890')
    })
  })
})
```

## Step 9: Security Checklist

### Pre-Deployment Security Checklist

1. **Authentication & Authorization**
   - [ ] Strong password policies implemented
   - [ ] Multi-factor authentication available
   - [ ] Session management secure
   - [ ] Role-based access control (RBAC)
   - [ ] API rate limiting configured

2. **Data Protection**
   - [ ] Sensitive data encryption at rest
   - [ ] Data transmission encrypted (HTTPS/TLS)
   - [ ] PII detection and masking
   - [ ] GDPR compliance measures
   - [ ] Data retention policies

3. **Input Validation & Sanitization**
   - [ ] All inputs validated and sanitized
   - [ ] SQL injection prevention
   - [ ] XSS protection
   - [ ] CSRF protection
   - [ ] File upload security

4. **API Security**
   - [ ] API authentication required
   - [ ] Request/response logging
   - [ ] Error handling secure
   - [ ] API versioning
   - [ ] Monitoring and alerting

5. **Infrastructure Security**
   - [ ] HTTPS enforced
   - [ ] Security headers configured
   - [ ] Firewall rules configured
   - [ ] Regular security updates
   - [ ] Backup and recovery plans

6. **Monitoring & Logging**
   - [ ] Security event logging
   - [ ] Real-time monitoring
   - [ ] Incident response plan
   - [ ] Regular security audits
   - [ ] Vulnerability scanning

## Next Steps

Now that security is implemented, proceed to:
1. [Deployment Guide](./09-deployment-guide.md)

This completes the security implementation guide. You now have comprehensive security measures covering authentication, authorization, data protection, API security, and compliance considerations for your AI Document SaaS application.