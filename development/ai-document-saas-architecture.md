# AI Document Creation SaaS - Technical Architecture Plan

## Table of Contents
1. [Project Structure & Organization](#1-project-structure--organization)
2. [Technology Stack](#2-technology-stack)
3. [Authentication Architecture](#3-authentication-architecture)
4. [Database Schema Design](#4-database-schema-design)
5. [AI Document Creation System](#5-ai-document-creation-system)
6. [API Routes & Middleware Structure](#6-api-routes--middleware-structure)
7. [Security Considerations](#7-security-considerations)
8. [Development & Deployment Setup](#8-development--deployment-setup)
9. [Configuration Templates](#9-configuration-templates)

---

## 1. Project Structure & Organization

```
ai-document-saas/
├── README.md
├── package.json
├── next.config.js
├── tailwind.config.js
├── postcss.config.js
├── tsconfig.json
├── .env.local
├── .env.example
├── vercel.json
├── supabase/
│   ├── config.toml
│   ├── migrations/
│   └── seed.sql
├── public/
│   ├── favicon.ico
│   ├── logo.png
│   └── og-image.jpg
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── sign-in/
│   │   │   │   └── page.tsx
│   │   │   ├── sign-up/
│   │   │   │   └── page.tsx
│   │   │   └── callback/
│   │   │       └── route.ts
│   │   ├── (dashboard)/
│   │   │   ├── dashboard/
│   │   │   │   ├── page.tsx
│   │   │   │   ├── layout.tsx
│   │   │   │   └── documents/
│   │   │   │       ├── page.tsx
│   │   │   │       └── [id]/
│   │   │   │           ├── page.tsx
│   │   │   │           └── edit/
│   │   │   │               └── page.tsx
│   │   │   ├── editor/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [documentId]/
│   │   │   │       └── page.tsx
│   │   │   └── settings/
│   │   │       ├── page.tsx
│   │   │       └── account/
│   │   │           └── page.tsx
│   │   ├── api/
│   │   │   ├── auth/
│   │   │   │   └── supabase/
│   │   │   │       └── route.ts
│   │   │   ├── documents/
│   │   │   │   ├── route.ts
│   │   │   │   ├── [id]/
│   │   │   │   │   └── route.ts
│   │   │   │   ├── [id]/
│   │   │   │   │   └── export/
│   │   │   │   │       └── route.ts
│   │   │   │   └── [id]/
│   │   │   │       └── share/
│   │   │   │           └── route.ts
│   │   │   ├── ai/
│   │   │   │   ├── generate/
│   │   │   │   │   └── route.ts
│   │   │   │   ├── improve/
│   │   │   │   │   └── route.ts
│   │   │   │   ├── suggest/
│   │   │   │   │   └── route.ts
│   │   │   │   └── export/
│   │   │   │       └── route.ts
│   │   │   ├── templates/
│   │   │   │   └── route.ts
│   │   │   └── webhooks/
│   │   │       └── supabase/
│   │   │           └── route.ts
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── ui/                    # Reusable UI components
│   │   │   ├── button.tsx
│   │   │   ├── input.tsx
│   │   │   ├── card.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── editor/
│   │   │   │   ├── toolbar.tsx
│   │   │   │   ├── menu.tsx
│   │   │   │   └── index.ts
│   │   │   └── index.ts
│   │   ├── auth/                  # Authentication components
│   │   │   ├── sign-in-form.tsx
│   │   │   ├── sign-up-form.tsx
│   │   │   ├── protected-route.tsx
│   │   │   └── index.ts
│   │   ├── document/              # Document components
│   │   │   ├── document-card.tsx
│   │   │   ├── document-list.tsx
│   │   │   ├── document-editor.tsx
│   │   │   ├── ai-assistant.tsx
│   │   │   └── index.ts
│   │   └── dashboard/             # Dashboard components
│   │       ├── sidebar.tsx
│   │       ├── header.tsx
│   │       ├── user-menu.tsx
│   │       ├── stats-card.tsx
│   │       └── index.ts
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts
│   │   │   ├── server.ts
│   │   │   ├── auth.ts
│   │   │   └── types.ts
│   │   ├── ai/
│   │   │   ├── openai.ts
│   │   │   ├── prompts.ts
│   │   │   ├── templates.ts
│   │   │   └── index.ts
│   │   ├── document/
│   │   │   ├── processor.ts
│   │   │   ├── exporter.ts
│   │   │   ├── parser.ts
│   │   │   └── index.ts
│   │   ├── utils/
│   │   │   ├── validation.ts
│   │   │   ├── formatting.ts
│   │   │   ├── storage.ts
│   │   │   └── index.ts
│   │   └── constants/
│   │       ├── document-types.ts
│   │       ├── ai-models.ts
│   │       └── index.ts
│   ├── hooks/
│   │   ├── use-document.ts
│   │   ├── use-ai-assistant.ts
│   │   ├── use-auth.ts
│   │   └── index.ts
│   ├── types/
│   │   ├── document.ts
│   │   ├── user.ts
│   │   ├── ai.ts
│   │   └── index.ts
│   └── middleware.ts
├── docs/
│   ├── API.md
│   ├── DEPLOYMENT.md
│   ├── AI-INTEGRATION.md
│   └── DEVELOPMENT.md
└── tests/
    ├── components/
    ├── pages/
    └── utils/
```

---

## 2. Technology Stack

### Core Technologies
- **Next.js 14/15**: React framework with App Router
- **TypeScript**: Type safety and developer experience
- **Supabase**: Backend-as-a-Service (PostgreSQL + Auth + Storage)
- **Tailwind CSS**: Utility-first styling
- **Shadcn/ui**: Component library

### Authentication & Backend
- **Supabase Auth**: Authentication with OAuth providers
- **Supabase Database**: PostgreSQL database
- **Supabase Storage**: File storage for document exports
- **Row Level Security (RLS)**: Database security

### AI Integration
- **OpenAI API**: GPT models for text generation
- **Anthropic Claude**: Alternative AI provider
- **Document Processing**: Rich text editing and formatting

### Document Processing
- **Superdoc**

### Development & Deployment
- **Vercel**: Primary deployment platform
- **Supabase CLI**: Local development
- **GitHub Actions**: CI/CD pipeline
- **ESLint & Prettier**: Code quality

---

## 3. Authentication Architecture

### 3.1 Supabase Client Configuration

```typescript
// src/lib/supabase/client.ts
import { createClient } from '@supabase/supabase-js'
import { Database } from './types'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient<Database>(supabaseUrl, supabase {
  auth: {
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: true
  }
})

// src/lib/supabase/server.ts
import { createServerComponentClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'

export const createServerSupabaseClient = () => {
  return createServerComponentClient<Database>({ cookies })
}

// src/lib/supabase/auth.ts
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
```

### 3.2 Authentication Flow

1. **OAuth Sign-in**:
   - User clicks "Sign in with Google/GitHub"
   - Redirected to Supabase OAuth flow
   - Callback to dashboard with authenticated session
   - Session persisted in browser

2. **Protected Routes**:
   - Middleware checks authentication
   - Redirects to sign-in if not authenticated
   - Automatic token refresh

3. **Session Management**:
   - Supabase handles session persistence
   - Automatic token refresh
   - Sign-out functionality

### 3.3 Middleware Configuration

```typescript
// src/middleware.ts
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

  // Protect API routes
  if (req.nextUrl.pathname.startsWith('/api') && !session) {
    return new NextResponse('Unauthorized', { status: 401 })
  }

  return res
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*'],
}
```

---

## 4. Database Schema Design

### 4.1 Supabase SQL Schema

```sql
-- Enable necessary extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create custom types
CREATE TYPE document_status AS ENUM ('draft', 'published', 'archived');
CREATE TYPE document_type AS ENUM ('letter', 'report', 'proposal', 'contract', 'memo', 'other');

-- Documents table
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  title VARCHAR(255) NOT NULL,
  content TEXT,
  content_html TEXT,
  type document_type DEFAULT 'other',
  status document_status DEFAULT 'draft',
  word_count INTEGER DEFAULT 0,
  is_template BOOLEAN DEFAULT FALSE,
  template_category VARCHAR(100),
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  last_edited_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Document versions for history
CREATE TABLE document_versions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
  version_number INTEGER NOT NULL,
  title VARCHAR(255) NOT NULL,
  content TEXT,
  content_html TEXT,
  created_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(document_id, version_number)
);

-- AI interactions log
CREATE TABLE ai_interactions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  document_id UUID REFERENCES documents(id) ON DELETE CASCADE,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  prompt TEXT NOT NULL,
  response TEXT,
  model VARCHAR(50) NOT NULL,
  tokens_used INTEGER,
  interaction_type VARCHAR(50), -- 'generate', 'improve', 'suggest'
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Document templates
CREATE TABLE document_templates (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  category VARCHAR(100) NOT NULL,
  content TEXT NOT NULL,
  content_html TEXT,
  variables JSONB DEFAULT '[]', -- Array of variable definitions
  is_public BOOLEAN DEFAULT FALSE,
  usage_count INTEGER DEFAULT 0,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- User preferences
CREATE TABLE user_preferences (
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
  default_document_type document_type DEFAULT 'other',
  auto_save BOOLEAN DEFAULT TRUE,
  ai_model VARCHAR(50) DEFAULT 'gpt-3.5-turbo',
  theme VARCHAR(20) DEFAULT 'light',
  editor_settings JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Row Level Security (RLS) Policies
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE document_versions ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_interactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE document_templates ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_preferences ENABLE ROW LEVEL SECURITY;

-- RLS Policies for documents
CREATE POLICY "Users can view own documents" ON documents
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can create own documents" ON documents
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own documents" ON documents
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own documents" ON documents
  FOR DELETE USING (auth.uid() = user_id);

-- Similar policies for other tables...
CREATE POLICY "Users can view own versions" ON document_versions
  FOR SELECT USING (EXISTS (
    SELECT 1 FROM documents WHERE documents.id = document_versions.document_id 
    AND documents.user_id = auth.uid()
  ));

-- Indexes for performance
CREATE INDEX idx_documents_user_id ON documents(user_id);
CREATE INDEX idx_documents_status ON documents(status);
CREATE INDEX idx_documents_created_at ON documents(created_at DESC);
CREATE INDEX idx_ai_interactions_user_id ON ai_interactions(user_id);
CREATE INDEX idx_ai_interactions_document_id ON ai_interactions(document_id);

-- Function to update updated_at timestamp
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Triggers for updated_at
CREATE TRIGGER update_documents_updated_at BEFORE UPDATE ON documents
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_user_preferences_updated_at BEFORE UPDATE ON user_preferences
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 4.2 Database Relationships

- **User → Documents**: One-to-many (user's documents)
- **Document → Versions**: One-to-many (document history)
- **Document → AI Interactions**: One-to-many (AI usage tracking)
- **User → Templates**: One-to-many (custom templates)
- **User → Preferences**: One-to-one (user settings)

---

## 5. AI Document Creation System

### 5.1 AI Service Integration

```typescript
// src/lib/ai/openai.ts
import OpenAI from 'openai'

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
})

export interface AIGenerationRequest {
  prompt: string
  type: 'generate' | 'improve' | 'suggest' | 'expand'
  documentType?: string
  context?: string
  tone?: string
  length?: 'short' | 'medium' | 'long'
}

export interface AIGenerationResponse {
  content: string
  wordCount: number
  tokensUsed: number
  suggestions?: string[]
}

export class AIService {
  private static async callOpenAI(messages: any[], model = 'gpt-3.5-turbo') {
    try {
      const response = await openai.chat.completions.create({
        model,
        messages,
        max_tokens: 2000,
        temperature: 0.7,
      })
      return response
    } catch (error) {
      console.error('OpenAI API Error:', error)
      throw new Error('AI service temporarily unavailable')
    }
  }

  static async generateContent(request: AIGenerationRequest): Promise<AIGenerationResponse> {
    const systemPrompt = this.buildSystemPrompt(request)
    
    const messages = [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: request.prompt }
    ]

    const response = await this.callOpenAI(messages)
    const content = response.choices[0]?.message?.content || ''

    return {
      content,
      wordCount: content.split(/\s+/).length,
      tokensUsed: response.usage?.total_tokens || 0,
    }
  }

  static async improveContent(content: string, instructions: string): Promise<AIGenerationResponse> {
    const prompt = `Improve the following text based on these instructions: ${instructions}\n\nOriginal text:\n${content}`
    
    const messages = [
      { 
        role: 'system', 
        content: 'You are a professional writing assistant. Improve the given text while maintaining the original meaning and tone.' 
      },
      { role: 'user', content: prompt }
    ]

    const response = await this.callOpenAI(messages)
    const improvedContent = response.choices[0]?.message?.content || ''

    return {
      content: improvedContent,
      wordCount: improvedContent.split(/\s+/).length,
      tokensUsed: response.usage?.total_tokens || 0,
    }
  }

  static async suggestImprovements(content: string): Promise<string[]> {
    const prompt = `Analyze the following text and suggest specific improvements:\n\n${content}`
    
    const messages = [
      { 
        role: 'system', 
        content: 'Provide 3-5 specific, actionable suggestions to improve this text. Focus on clarity, style, and effectiveness.' 
      },
      { role: 'user', content: prompt }
    ]

    const response = await this.callOpenAI(messages)
    const suggestionsText = response.choices[0]?.message?.content || ''
    
    // Parse suggestions (split by line breaks or bullet points)
    const suggestions = suggestionsText
      .split('\n')
      .filter(line => line.trim().length > 0)
      .slice(0, 5)

    return suggestions
  }

  private static buildSystemPrompt(request: AIGenerationRequest): string {
    let prompt = 'You are a professional document writing assistant. '
    
    if (request.documentType) {
      prompt += `Create a ${request.documentType}. `
    }
    
    if (request.tone) {
      prompt += `Use a ${request.tone} tone. `
    }
    
    if (request.length) {
      prompt += `Keep the response ${request.length}. `
    }
    
    prompt += 'Create well-structured, professional content that is clear and engaging.'
    
    return prompt
  }
}
```

### 5.2 Document Processing System

```typescript
// src/lib/document/processor.ts
import { Document, Packer, Paragraph, TextRun, HeadingLevel, AlignmentType } from 'docx'
import jsPDF from 'jspdf'

export interface DocumentProcessorOptions {
  format: 'html' | 'docx' | 'pdf'
  includeMetadata: boolean
  customStyles?: Record<string, any>
}

export class DocumentProcessor {
  static async exportDocument(
    content: string, 
    title: string, 
    options: DocumentProcessorOptions
  ): Promise<Buffer | string> {
    switch (options.format) {
      case 'html':
        return this.exportAsHTML(content, title, options)
      case 'docx':
        return await this.exportAsDOCX(content, title, options)
      case 'pdf':
        return await this.exportAsPDF(content, title, options)
      default:
        throw new Error(`Unsupported format: ${options.format}`)
    }
  }

  private static exportAsHTML(content: string, title: string, options: DocumentProcessorOptions): string {
    const metadata = options.includeMetadata ? this.generateMetadata(title) : ''
    return `
      <!DOCTYPE html>
      <html lang="en">
      <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>${title}</title>
        ${metadata}
        <style>
          body { 
            font-family: 'Times New Roman', serif; 
            line-height: 1.6; 
            max-width: 800px; 
            margin: 0 auto; 
            padding: 20px;
          }
          h1, h2, h3 { 
            color: #2c3e50; 
            margin-top: 30px;
          }
          p { 
            margin-bottom: 16px; 
            text-align: justify;
          }
        </style>
      </head>
      <body>
        <h1>${title}</h1>
        ${content}
      </body>
      </html>
    `
  }

  private static async exportAsDOCX(content: string, title: string, options: DocumentProcessorOptions): Promise<Buffer> {
    const doc = new Document({
      sections: [{
        properties: {},
        children: [
          new Paragraph({
            children: [
              new TextRun({
                text: title,
                bold: true,
                size: 32,
              }),
            ],
            heading: HeadingLevel.TITLE,
            alignment: AlignmentType.CENTER,
          }),
          ...this.parseContentToParagraphs(content),
        ],
      }],
    })

    const buffer = await Packer.toBuffer(doc)
    return buffer
  }

  private static async exportAsPDF(content: string, title: string, options: DocumentProcessorOptions): Promise<string> {
    const pdf = new jsPDF()
    
    // Add title
    pdf.setFontSize(20)
    pdf.text(title, 20, 30)
    
    // Add content
    pdf.setFontSize(12)
    const lines = pdf.splitTextToSize(content, 170)
    pdf.text(lines, 20, 50)
    
    return pdf.output('datauristring')
  }

  private static parseContentToParagraphs(content: string): Paragraph[] {
    // Simple HTML to DOCX conversion
    const sections = content.split('\n\n')
    return sections.map(section => {
      const trimmed = section.trim()
      if (trimmed) {
        return new Paragraph({
          children: [
            new TextRun({
              text: trimmed,
              size: 24,
            }),
          ],
        })
      }
      return new Paragraph({})
    })
  }

  private static generateMetadata(title: string): string {
    return `
      <meta name="author" content="AI Document Creator">
      <meta name="description" content="Generated document: ${title}">
      <meta name="generator" content="AI Document SaaS">
    `
  }
}
```

### 5.3 Document Editor Integration

```typescript
// src/components/document/document-editor.tsx
'use client'

import { useState, useEffect } from 'react'
import { EditorContent, useEditor } from '@tiptap/react'
import StarterKit from '@tiptap/starter-kit'
import { supabase } from '@/lib/supabase/client'
import { AIService } from '@/lib/ai/openai'
import { Button } from '@/components/ui/button'
import { AISidebar } from './ai-assistant'
import { DocumentToolbar } from './toolbar'

interface DocumentEditorProps {
  documentId: string
  initialContent?: string
  title: string
}

export function DocumentEditor({ documentId, initialContent = '', title }: DocumentEditorProps) {
  const [isSaving, setIsSaving] = useState(false)
  const [wordCount, setWordCount] = useState(0)
  const [isAIOpen, setIsAIOpen] = useState(false)

  const editor = useEditor({
    extensions: [StarterKit],
    content: initialContent,
    onUpdate: ({ editor }) => {
      const text = editor.getText()
      setWordCount(text.split(/\s+/).filter(word => word.length > 0).length)
    },
  })

  const saveDocument = async (content: string) => {
    setIsSaving(true)
    try {
      const { error } = await supabase
        .from('documents')
        .update({
          content: content,
          content_html: editor?.getHTML() || '',
          word_count: wordCount,
          last_edited_at: new Date().toISOString(),
        })
        .eq('id', documentId)

      if (error) throw error
    } catch (error) {
      console.error('Error saving document:', error)
    } finally {
      setIsSaving(false)
    }
  }

  const handleAIRequest = async (request: any) => {
    try {
      const response = await fetch('/api/ai/generate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          ...request,
          documentId,
          currentContent: editor?.getText(),
        }),
      })

      if (response.ok) {
        const result = await response.json()
        // Insert AI response at current cursor position
        editor?.commands.insertContent(result.content)
        await saveDocument(editor?.getText() || '')
      }
    } catch (error) {
      console.error('AI request failed:', error)
    }
  }

  return (
    <div className="flex h-screen">
      <div className="flex-1 flex flex-col">
        <DocumentToolbar
          onSave={() => saveDocument(editor?.getText() || '')}
          onAI={() => setIsAIOpen(!isAIOpen)}
          isSaving={isSaving}
          wordCount={wordCount}
          title={title}
        />
        
        <div className="flex-1 p-6">
          <EditorContent 
            editor={editor} 
            className="prose max-w-none min-h-screen"
          />
        </div>
      </div>

      {isAIOpen && (
        <AISidebar
          onAIRequest={handleAIRequest}
          onClose={() => setIsAIOpen(false)}
        />
      )}
    </div>
  )
}
```

---

## 6. API Routes & Middleware Structure

### 6.1 Document API Routes

```typescript
// src/app/api/documents/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase/server'

export async function GET(request: NextRequest) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const { data, error } = await supabase
    .from('documents')
    .select('*')
    .eq('user_id', user.id)
    .order('updated_at', { ascending: false })

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  return NextResponse.json({ documents: data })
}

export async function POST(request: NextRequest) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const body = await request.json()
  const { title, content = '', type = 'other' } = body

  const { data, error } = await supabase
    .from('documents')
    .insert({
      user_id: user.id,
      title,
      content,
      content_html: '',
      type,
      status: 'draft',
    })
    .select()
    .single()

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  return NextResponse.json({ document: data }, { status: 201 })
}

// src/app/api/documents/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase/server'

export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const { data, error } = await supabase
    .from('documents')
    .select('*')
    .eq('id', params.id)
    .eq('user_id', user.id)
    .single()

  if (error) {
    return NextResponse.json({ error: error.message }, { status: error.code === 'PGRST116' ? 404 : 500 })
  }

  return NextResponse.json({ document: data })
}

export async function PUT(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const body = await request.json()
  const { title, content, type, status } = body

  // Create version before update
  await supabase
    .from('document_versions')
    .insert({
      document_id: params.id,
      title: body.originalTitle,
      content: body.originalContent,
      content_html: body.originalContentHTML,
      created_by: user.id,
    })

  const { data, error } = await supabase
    .from('documents')
    .update({
      title,
      content,
      content_html: body.contentHTML,
      type,
      status,
      last_edited_at: new Date().toISOString(),
    })
    .eq('id', params.id)
    .eq('user_id', user.id)
    .select()
    .single()

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  return NextResponse.json({ document: data })
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const { error } = await supabase
    .from('documents')
    .delete()
    .eq('id', params.id)
    .eq('user_id', user.id)

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  return NextResponse.json({ success: true })
}
```

### 6.2 AI API Routes

```typescript
// src/app/api/ai/generate/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { AIService } from '@/lib/ai/openai'
import { createServerSupabaseClient } from '@/lib/supabase/server'

export async function POST(request: NextRequest) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const body = await request.json()
  const { prompt, type, documentType, context, tone, length, documentId } = body

  try {
    let result
    switch (type) {
      case 'generate':
        result = await AIService.generateContent({ prompt, type, documentType, tone, length })
        break
      case 'improve':
        result = await AIService.improveContent(context || '', prompt)
        break
      case 'suggest':
        const suggestions = await AIService.suggestImprovements(context || '')
        return NextResponse.json({ suggestions })
      default:
        return NextResponse.json({ error: 'Invalid AI request type' }, { status: 400 })
    }

    // Log AI interaction
    await supabase
      .from('ai_interactions')
      .insert({
        document_id: documentId,
        user_id: user.id,
        prompt,
        response: result.content,
        model: 'gpt-3.5-turbo',
        tokens_used: result.tokensUsed,
        interaction_type: type,
      })

    return NextResponse.json(result)
  } catch (error) {
    console.error('AI generation error:', error)
    return NextResponse.json({ error: 'AI service unavailable' }, { status: 500 })
  }
}
```

---

## 7. Security Considerations

### 7.1 Supabase Security Policies

```sql
-- RLS is enabled on all tables
-- Users can only access their own data
-- API keys and sensitive data stored securely

-- Additional security for document sharing
CREATE POLICY "Users can share documents with public link" ON documents
  FOR SELECT USING (
    user_id = auth.uid() OR 
    EXISTS (
      SELECT 1 FROM document_shares 
      WHERE document_shares.document_id = documents.id 
      AND document_shares.is_public = true
    )
  );

-- Audit log for all document operations
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id),
  document_id UUID REFERENCES documents(id),
  action VARCHAR(50) NOT NULL,
  metadata JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### 7.2 Environment Security

- **Supabase Keys**: Anon key is public, service role key is server-only
- **API Keys**: OpenAI API key stored in Vercel environment variables
- **HTTPS Only**: All connections encrypted
- **Rate Limiting**: Prevent AI API abuse

### 7.3 Data Protection

- **Row Level Security**: Database-level access control
- **API Validation**: Input sanitization and validation
- **Session Security**: Secure session handling via Supabase
- **Content Security**: XSS protection in editor

---

## 8. Development & Deployment Setup

### 8.1 Environment Configuration

```bash
# .env.local
# Supabase
NEXT_PUBLIC_SUPABASE_URL="your-supabase-project-url"
NEXT_PUBLIC_SUPABASE_ANON_KEY="your-supabase-anon-key"
SUPABASE_SERVICE_ROLE_KEY="your-supabase-service-role-key"

# AI Services
OPENAI_API_KEY="your-openai-api-key"
ANTHROPIC_API_KEY="your-anthropic-api-key"

# Application
NEXT_PUBLIC_APP_URL="http://localhost:3000"
NEXTAUTH_SECRET="your-nextauth-secret"

# Optional: Analytics
NEXT_PUBLIC_ANALYTICS_ID="your-analytics-id"
```

### 8.2 Supabase Setup

```bash
# Install Supabase CLI
npm install -g supabase

# Login to Supabase
supabase login

# Initialize project
supabase init

# Start local development
supabase start

# Link to remote project
supabase link --project-ref your-project-ref

# Push database changes
supabase db push

# Generate TypeScript types
supabase gen types typescript --project-id your-project-ref > src/lib/supabase/types.ts
```

### 8.3 Package.json Scripts

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "db:generate": "supabase gen types typescript --project-id $SUPABASE_PROJECT_ID > src/lib/supabase/types.ts",
    "db:push": "supabase db push",
    "db:reset": "supabase db reset",
    "db:studio": "supabase studio",
    "test": "jest",
    "test:watch": "jest --watch"
  }
}
```

### 8.4 Vercel Deployment

```json
// vercel.json
{
  "framework": "nextjs",
  "buildCommand": "npm run build",
  "installCommand": "npm ci",
  "outputDirectory": ".next",
  "env": {
    "NEXT_PUBLIC_SUPABASE_URL": "@supabase-url",
    "NEXT_PUBLIC_SUPABASE_ANON_KEY": "@supabase-anon-key",
    "OPENAI_API_KEY": "@openai-api-key",
    "NEXT_PUBLIC_APP_URL": "@app-url"
  },
  "functions": {
    "src/app/api/ai/**/*.ts": {
      "maxDuration": 30
    }
  }
}
```

---

## 9. Configuration Templates

### 9.1 Next.js Configuration

```javascript
// next.config.js
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
}

module.exports = nextConfig
```

### 9.2 TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts"
  ],
  "exclude": ["node_modules"]
}
```

---

## Implementation Priority

### Phase 1: Core Setup (Week 1)
1. Project initialization with Next.js + Supabase
2. Authentication flow implementation
3. Basic document CRUD operations
4. Simple document editor

### Phase 2: AI Integration (Week 2)
1. OpenAI API integration
2. AI-powered content generation
3. Document improvement suggestions
4. AI interaction logging

### Phase 3: Advanced Features (Week 3)
1. Document export (HTML, DOCX, PDF)
2. Document versioning
3. Templates system
4. Advanced editor features

### Phase 4: Polish & Deploy (Week 4)
1. UI/UX improvements
2. Performance optimization
3. Security audit
4. Production deployment

This architecture plan provides a solid foundation for your AI document creation SaaS, with Supabase handling authentication, data storage, and real-time features while maintaining security and scalability for future growth.