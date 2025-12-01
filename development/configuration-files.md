# Configuration Files & Examples

## Package.json Configuration

```json
{
  "name": "ai-document-saas",
  "version": "1.0.0",
  "private": true,
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
  },
  "dependencies": {
    "next": "14.1.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@supabase/supabase-js": "^2.39.0",
    "@supabase/auth-helpers-nextjs": "^0.8.7",
    "@tiptap/react": "^2.1.16",
    "@tiptap/starter-kit": "^2.1.16",
    "@tiptap/extension-placeholder": "^2.1.16",
    "@tiptap/extension-character-count": "^2.1.16",
    "openai": "^4.24.1",
    "docx": "^8.5.0",
    "jspdf": "^2.5.1",
    "html2canvas": "^1.4.1",
    "html-to-docx": "^1.8.0",
    "@radix-ui/react-dialog": "^1.0.5",
    "@radix-ui/react-dropdown-menu": "^2.0.6",
    "@radix-ui/react-label": "^2.0.2",
    "@radix-ui/react-select": "^2.0.0",
    "@radix-ui/react-toast": "^1.1.5",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.0.0",
    "tailwind-merge": "^2.2.0",
    "lucide-react": "^0.323.0",
    "zod": "^3.22.4",
    "react-hook-form": "^7.49.3",
    "@hookform/resolvers": "^3.3.4",
    "date-fns": "^3.2.0",
    "react-hot-toast": "^2.4.1",
    "framer-motion": "^10.18.0"
  },
  "devDependencies": {
    "@types/node": "^20.11.0",
    "@types/react": "^18.2.48",
    "@types/react-dom": "^18.2.18",
    "typescript": "^5.3.3",
    "tailwindcss": "^3.4.1",
    "postcss": "^8.4.33",
    "autoprefixer": "^10.4.17",
    "eslint": "^8.56.0",
    "eslint-config-next": "14.1.0",
    "@tailwindcss/typography": "^0.5.10",
    "prettier": "^3.2.4",
    "husky": "^8.0.3",
    "lint-staged": "^15.2.0",
    "jest": "^29.7.0",
    "@testing-library/react": "^14.2.0",
    "@testing-library/jest-dom": "^6.4.0"
  }
}
```

## Tailwind Configuration

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: ["class"],
  content: [
    './pages/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
    './app/**/*.{ts,tsx}',
    './src/**/*.{ts,tsx}',
  ],
  theme: {
    container: {
      center: true,
      padding: "2rem",
      screens: {
        "2xl": "1400px",
      },
    },
    extend: {
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        accent: {
          DEFAULT: "hsl(var(--accent))",
          foreground: "hsl(var(--accent-foreground))",
        },
        popover: {
          DEFAULT: "hsl(var(--popover))",
          foreground: "hsl(var(--popover-foreground))",
        },
        card: {
          DEFAULT: "hsl(var(--card))",
          foreground: "hsl(var(--card-foreground))",
        },
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
      keyframes: {
        "accordion-down": {
          from: { height: "0" },
          to: { height: "var(--radix-accordion-content-height)" },
        },
        "accordion-up": {
          from: { height: "var(--radix-accordion-content-height)" },
          to: { height: "0" },
        },
        "fade-in": {
          from: { opacity: "0", transform: "translateY(10px)" },
          to: { opacity: "1", transform: "translateY(0)" },
        },
        "slide-in": {
          from: { transform: "translateX(-100%)" },
          to: { transform: "translateX(0)" },
        },
      },
      animation: {
        "accordion-down": "accordion-down 0.2s ease-out",
        "accordion-up": "accordion-up 0.2s ease-out",
        "fade-in": "fade-in 0.5s ease-out",
        "slide-in": "slide-in 0.3s ease-out",
      },
    },
  },
  plugins: [
    require("@tailwindcss/typography"),
    require("tailwindcss-animate"),
  ],
}
```

## Vercel Configuration

```json
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
    },
    "src/app/api/documents/**/export/route.ts": {
      "maxDuration": 30
    }
  },
  "rewrites": [
    {
      "source": "/api/(.*)",
      "destination": "/api/$1"
    }
  ]
}
```

## Environment Variables Template

```bash
# .env.example
# Supabase Configuration
NEXT_PUBLIC_SUPABASE_URL=your-supabase-project-url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-supabase-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-supabase-service-role-key
SUPABASE_PROJECT_ID=your-project-id

# AI Services
OPENAI_API_KEY=your-openai-api-key
ANTHROPIC_API_KEY=your-anthropic-api-key

# Application Configuration
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXTAUTH_SECRET=your-nextauth-secret

# Optional: Analytics & Monitoring
NEXT_PUBLIC_ANALYTICS_ID=your-analytics-id
SENTRY_DSN=your-sentry-dsn

# Optional: Email Service (for notifications)
SMTP_HOST=your-smtp-host
SMTP_PORT=587
SMTP_USER=your-email@example.com
SMTP_PASSWORD=your-email-password

# Optional: File Upload Limits
MAX_FILE_SIZE=10485760
```

## Supabase SQL Setup

```sql
-- supabase/migrations/001_initial_schema.sql

-- Create document status enum
CREATE TYPE document_status AS ENUM ('draft', 'published', 'archived');
CREATE TYPE document_type AS ENUM ('letter', 'report', 'proposal', 'contract', 'memo', 'other');

-- Create documents table
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

-- Create indexes
CREATE INDEX idx_documents_user_id ON documents(user_id);
CREATE INDEX idx_documents_status ON documents(status);
CREATE INDEX idx_documents_created_at ON documents(created_at DESC);

-- Enable RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- RLS Policies
CREATE POLICY "Users can manage own documents" ON documents
  FOR ALL USING (auth.uid() = user_id);

-- Create audit logs table
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

-- Enable RLS on audit logs
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own audit logs" ON audit_logs
  FOR SELECT USING (auth.uid() = user_id);
```

## GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Vercel

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npm run type-check
      - run: npm run lint
      - run: npm run test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.ORG_ID }}
          vercel-project-id: ${{ secrets.PROJECT_ID }}
          working-directory: ./
          vercel-args: '--prod'
```

## ESLint Configuration

```json
{
  "extends": [
    "next/core-web-vitals",
    "@typescript-eslint/recommended"
  ],
  "parser": "@typescript-eslint/parser",
  "plugins": ["@typescript-eslint"],
  "rules": {
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/no-explicit-any": "warn",
    "prefer-const": "error",
    "no-var": "error"
  },
  "ignorePatterns": ["dist", ".next", "node_modules"]
}
```

## TypeScript Types Example

```typescript
// src/types/document.ts
export interface Document {
  id: string
  user_id: string
  title: string
  content: string
  content_html: string
  type: DocumentType
  status: DocumentStatus
  word_count: number
  is_template: boolean
  template_category?: string
  metadata: Record<string, any>
  created_at: string
  updated_at: string
  last_edited_at: string
}

export type DocumentType = 'letter' | 'report' | 'proposal' | 'contract' | 'memo' | 'other'
export type DocumentStatus = 'draft' | 'published' | 'archived'

export interface DocumentTemplate {
  id: string
  user_id: string
  name: string
  description?: string
  category: string
  content: string
  content_html: string
  variables: TemplateVariable[]
  is_public: boolean
  usage_count: number
  created_at: string
  updated_at: string
}

export interface TemplateVariable {
  name: string
  description: string
  type: 'text' | 'number' | 'date' | 'select'
  default?: string
  options?: string[]
}

export interface AIGenerationRequest {
  prompt: string
  type: 'generate' | 'improve' | 'suggest' | 'expand'
  documentType?: DocumentType
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
```

## Database Connection Example

```typescript
// src/lib/supabase/types.ts (Generated file example)
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
          type: 'letter' | 'report' | 'proposal' | 'contract' | 'memo' | 'other'
          status: 'draft' | 'published' | 'archived'
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
          type?: 'letter' | 'report' | 'proposal' | 'contract' | 'memo' | 'other'
          status?: 'draft' | 'published' | 'archived'
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
          type?: 'letter' | 'report' | 'proposal' | 'contract' | 'memo' | 'other'
          status?: 'draft' | 'published' | 'archived'
          word_count?: number
          is_template?: boolean
          template_category?: string | null
          metadata?: Record<string, any>
          created_at?: string
          updated_at?: string
          last_edited_at?: string
        }
      }
    }
  }
}
```

This comprehensive architecture plan and configuration files provide everything needed to implement your AI document creation SaaS with Supabase as the backend. The plan is optimized for single-user functionality with scaling capability, includes all necessary security measures, and provides a clear roadmap for implementation.