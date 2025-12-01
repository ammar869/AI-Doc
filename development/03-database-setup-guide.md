# Database Schema Setup Guide

## Overview
This guide covers setting up the database schema for the AI Document SaaS application, including tables, relationships, and security policies.

## Step 1: Database Selection

For this project, we'll use SQLite for development and PostgreSQL for production. Better Auth can work with both, but PostgreSQL is recommended for production due to better concurrency support.

### Development Setup (SQLite)
```typescript
// In src/lib/auth.ts
export const auth = betterAuth({
  database: {
    type: "sqlite",
    databaseUrl: "./better-auth.db"
  },
  // ... other config
})
```

### Production Setup (PostgreSQL)
```typescript
// In src/lib/auth.ts
export const auth = betterAuth({
  database: {
    type: "postgres",
    url: process.env.DATABASE_URL!
  },
  // ... other config
})
```

## Step 2: Initial Database Setup

### Generate Database Schema

Run the Better Auth CLI to generate the initial schema:

```bash
# Generate the schema
npx @better-auth/cli generate

# Apply migrations (for built-in adapters)
npx @better-auth/cli migrate
```

This will create the necessary tables for user authentication.

### Custom Database Schema

Create additional tables for your application:

Create `supabase/migrations/001_initial_schema.sql`:

```sql
-- Enable necessary extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create custom types
CREATE TYPE document_status AS ENUM ('draft', 'published', 'archived');
CREATE TYPE document_type AS ENUM ('letter', 'report', 'proposal', 'contract', 'memo', 'other');

-- Documents table
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
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
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
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
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
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

-- Create indexes for performance
CREATE INDEX idx_documents_user_id ON documents(user_id);
CREATE INDEX idx_documents_status ON documents(status);
CREATE INDEX idx_documents_created_at ON documents(created_at DESC);
CREATE INDEX idx_ai_interactions_user_id ON ai_interactions(user_id);
CREATE INDEX idx_ai_interactions_document_id ON ai_interactions(document_id);
CREATE INDEX idx_document_versions_document_id ON document_versions(document_id);
CREATE INDEX idx_document_templates_user_id ON document_templates(user_id);

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

CREATE TRIGGER update_document_templates_updated_at BEFORE UPDATE ON document_templates
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

## Step 3: Row Level Security (RLS) Policies

Since we're using Better Auth, we need to set up our own RLS policies. Better Auth doesn't automatically create them like Supabase.

Create `supabase/migrations/002_rls_policies.sql`:

```sql
-- Enable RLS on all custom tables
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE document_versions ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_interactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE document_templates ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_preferences ENABLE ROW LEVEL SECURITY;

-- RLS Policies for documents
CREATE POLICY "Users can view own documents" ON documents
  FOR SELECT USING (
    auth.uid() = user_id OR 
    is_template = true
  );

CREATE POLICY "Users can create own documents" ON documents
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own documents" ON documents
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own documents" ON documents
  FOR DELETE USING (auth.uid() = user_id);

-- RLS Policies for document versions
CREATE POLICY "Users can view own document versions" ON document_versions
  FOR SELECT USING (EXISTS (
    SELECT 1 FROM documents WHERE documents.id = document_versions.document_id 
    AND documents.user_id = auth.uid()
  ));

CREATE POLICY "Users can create document versions" ON document_versions
  FOR INSERT WITH CHECK (EXISTS (
    SELECT 1 FROM documents WHERE documents.id = document_versions.document_id 
    AND documents.user_id = auth.uid()
  ));

-- RLS Policies for AI interactions
CREATE POLICY "Users can view own AI interactions" ON ai_interactions
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can create own AI interactions" ON ai_interactions
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own AI interactions" ON ai_interactions
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own AI interactions" ON ai_interactions
  FOR DELETE USING (auth.uid() = user_id);

-- RLS Policies for document templates
CREATE POLICY "Users can view own and public templates" ON document_templates
  FOR SELECT USING (
    auth.uid() = user_id OR 
    is_public = true
  );

CREATE POLICY "Users can create own templates" ON document_templates
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own templates" ON document_templates
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own templates" ON document_templates
  FOR DELETE USING (auth.uid() = user_id);

-- RLS Policies for user preferences
CREATE POLICY "Users can view own preferences" ON user_preferences
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can create own preferences" ON user_preferences
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own preferences" ON user_preferences
  FOR UPDATE USING (auth.uid() = user_id);
```

## Step 4: Database TypeScript Types

Generate TypeScript types for your database:

```bash
# For PostgreSQL
npx supabase gen types typescript --project-id your-project-id > src/lib/database/types.ts

# For SQLite, create manual types
```

Create `src/lib/database/types.ts`:

```typescript
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
          type: DocumentType
          status: DocumentStatus
          word_count: number
          is_template: boolean
          template_category: string | null
          metadata: Record<string, any>
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
          type?: DocumentType
          status?: DocumentStatus
          word_count?: number
          is_template?: boolean
          template_category?: string | null
          metadata?: Record<string, any>
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
          type?: DocumentType
          status?: DocumentStatus
          word_count?: number
          is_template?: boolean
          template_category?: string | null
          metadata?: Record<string, any>
          created_at?: string
          updated_at?: string
          last_edited_at?: string
        }
      }
      document_versions: {
        Row: {
          id: string
          document_id: string
          version_number: number
          title: string
          content: string | null
          content_html: string | null
          created_by: string | null
          created_at: string
        }
        Insert: {
          id?: string
          document_id: string
          version_number: number
          title: string
          content?: string | null
          content_html?: string | null
          created_by?: string | null
          created_at?: string
        }
        Update: {
          id?: string
          document_id?: string
          version_number?: number
          title?: string
          content?: string | null
          content_html?: string | null
          created_by?: string | null
          created_at?: string
        }
      }
      ai_interactions: {
        Row: {
          id: string
          document_id: string | null
          user_id: string
          prompt: string
          response: string | null
          model: string
          tokens_used: number | null
          interaction_type: string | null
          created_at: string
        }
        Insert: {
          id?: string
          document_id?: string | null
          user_id: string
          prompt: string
          response?: string | null
          model: string
          tokens_used?: number | null
          interaction_type?: string | null
          created_at?: string
        }
        Update: {
          id?: string
          document_id?: string | null
          user_id?: string
          prompt?: string
          response?: string | null
          model?: string
          tokens_used?: number | null
          interaction_type?: string | null
          created_at?: string
        }
      }
      document_templates: {
        Row: {
          id: string
          user_id: string | null
          name: string
          description: string | null
          category: string
          content: string
          content_html: string | null
          variables: any[]
          is_public: boolean
          usage_count: number
          created_at: string
          updated_at: string
        }
        Insert: {
          id?: string
          user_id?: string | null
          name: string
          description?: string | null
          category: string
          content: string
          content_html?: string | null
          variables?: any[]
          is_public?: boolean
          usage_count?: number
          created_at?: string
          updated_at?: string
        }
        Update: {
          id?: string
          user_id?: string | null
          name?: string
          description?: string | null
          category?: string
          content?: string
          content_html?: string | null
          variables?: any[]
          is_public?: boolean
          usage_count?: number
          created_at?: string
          updated_at?: string
        }
      }
      user_preferences: {
        Row: {
          user_id: string
          default_document_type: DocumentType
          auto_save: boolean
          ai_model: string
          theme: string
          editor_settings: Record<string, any>
          created_at: string
          updated_at: string
        }
        Insert: {
          user_id: string
          default_document_type?: DocumentType
          auto_save?: boolean
          ai_model?: string
          theme?: string
          editor_settings?: Record<string, any>
          created_at?: string
          updated_at?: string
        }
        Update: {
          user_id?: string
          default_document_type?: DocumentType
          auto_save?: boolean
          ai_model?: string
          theme?: string
          editor_settings?: Record<string, any>
          created_at?: string
          updated_at?: string
        }
      }
    }
  }
}

export type DocumentType = 'letter' | 'report' | 'proposal' | 'contract' | 'memo' | 'other'
export type DocumentStatus = 'draft' | 'published' | 'archived'
```

## Step 5: Database Client Setup

Create `src/lib/database/client.ts`:

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

// For database operations (not auth)
export const db = createClient<Database>(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!, // Only on server-side
  {
    auth: {
      autoRefreshToken: false,
      persistSession: false
    }
  }
)
```

## Step 6: Database Utilities

Create `src/lib/database/utils.ts`:

```typescript
import { auth } from '@/lib/auth'
import { db } from './client'
import { Database, DocumentType, DocumentStatus } from './types'

export type Document = Database['public']['Tables']['documents']['Row']
export type DocumentInsert = Database['public']['Tables']['documents']['Insert']
export type DocumentUpdate = Database['public']['Tables']['documents']['Update']

export type DocumentVersion = Database['public']['Tables']['document_versions']['Row']
export type AIInteraction = Database['public']['Tables']['ai_interactions']['Row']
export type DocumentTemplate = Database['public']['Tables']['document_templates']['Row']
export type UserPreferences = Database['public']['Tables']['user_preferences']['Row']

// Get current user from Better Auth
export async function getCurrentUser() {
  const session = await auth()
  return session?.user
}

// Database query helper with auth
export async function queryWithAuth<T>(
  query: any
): Promise<T> {
  const session = await auth()
  
  if (!session) {
    throw new Error('Not authenticated')
  }
  
  return query.eq('user_id', session.user.id)
}

// Create new document
export async function createDocument(data: Omit<DocumentInsert, 'user_id'>) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Not authenticated')
  
  const { data: document, error } = await db
    .from('documents')
    .insert({
      ...data,
      user_id: user.id
    })
    .select()
    .single()
  
  if (error) throw error
  return document
}

// Get user documents
export async function getUserDocuments(filters?: {
  status?: DocumentStatus
  type?: DocumentType
  limit?: number
  offset?: number
}) {
  let query = db
    .from('documents')
    .select('*')
    .order('updated_at', { ascending: false })
  
  if (filters?.status) {
    query = query.eq('status', filters.status)
  }
  
  if (filters?.type) {
    query = query.eq('type', filters.type)
  }
  
  if (filters?.limit) {
    query = query.limit(filters.limit)
  }
  
  if (filters?.offset) {
    query = query.range(filters.offset, filters.offset + (filters.limit || 10) - 1)
  }
  
  const { data, error } = await query
  if (error) throw error
  return data
}

// Get document by ID
export async function getDocument(id: string) {
  const { data, error } = await db
    .from('documents')
    .select('*')
    .eq('id', id)
    .single()
  
  if (error) throw error
  return data
}

// Update document
export async function updateDocument(id: string, data: DocumentUpdate) {
  const { data: document, error } = await db
    .from('documents')
    .update(data)
    .eq('id', id)
    .select()
    .single()
  
  if (error) throw error
  return document
}

// Delete document
export async function deleteDocument(id: string) {
  const { error } = await db
    .from('documents')
    .delete()
    .eq('id', id)
  
  if (error) throw error
}

// Create document version
export async function createDocumentVersion(documentId: string, data: {
  title: string
  content: string | null
  content_html: string | null
}) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Not authenticated')
  
  // Get the latest version number
  const { data: versions } = await db
    .from('document_versions')
    .select('version_number')
    .eq('document_id', documentId)
    .order('version_number', { ascending: false })
    .limit(1)
  
  const nextVersion = (versions?.[0]?.version_number || 0) + 1
  
  const { data: version, error } = await db
    .from('document_versions')
    .insert({
      document_id: documentId,
      version_number: nextVersion,
      title: data.title,
      content: data.content,
      content_html: data.content_html,
      created_by: user.id
    })
    .select()
    .single()
  
  if (error) throw error
  return version
}

// Log AI interaction
export async function logAIInteraction(data: {
  documentId?: string
  prompt: string
  response?: string
  model: string
  tokensUsed?: number
  interactionType: string
}) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Not authenticated')
  
  const { data: interaction, error } = await db
    .from('ai_interactions')
    .insert({
      document_id: data.documentId || null,
      user_id: user.id,
      prompt: data.prompt,
      response: data.response || null,
      model: data.model,
      tokens_used: data.tokensUsed || null,
      interaction_type: data.interactionType
    })
    .select()
    .single()
  
  if (error) throw error
  return interaction
}

// Get or create user preferences
export async function getUserPreferences() {
  const user = await getCurrentUser()
  if (!user) throw new Error('Not authenticated')
  
  const { data, error } = await db
    .from('user_preferences')
    .select('*')
    .eq('user_id', user.id)
    .single()
  
  if (error && error.code !== 'PGRST116') {
    throw error
  }
  
  // If preferences don't exist, create default ones
  if (!data) {
    const { data: newPreferences, error: insertError } = await db
      .from('user_preferences')
      .insert({
        user_id: user.id
      })
      .select()
      .single()
    
    if (insertError) throw insertError
    return newPreferences
  }
  
  return data
}

// Update user preferences
export async function updateUserPreferences(data: Partial<UserPreferences>) {
  const user = await getCurrentUser()
  if (!user) throw new Error('Not authenticated')
  
  const { data: preferences, error } = await db
    .from('user_preferences')
    .update(data)
    .eq('user_id', user.id)
    .select()
    .single()
  
  if (error) throw error
  return preferences
}
```

## Step 7: Apply Migrations

For PostgreSQL (Supabase):
```bash
# Apply migrations
supabase db push

# Or for local development
supabase start
supabase db reset
```

For SQLite:
```bash
# SQLite schema is created automatically
# Just ensure the auth database is initialized
```

## Step 8: Seed Data (Optional)

Create `supabase/seed.sql`:

```sql
-- Insert some sample templates
INSERT INTO document_templates (name, description, category, content, is_public) VALUES
('Business Letter Template', 'Standard business letter format', 'Letter', 'Dear [Recipient Name],

I am writing to [Purpose]. Please find details below:

[Content]

Thank you for your time and consideration.

Best regards,
[Your Name]', true),
('Project Proposal Template', 'Standard project proposal format', 'Proposal', 'Project Title: [Project Name]

Executive Summary:
[Summary]

Objectives:
[Objectives]

Timeline:
[Timeline]

Budget:
[Budget]

Conclusion:
[Conclusion]', true);

-- Create a sample document for testing
INSERT INTO documents (title, content, type, user_id) VALUES
('My First Document', 'This is my first document created with the AI Document Creator.', 'other', (SELECT id FROM auth.users LIMIT 1));
```

## Testing Database Setup

### 1. Test Database Connection
```typescript
// Test in a component or API route
import { getUserDocuments } from '@/lib/database/utils'

export async function testDatabase() {
  try {
    const documents = await getUserDocuments()
    console.log('Database connection successful:', documents)
    return true
  } catch (error) {
    console.error('Database connection failed:', error)
    return false
  }
}
```

### 2. Test RLS Policies
```typescript
// Test that users can only see their own data
import { createDocument } from '@/lib/database/utils'

export async function testRLS() {
  try {
    const document = await createDocument({
      title: 'Test Document',
      content: 'Test content',
      type: 'other'
    })
    console.log('RLS working:', document)
    return true
  } catch (error) {
    console.error('RLS issue:', error)
    return false
  }
}
```

## Production Considerations

### 1. PostgreSQL for Production
- Use PostgreSQL for better concurrency and reliability
- Set up connection pooling
- Configure proper indexes for performance

### 2. Backup Strategy
- Set up regular database backups
- Test restore procedures
- Monitor database size and performance

### 3. Security
- Ensure all API keys are secure
- Use environment variables for database connections
- Regular security updates

### 4. Performance Optimization
- Monitor query performance
- Add appropriate indexes
- Consider read replicas for heavy read workloads

## Common Issues and Solutions

### 1. Migration Failures
- Check for syntax errors in SQL
- Ensure proper extension installation
- Verify user permissions

### 2. RLS Policy Issues
- Test policies with authenticated users
- Check policy syntax
- Verify foreign key relationships

### 3. Type Generation Issues
- Regenerate types after schema changes
- Ensure database is accessible
- Check connection string validity

## Next Steps

Now that the database is set up, proceed to:
1. [Document CRUD Operations Guide](./04-document-crud-guide.md)
2. [AI Integration Guide](./05-ai-integration-guide.md)

This completes the database schema setup. You now have a secure, well-structured database ready for your AI Document SaaS application.