# Project Setup & Initialization Guide

## Overview
This guide will walk you through setting up the AI Document SaaS project from scratch, including all necessary dependencies and configurations.

## Prerequisites
Before starting, ensure you have:
- Node.js 18+ installed
- npm or yarn package manager
- Git for version control
- Code editor (VS Code recommended)

## Step 1: Create Next.js Project

```bash
# Create new Next.js project with TypeScript
npx create-next-app@latest ai-document-saas --typescript --tailwind --eslint --app

# Navigate to project directory
cd ai-document-saas

# Install additional dependencies
npm install @supabase/supabase-js @supabase/auth-helpers-nextjs
npm install @harbour-enterprises/superdoc
npm install openai
npm install jspdf docx
npm install lucide-react
```

## Step 2: Project Structure Setup

Create the following directory structure:

```
ai-document-saas/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   ├── (dashboard)/
│   │   └── api/
│   ├── components/
│   │   ├── ui/
│   │   ├── auth/
│   │   ├── document/
│   │   └── dashboard/
│   ├── lib/
│   │   ├── supabase/
│   │   ├── ai/
│   │   ├── document/
│   │   ├── utils/
│   │   └── constants/
│   ├── hooks/
│   └── types/
├── supabase/
│   └── migrations/
└── public/
```

Create directories using:

```bash
# Create all directories at once
mkdir -p src/app/\\\(auth\\\\\) src/app/\\\(dashboard\\\\\) src/app/api/{auth,documents,ai,templates}
mkdir -p src/components/{ui,auth,document,dashboard}
mkdir -p src/lib/{supabase,ai,document,utils,constants}
mkdir -p src/{hooks,types}
mkdir -p supabase/migrations
```

## Step 3: Environment Configuration

Create `.env.local` file:

```bash
# Supabase Configuration
NEXT_PUBLIC_SUPABASE_URL="your-supabase-project-url"
NEXT_PUBLIC_SUPABASE_ANON_KEY="your-supabase-anon-key"
SUPABASE_SERVICE_ROLE_KEY="your-supabase-service-role-key"

# AI Services
OPENAI_API_KEY="your-openai-api-key"

# Application
NEXT_PUBLIC_APP_URL="http://localhost:3000"

# Authentication
NEXTAUTH_SECRET="your-nextauth-secret"
```

## Step 4: Basic Configuration Files

### Tailwind CSS Configuration

Update `tailwind.config.js`:

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
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
      },
    },
  },
  plugins: [],
}
```

### TypeScript Configuration

Update `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "es6"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
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
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### Next.js Configuration

Create/Update `next.config.js`:

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
}

module.exports = nextConfig
```

## Step 5: Basic Global Styles

Update `src/app/globals.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96%;
    --secondary-foreground: 222.2 84% 4.9%;
    --muted: 210 40% 96%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96%;
    --accent-foreground: 222.2 84% 4.9%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 222.2 84% 4.9%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
    --primary: 210 40% 98%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 212.7 26.8% 83.9%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}

.prose {
  @apply max-w-none;
}

.prose h1,
.prose h2,
.prose h3,
.prose h4,
.prose h5,
.prose h6 {
  @apply text-foreground;
}

.prose p {
  @apply text-foreground;
}
```

## Step 6: Update Package.json Scripts

Add these scripts to your `package.json`:

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
    "db:studio": "supabase studio"
  }
}
```

## Step 7: Install and Test

```bash
# Install dependencies
npm install

# Test the setup
npm run dev
```

Your application should now be running on `http://localhost:3000`.

## Common Issues and Solutions

### 1. Module Resolution Issues
If you encounter module resolution errors, ensure your `baseUrl` and `paths` in `tsconfig.json` are correctly configured.

### 2. Supabase Connection Issues
- Verify your Supabase URL and keys are correct
- Ensure your Supabase project is active
- Check that the environment variables are loaded properly

### 3. TypeScript Errors
- Run `npm run type-check` to identify specific type issues
- Ensure all dependencies are properly installed
- Check that your `tsconfig.json` includes all necessary paths

## Next Steps

Now that your project is set up, proceed to:
1. [Authentication Implementation Guide](./02-authentication-guide.md)
2. [Database Schema Setup Guide](./03-database-setup-guide.md)

This completes the basic project setup. You now have a solid foundation with Next.js, TypeScript, Tailwind CSS, and the necessary dependencies to build your AI Document SaaS application.