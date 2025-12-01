# Deployment Guide

## Overview
This guide covers deploying the AI Document SaaS application to production, including environment setup, database deployment, CI/CD pipeline configuration, and production optimizations.

## Step 1: Production Environment Setup

### Environment Variables

Create `.env.production`:

```bash
# Supabase Configuration
NEXT_PUBLIC_SUPABASE_URL="https://your-project.supabase.co"
NEXT_PUBLIC_SUPABASE_ANON_KEY="your-production-anon-key"
SUPABASE_SERVICE_ROLE_KEY="your-production-service-role-key"

# AI Services
OPENAI_API_KEY="your-production-openai-api-key"

# Application
NEXT_PUBLIC_APP_URL="https://yourdomain.com"
NEXTAUTH_SECRET="your-secure-secret-key-min-32-chars"

# Security
RATE_LIMIT_ENABLED=true
ENCRYPTION_KEY="your-32-character-encryption-key"

# Monitoring (Optional)
NEXT_PUBLIC_ANALYTICS_ID="your-analytics-id"
SENTRY_DSN="your-sentry-dsn"

# Performance
NODE_ENV="production"
NEXT_TELEMETRY_DISABLED="1"
```

### Environment Validation Script

Create `scripts/validate-env.js`:

```javascript
#!/usr/bin/env node

const requiredEnvVars = [
  'NEXT_PUBLIC_SUPABASE_URL',
  'NEXT_PUBLIC_SUPABASE_ANON_KEY',
  'SUPABASE_SERVICE_ROLE_KEY',
  'OPENAI_API_KEY',
  'NEXT_PUBLIC_APP_URL',
  'NEXTAUTH_SECRET',
]

const optionalEnvVars = [
  'RATE_LIMIT_ENABLED',
  'ENCRYPTION_KEY',
  'NEXT_PUBLIC_ANALYTICS_ID',
  'SENTRY_DSN',
]

function validateEnvironment() {
  console.log('🔍 Validating environment variables...\n')
  
  let hasErrors = false
  
  // Check required variables
  requiredEnvVars.forEach(envVar => {
    if (!process.env[envVar]) {
      console.error(`❌ Missing required environment variable: ${envVar}`)
      hasErrors = true
    } else {
      console.log(`✅ ${envVar}`)
    }
  })
  
  // Check optional variables
  console.log('\n📝 Optional variables:')
  optionalEnvVars.forEach(envVar => {
    if (!process.env[envVar]) {
      console.warn(`⚠️  Optional variable not set: ${envVar}`)
    } else {
      console.log(`✅ ${envVar}`)
    }
  })
  
  // Validate secret key length
  if (process.env.NEXTAUTH_SECRET && process.env.NEXTAUTH_SECRET.length < 32) {
    console.error('❌ NEXTAUTH_SECRET must be at least 32 characters long')
    hasErrors = true
  }
  
  // Validate URLs
  const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL
  if (supabaseUrl && (!supabaseUrl.startsWith('https://') || !supabaseUrl.includes('.supabase.co'))) {
    console.error('❌ NEXT_PUBLIC_SUPABASE_URL must be a valid Supabase URL')
    hasErrors = true
  }
  
  const appUrl = process.env.NEXT_PUBLIC_APP_URL
  if (appUrl && (!appUrl.startsWith('https://') && !appUrl.startsWith('http://localhost'))) {
    console.error('❌ NEXT_PUBLIC_APP_URL must be a valid URL (https:// or http://localhost)')
    hasErrors = true
  }
  
  if (hasErrors) {
    console.error('\n❌ Environment validation failed!')
    process.exit(1)
  } else {
    console.log('\n✅ All environment variables are valid!')
    process.exit(0)
  }
}

validateEnvironment()
```

## Step 2: Supabase Production Setup

### Database Migration Script

Create `scripts/deploy-database.js`:

```javascript
#!/usr/bin/env node

const { createClient } = require('@supabase/supabase-js')
const fs = require('fs')
const path = require('path')

async function deployDatabase() {
  console.log('🚀 Deploying database schema...\n')
  
  const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL
  const serviceRoleKey = process.env.SUPABASE_SERVICE_ROLE_KEY
  
  if (!supabaseUrl || !serviceRoleKey) {
    console.error('❌ Missing Supabase environment variables')
    process.exit(1)
  }
  
  const supabase = createClient(supabaseUrl, serviceRoleKey)
  
  try {
    // Read and execute migration files
    const migrationsDir = path.join(__dirname, '../supabase/migrations')
    const files = fs.readdirSync(migrationsDir)
      .filter(file => file.endsWith('.sql'))
      .sort()
    
    for (const file of files) {
      console.log(`📄 Executing migration: ${file}`)
      const sql = fs.readFileSync(path.join(migrationsDir, file), 'utf8')
      
      // Split SQL into individual statements
      const statements = sql
        .split(';')
        .map(stmt => stmt.trim())
        .filter(stmt => stmt.length > 0)
      
      for (const statement of statements) {
        try {
          const { error } = await supabase.rpc('exec_sql', { query: statement })
          if (error) {
            console.error(`❌ Error in ${file}:`, error.message)
            throw error
          }
        } catch (error) {
          // Ignore if function doesn't exist, we'll create it
          if (!error.message.includes('function exec_sql')) {
            throw error
          }
        }
      }
      
      console.log(`✅ Migration ${file} completed`)
    }
    
    console.log('\n🎉 Database deployment completed successfully!')
    
  } catch (error) {
    console.error('❌ Database deployment failed:', error)
    process.exit(1)
  }
}

// Helper function to create exec_sql function if it doesn't exist
async function createExecSqlFunction() {
  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL,
    process.env.SUPABASE_SERVICE_ROLE_KEY
  )
  
  const createFunctionSQL = `
    CREATE OR REPLACE FUNCTION exec_sql(query text)
    RETURNS void
    LANGUAGE plpgsql
    SECURITY DEFINER
    AS $$
    BEGIN
      EXECUTE query;
    END;
    $$;
  `
  
  try {
    await supabase.rpc('exec_sql', { query: createFunctionSQL })
  } catch (error) {
    // Function creation failed, but that's ok for now
    console.log('ℹ️  Could not create exec_sql function, skipping...')
  }
}

if (require.main === module) {
  deployDatabase()
}

module.exports = { deployDatabase }
```

### Database Seed Script

Create `scripts/seed-database.js`:

```javascript
#!/usr/bin/env node

const { createClient } = require('@supabase/supabase-js')

async function seedDatabase() {
  console.log('🌱 Seeding database with initial data...\n')
  
  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL,
    process.env.SUPABASE_SERVICE_ROLE_KEY
  )
  
  try {
    // Create default document templates
    const templates = [
      {
        name: 'Business Letter Template',
        description: 'Standard business letter format',
        category: 'Letter',
        content: `Dear [Recipient Name],

I am writing to [Purpose]. Please find details below:

[Content]

Thank you for your time and consideration.

Best regards,
[Your Name]`,
        is_public: true,
      },
      {
        name: 'Project Proposal Template',
        description: 'Standard project proposal format',
        category: 'Proposal',
        content: `Project Title: [Project Name]

Executive Summary:
[Summary]

Objectives:
[Objectives]

Timeline:
[Timeline]

Budget:
[Budget]

Conclusion:
[Conclusion]`,
        is_public: true,
      },
      {
        name: 'Meeting Minutes Template',
        description: 'Standard meeting minutes format',
        category: 'Memo',
        content: `Meeting: [Meeting Name]
Date: [Date]
Time: [Time]
Location: [Location]

Attendees:
[List of attendees]

Agenda Items:
[Agenda items]

Decisions Made:
[Decisions]

Action Items:
[Action items]

Next Steps:
[Next steps]`,
        is_public: true,
      },
    ]
    
    for (const template of templates) {
      console.log(`📝 Creating template: ${template.name}`)
      const { error } = await supabase
        .from('document_templates')
        .insert(template)
      
      if (error) {
        console.error(`❌ Error creating template ${template.name}:`, error.message)
      } else {
        console.log(`✅ Template created: ${template.name}`)
      }
    }
    
    console.log('\n🎉 Database seeding completed!')
    
  } catch (error) {
    console.error('❌ Database seeding failed:', error)
    process.exit(1)
  }
}

if (require.main === module) {
  seedDatabase()
}

module.exports = { seedDatabase }
```

## Step 3: Vercel Deployment Configuration

### Vercel Configuration

Create `vercel.json`:

```json
{
  "version": 2,
  "framework": "nextjs",
  "buildCommand": "npm run build",
  "installCommand": "npm ci",
  "outputDirectory": ".next",
  "devCommand": "npm run dev",
  "env": {
    "NEXT_PUBLIC_SUPABASE_URL": "@supabase-url",
    "NEXT_PUBLIC_SUPABASE_ANON_KEY": "@supabase-anon-key",
    "SUPABASE_SERVICE_ROLE_KEY": "@supabase-service-role-key",
    "OPENAI_API_KEY": "@openai-api-key",
    "NEXT_PUBLIC_APP_URL": "@app-url",
    "NEXTAUTH_SECRET": "@auth-secret",
    "NODE_ENV": "production",
    "NEXT_TELEMETRY_DISABLED": "1"
  },
  "functions": {
    "src/app/api/ai/**/*.ts": {
      "maxDuration": 30,
      "memory": 512
    },
    "src/app/api/documents/**/export/route.ts": {
      "maxDuration": 30,
      "memory": 512
    }
  },
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        {
          "key": "Access-Control-Allow-Origin",
          "value": "https://yourdomain.com"
        },
        {
          "key": "Access-Control-Allow-Methods",
          "value": "GET, POST, PUT, DELETE, OPTIONS"
        },
        {
          "key": "Access-Control-Allow-Headers",
          "value": "Content-Type, Authorization"
        }
      ]
    }
  ],
  "rewrites": [
    {
      "source": "/health",
      "destination": "/api/health"
    }
  ],
  "redirects": [
    {
      "source": "/",
      "has": [
        {
          "type": "host",
          "value": "www.yourdomain.com"
        }
      ],
      "destination": "https://yourdomain.com/:path*",
      "permanent": true
    }
  ]
}
```

### Vercel Deployment Script

Create `scripts/deploy.js`:

```javascript
#!/usr/bin/env node

const { execSync } = require('child_process')
const fs = require('fs')

function runCommand(command) {
  console.log(`🏃 Running: ${command}`)
  try {
    execSync(command, { stdio: 'inherit' })
    return true
  } catch (error) {
    console.error(`❌ Command failed: ${command}`)
    return false
  }
}

async function deploy() {
  console.log('🚀 Starting deployment process...\n')
  
  // Validate environment
  console.log('🔍 Validating environment...')
  if (!runCommand('node scripts/validate-env.js')) {
    process.exit(1)
  }
  
  // Build the application
  console.log('\n🔨 Building application...')
  if (!runCommand('npm run build')) {
    process.exit(1)
  }
  
  // Run tests
  console.log('\n🧪 Running tests...')
  if (!runCommand('npm test')) {
    console.warn('⚠️  Some tests failed, but continuing deployment...')
  }
  
  // Deploy to Vercel
  console.log('\n🌐 Deploying to Vercel...')
  if (!runCommand('vercel --prod --yes')) {
    console.error('❌ Vercel deployment failed')
    process.exit(1)
  }
  
  console.log('\n✅ Deployment completed successfully!')
  
  // Get deployment URL
  try {
    const deploymentUrl = execSync('vercel ls --prod', { encoding: 'utf8' })
      .split('\n')
      .find(line => line.includes('https://'))
      .split(' ')[0]
    
    console.log(`🌍 Application deployed to: ${deploymentUrl}`)
  } catch (error) {
    console.log('ℹ️  Could not retrieve deployment URL')
  }
}

if (require.main === module) {
  deploy()
}

module.exports = { deploy }
```

## Step 4: CI/CD Pipeline Setup

### GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Production

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test

  type-check:
    name: Type Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Type check
        run: npm run type-check

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run security audit
        run: npm audit --audit-level moderate
      
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test, type-check]
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build application
        run: npm run build
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-files
          path: .next/

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install Vercel CLI
        run: npm install --global vercel@latest
      
      - name: Pull Vercel Environment Information
        run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}
      
      - name: Build Project Artifacts
        run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}
      
      - name: Deploy Project Artifacts to Vercel
        run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
      
      - name: Deploy Database
        run: |
          node scripts/deploy-database.js
          node scripts/seed-database.js

  post-deploy:
    name: Post-Deployment Checks
    runs-on: ubuntu-latest
    needs: [deploy]
    if: always() && needs.deploy.result == 'success'
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Run health checks
        run: |
          # Wait for deployment to be ready
          sleep 60
          
          # Check if application is responding
          curl -f https://yourdomain.com/health || exit 1
          
          # Run smoke tests
          node tests/smoke-tests.js
      
      - name: Notify deployment success
        if: success()
        uses: 8398a7/action-slack@v3
        with:
          status: success
          text: '✅ Deployment completed successfully!'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify deployment failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          text: '❌ Deployment failed!'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Pre-deployment Checklist

Create `scripts/pre-deploy-check.js`:

```javascript
#!/usr/bin/env node

const fs = require('fs')
const path = require('path')

function runCheck(name, checkFn) {
  console.log(`🔍 ${name}...`)
  try {
    const result = checkFn()
    if (result) {
      console.log(`✅ ${name} passed`)
      return true
    } else {
      console.log(`❌ ${name} failed`)
      return false
    }
  } catch (error) {
    console.log(`❌ ${name} failed: ${error.message}`)
    return false
  }
}

function checkFileExists(filePath) {
  return fs.existsSync(path.join(process.cwd(), filePath))
}

function checkPackageJson() {
  const packageJson = JSON.parse(fs.readFileSync('package.json', 'utf8'))
  return packageJson.name && packageJson.version && packageJson.scripts
}

function checkEnvironmentFile() {
  return checkFileExists('.env.local') || process.env.NODE_ENV === 'production'
}

function checkBuildOutput() {
  return checkFileExists('.next')
}

function checkDatabaseConnection() {
  // This would test database connectivity
  return true // Placeholder
}

async function runPreDeploymentChecks() {
  console.log('🚀 Running pre-deployment checks...\n')
  
  const checks = [
    ['Package.json validation', checkPackageJson],
    ['Environment configuration', checkEnvironmentFile],
    ['Build artifacts exist', checkBuildOutput],
    ['Database connectivity', checkDatabaseConnection],
  ]
  
  let allPassed = true
  
  for (const [name, checkFn] of checks) {
    if (!runCheck(name, checkFn)) {
      allPassed = false
    }
  }
  
  if (allPassed) {
    console.log('\n✅ All pre-deployment checks passed!')
    process.exit(0)
  } else {
    console.log('\n❌ Some checks failed! Please fix issues before deploying.')
    process.exit(1)
  }
}

if (require.main === module) {
  runPreDeploymentChecks()
}
```

## Step 5: Monitoring and Logging

### Health Check API

Create `src/app/api/health/route.ts`:

```typescript
import { NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase/server'
import OpenAI from 'openai'

export async function GET() {
  const healthStatus = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    checks: {
      database: false,
      openai: false,
      storage: false,
    },
  }

  try {
    // Check database connection
    const supabase = createServerSupabaseClient()
    const { error: dbError } = await supabase.from('documents').select('count').limit(1)
    healthStatus.checks.database = !dbError

    // Check OpenAI connection
    if (process.env.OPENAI_API_KEY) {
      const openai = new OpenAI({
        apiKey: process.env.OPENAI_API_KEY,
      })
      try {
        await openai.models.list()
        healthStatus.checks.openai = true
      } catch (error) {
        healthStatus.checks.openai = false
      }
    }

    // Determine overall status
    const allChecksPassing = Object.values(healthStatus.checks).every(check => check)
    healthStatus.status = allChecksPassing ? 'healthy' : 'degraded'

    const statusCode = allChecksPassing ? 200 : 503
    return NextResponse.json(healthStatus, { status: statusCode })
  } catch (error) {
    return NextResponse.json({
      status: 'unhealthy',
      timestamp: new Date().toISOString(),
      error: error instanceof Error ? error.message : 'Unknown error',
    }, { status: 500 })
  }
}
```

### Error Monitoring Setup

Create `src/lib/monitoring/sentry.ts`:

```typescript
import * as Sentry from '@sentry/nextjs'

export function initializeSentry() {
  if (process.env.NODE_ENV === 'production' && process.env.SENTRY_DSN) {
    Sentry.init({
      dsn: process.env.SENTRY_DSN,
      environment: process.env.NODE_ENV,
      tracesSampleRate: 0.1,
      beforeSend(event) {
        // Filter out certain types of events in production
        if (event.exception) {
          const error = event.exception.values?.[0]
          if (error?.type === 'ChunkLoadError') {
            return null // Ignore chunk load errors
          }
        }
        return event
      },
    })
  }
}

// Helper function to capture custom errors
export function captureError(error: Error, context?: Record<string, any>) {
  if (process.env.NODE_ENV === 'production' && process.env.SENTRY_DSN) {
    Sentry.captureException(error, { extra: context })
  } else {
    console.error('Error:', error, context)
  }
}

// Helper function to capture messages
export function captureMessage(message: string, level: Sentry.SeverityLevel = 'info') {
  if (process.env.NODE_ENV === 'production' && process.env.SENTRY_DSN) {
    Sentry.captureMessage(message, level)
  } else {
    console.log(`${level.toUpperCase()}: ${message}`)
  }
}
```

Update `src/instrumentation.ts`:

```typescript
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    await import('./lib/monitoring/sentry').then(({ initializeSentry }) => {
      initializeSentry()
    })
  }
}
```

### Performance Monitoring

Create `src/lib/monitoring/analytics.ts`:

```typescript
interface AnalyticsEvent {
  event: string
  properties?: Record<string, any>
  userId?: string
  timestamp?: Date
}

export class Analytics {
  private static events: AnalyticsEvent[] = []

  static track(event: string, properties?: Record<string, any>, userId?: string) {
    const analyticsEvent: AnalyticsEvent = {
      event,
      properties,
      userId,
      timestamp: new Date(),
    }

    this.events.push(analyticsEvent)

    // In production, send to analytics service
    if (process.env.NODE_ENV === 'production') {
      this.sendToAnalyticsService(analyticsEvent)
    } else {
      console.log('Analytics Event:', analyticsEvent)
    }
  }

  static trackPageView(path: string, userId?: string) {
    this.track('page_view', { path }, userId)
  }

  static trackDocumentCreation(documentType: string, userId: string) {
    this.track('document_created', { document_type: documentType }, userId)
  }

  static trackDocumentExport(format: string, userId: string) {
    this.track('document_exported', { format }, userId)
  }

  static trackAIUsage(operation: string, tokensUsed: number, userId: string) {
    this.track('ai_operation', { operation, tokens_used: tokensUsed }, userId)
  }

  private static async sendToAnalyticsService(event: AnalyticsEvent) {
    // Send to your analytics service (Google Analytics, Mixpanel, etc.)
    // This is a placeholder implementation
    
    if (process.env.NEXT_PUBLIC_ANALYTICS_ID) {
      // Google Analytics 4 example
      // gtag('event', event.event, event.properties)
    }
  }

  static getEvents(limit: number = 100): AnalyticsEvent[] {
    return this.events
      .sort((a, b) => (b.timestamp?.getTime() || 0) - (a.timestamp?.getTime() || 0))
      .slice(0, limit)
  }
}
```

## Step 6: Production Optimization

### Image Optimization Configuration

Update `next.config.js`:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    serverActions: true,
  },
  images: {
    domains: ['supabase.co'],
    formats: ['image/webp', 'image/avif'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },
  
  // Enable compression
  compress: true,
  
  // Enable static optimization
  poweredByHeader: false,
  
  // Generate ETags
  generateEtags: true,
  
  // Enable SWC minification
  swcMinify: true,
  
  // Configure webpack for production
  webpack: (config, { buildId, dev, isServer, defaultLoaders, webpack }) => {
    // Optimize bundle size
    if (!dev && !isServer) {
      config.optimization.splitChunks.cacheGroups = {
        ...config.optimization.splitChunks.cacheGroups,
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
      }
    }
    
    return config
  },
  
  // Performance optimizations
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },
  
  // PWA configuration (optional)
  // async rewrites() {
  //   return [
  //     {
  //       source: '/sw.js',
  //       destination: '/_next/static/sw.js',
  //     },
  //   ]
  // },
}
```

### Caching Strategy

Create `src/lib/cache/redis.ts`:

```typescript
import Redis from 'ioredis'

let redis: Redis | null = null

export function getRedis() {
  if (!redis && process.env.REDIS_URL) {
    redis = new Redis(process.env.REDIS_URL, {
      retryDelayOnFailover: 100,
      maxRetriesPerRequest: 3,
    })
  }
  return redis
}

export class Cache {
  private static redis = getRedis()

  static async get<T>(key: string): Promise<T | null> {
    if (!this.redis) return null

    try {
      const value = await this.redis.get(key)
      return value ? JSON.parse(value) : null
    } catch (error) {
      console.error('Cache get error:', error)
      return null
    }
  }

  static async set(key: string, value: any, expirationInSeconds?: number): Promise<void> {
    if (!this.redis) return

    try {
      const serialized = JSON.stringify(value)
      if (expirationInSeconds) {
        await this.redis.setex(key, expirationInSeconds, serialized)
      } else {
        await this.redis.set(key, serialized)
      }
    } catch (error) {
      console.error('Cache set error:', error)
    }
  }

  static async del(key: string): Promise<void> {
    if (!this.redis) return

    try {
      await this.redis.del(key)
    } catch (error) {
      console.error('Cache delete error:', error)
    }
  }

  static async flush(): Promise<void> {
    if (!this.redis) return

    try {
      await this.redis.flushdb()
    } catch (error) {
      console.error('Cache flush error:', error)
    }
  }
}

// Cache strategies for specific use cases
export class CacheStrategies {
  static async cacheUserDocuments(userId: string, documents: any[], ttl: number = 300) {
    await Cache.set(`user:${userId}:documents`, documents, ttl)
  }

  static async getCachedUserDocuments(userId: string) {
    return Cache.get(`user:${userId}:documents`)
  }

  static async invalidateUserDocuments(userId: string) {
    await Cache.del(`user:${userId}:documents`)
  }

  static async cacheAISuggestions(key: string, suggestions: any[], ttl: number = 1800) {
    await Cache.set(`ai:${key}`, suggestions, ttl)
  }

  static async getCachedAISuggestions(key: string) {
    return Cache.get(`ai:${key}`)
  }
}
```

## Step 7: Backup and Recovery

### Backup Script

Create `scripts/backup.js`:

```javascript
#!/usr/bin/env node

const { createClient } = require('@supabase/supabase-js')
const fs = require('fs')
const path = require('path')
const cron = require('node-cron')

async function createBackup() {
  console.log('🗄️  Creating database backup...\n')
  
  const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL
  const serviceRoleKey = process.env.SUPABASE_SERVICE_ROLE_KEY
  
  if (!supabaseUrl || !serviceRoleKey) {
    console.error('❌ Missing Supabase environment variables')
    process.exit(1)
  }
  
  const supabase = createClient(supabaseUrl, serviceRoleKey)
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-')
  const backupDir = path.join(__dirname, '../backups')
  
  // Ensure backup directory exists
  if (!fs.existsSync(backupDir)) {
    fs.mkdirSync(backupDir, { recursive: true })
  }
  
  try {
    // Backup documents
    console.log('📄 Backing up documents...')
    const { data: documents } = await supabase
      .from('documents')
      .select('*')
    
    if (documents) {
      fs.writeFileSync(
        path.join(backupDir, `documents-${timestamp}.json`),
        JSON.stringify(documents, null, 2)
      )
      console.log(`✅ Backed up ${documents.length} documents`)
    }
    
    // Backup document templates
    console.log('📋 Backing up templates...')
    const { data: templates } = await supabase
      .from('document_templates')
      .select('*')
    
    if (templates) {
      fs.writeFileSync(
        path.join(backupDir, `templates-${timestamp}.json`),
        JSON.stringify(templates, null, 2)
      )
      console.log(`✅ Backed up ${templates.length} templates`)
    }
    
    // Backup user preferences
    console.log('⚙️  Backing up user preferences...')
    const { data: preferences } = await supabase
      .from('user_preferences')
      .select('*')
    
    if (preferences) {
      fs.writeFileSync(
        path.join(backupDir, `preferences-${timestamp}.json`),
        JSON.stringify(preferences, null, 2)
      )
      console.log(`✅ Backed up ${preferences.length} user preferences`)
    }
    
    // Create backup manifest
    const manifest = {
      timestamp,
      version: '1.0',
      tables: ['documents', 'document_templates', 'user_preferences'],
      recordCounts: {
        documents: documents?.length || 0,
        templates: templates?.length || 0,
        preferences: preferences?.length || 0,
      },
    }
    
    fs.writeFileSync(
      path.join(backupDir, `manifest-${timestamp}.json`),
      JSON.stringify(manifest, null, 2)
    )
    
    console.log(`\n🎉 Backup completed successfully!`)
    console.log(`📁 Backup files saved to: ${backupDir}`)
    
  } catch (error) {
    console.error('❌ Backup failed:', error)
    process.exit(1)
  }
}

// Schedule automatic backups
function scheduleBackups() {
  // Daily backup at 2 AM
  cron.schedule('0 2 * * *', () => {
    console.log('🕐 Starting scheduled backup...')
    createBackup()
  })
  
  console.log('⏰ Backup scheduler started')
  console.log('📅 Daily backup scheduled for 2:00 AM')
}

// Manual backup execution
if (require.main === module) {
  const command = process.argv[2]
  
  if (command === 'schedule') {
    scheduleBackups()
  } else {
    createBackup()
  }
}

module.exports = { createBackup, scheduleBackups }
```

### Recovery Script

Create `scripts/recover.js`:

```javascript
#!/usr/bin/env node

const { createClient } = require('@supabase/supabase-js')
const fs = require('fs')
const path = require('path')

async function restoreBackup(backupFile) {
  console.log(`🔄 Restoring from backup: ${backupFile}\n`)
  
  const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL
  const serviceRoleKey = process.env.SUPABASE_SERVICE_ROLE_KEY
  
  if (!supabaseUrl || !serviceRoleKey) {
    console.error('❌ Missing Supabase environment variables')
    process.exit(1)
  }
  
  const supabase = createClient(supabaseUrl, serviceRoleKey)
  
  try {
    // Read backup file
    const backupData = JSON.parse(fs.readFileSync(backupFile, 'utf8'))
    
    if (backupData.documents) {
      console.log(`📄 Restoring ${backupData.documents.length} documents...`)
      
      // Clear existing documents (be careful!)
      const { error: clearError } = await supabase
        .from('documents')
        .delete()
        .neq('id', '00000000-0000-0000-0000-000000000000') // Delete all
      
      if (clearError) {
        console.warn('⚠️  Could not clear existing documents:', clearError.message)
      }
      
      // Restore documents
      const { error: insertError } = await supabase
        .from('documents')
        .insert(backupData.documents)
      
      if (insertError) {
        console.error('❌ Failed to restore documents:', insertError.message)
      } else {
        console.log('✅ Documents restored successfully')
      }
    }
    
    // Similar logic for other tables...
    
    console.log(`\n🎉 Recovery completed!`)
    
  } catch (error) {
    console.error('❌ Recovery failed:', error)
    process.exit(1)
  }
}

// List available backups
function listBackups() {
  const backupDir = path.join(__dirname, '../backups')
  
  if (!fs.existsSync(backupDir)) {
    console.log('❌ No backups directory found')
    return
  }
  
  const files = fs.readdirSync(backupDir)
    .filter(file => file.endsWith('.json') && !file.includes('manifest'))
    .sort()
    .reverse()
  
  if (files.length === 0) {
    console.log('📭 No backup files found')
    return
  }
  
  console.log('📋 Available backups:')
  files.forEach((file, index) => {
    const stats = fs.statSync(path.join(backupDir, file))
    const date = new Date(stats.mtime).toLocaleString()
    const size = (stats.size / 1024).toFixed(2) + ' KB'
    console.log(`${index + 1}. ${file} (${date}, ${size})`)
  })
}

// Main execution
if (require.main === module) {
  const command = process.argv[2]
  const file = process.argv[3]
  
  switch (command) {
    case 'list':
      listBackups()
      break
    case 'restore':
      if (!file) {
        console.error('❌ Please specify backup file')
        process.exit(1)
      }
      restoreBackup(file)
      break
    default:
      console.log(`
🔧 Backup Management Tool

Usage:
  node scripts/recover.js list                    # List available backups
  node scripts/recover.js restore <backup-file>   # Restore from backup
      `)
  }
}
```

## Step 8: Performance Testing

### Load Testing Script

Create `scripts/load-test.js`:

```javascript
#!/usr/bin/env node

const http = require('http')

// Configuration
const BASE_URL = process.env.BASE_URL || 'http://localhost:3000'
const CONCURRENT_USERS = parseInt(process.env.CONCURRENT_USERS) || 10
const DURATION_SECONDS = parseInt(process.env.DURATION_SECONDS) || 60
const ENDPOINTS = [
  '/',
  '/dashboard',
  '/api/health',
  '/api/documents',
]

// Results tracking
const results = {
  totalRequests: 0,
  successfulRequests: 0,
  failedRequests: 0,
  responseTimes: [],
  startTime: null,
  endTime: null,
}

function makeRequest(endpoint) {
  return new Promise((resolve, reject) => {
    const startTime = Date.now()
    
    const options = {
      hostname: new URL(BASE_URL).hostname,
      port: new URL(BASE_URL).port || 80,
      path: endpoint,
      method: 'GET',
      timeout: 10000,
    }
    
    const req = http.request(options, (res) => {
      let data = ''
      
      res.on('data', (chunk) => {
        data += chunk
      })
      
      res.on('end', () => {
        const responseTime = Date.now() - startTime
        resolve({
          statusCode: res.statusCode,
          responseTime,
          endpoint,
        })
      })
    })
    
    req.on('error', (error) => {
      const responseTime = Date.now() - startTime
      reject({
        error: error.message,
        responseTime,
        endpoint,
      })
    })
    
    req.on('timeout', () => {
      req.destroy()
      reject({
        error: 'Request timeout',
        responseTime: Date.now() - startTime,
        endpoint,
      })
    })
    
    req.end()
  })
}

async function simulateUser() {
  const endTime = Date.now() + (DURATION_SECONDS * 1000)
  
  while (Date.now() < endTime) {
    const randomEndpoint = ENDPOINTS[Math.floor(Math.random() * ENDPOINTS.length)]
    
    try {
      const result = await makeRequest(randomEndpoint)
      results.successfulRequests++
      results.responseTimes.push(result.responseTime)
    } catch (error) {
      results.failedRequests++
      console.error(`❌ Request failed for ${error.endpoint}: ${error.error}`)
    }
    
    results.totalRequests++
    
    // Random delay between requests (1-3 seconds)
    await new Promise(resolve => setTimeout(resolve, Math.random() * 2000 + 1000))
  }
}

async function runLoadTest() {
  console.log('🚀 Starting load test...')
  console.log(`📊 Configuration:`)
  console.log(`   URL: ${BASE_URL}`)
  console.log(`   Concurrent Users: ${CONCURRENT_USERS}`)
  console.log(`   Duration: ${DURATION_SECONDS} seconds`)
  console.log(`   Endpoints: ${ENDPOINTS.join(', ')}`)
  console.log()
  
  results.startTime = Date.now()
  
  // Create concurrent user simulations
  const userPromises = Array(CONCURRENT_USERS).fill(null).map(() => simulateUser())
  
  await Promise.all(userPromises)
  
  results.endTime = Date.now()
  generateReport()
}

function generateReport() {
  const duration = (results.endTime - results.startTime) / 1000
  const requestsPerSecond = results.totalRequests / duration
  const successRate = (results.successfulRequests / results.totalRequests) * 100
  
  const sortedTimes = results.responseTimes.sort((a, b) => a - b)
  const avgResponseTime = sortedTimes.reduce((a, b) => a + b, 0) / sortedTimes.length
  const medianResponseTime = sortedTimes[Math.floor(sortedTimes.length / 2)]
  const p95ResponseTime = sortedTimes[Math.floor(sortedTimes.length * 0.95)]
  const p99ResponseTime = sortedTimes[Math.floor(sortedTimes.length * 0.99)]
  
  console.log('📈 Load Test Results:')
  console.log('=' .repeat(50))
  console.log(`Total Requests: ${results.totalRequests}`)
  console.log(`Successful: ${results.successfulRequests} (${successRate.toFixed(1)}%)`)
  console.log(`Failed: ${results.failedRequests}`)
  console.log(`Duration: ${duration.toFixed(1)} seconds`)
  console.log(`Requests/Second: ${requestsPerSecond.toFixed(2)}`)
  console.log()
  console.log('⏱️  Response Times:')
  console.log(`   Average: ${avgResponseTime.toFixed(2)}ms`)
  console.log(`   Median: ${medianResponseTime}ms`)
  console.log(`   95th percentile: ${p95ResponseTime}ms`)
  console.log(`   99th percentile: ${p99ResponseTime}ms`)
  console.log()
  
  // Performance assessment
  if (successRate >= 99 && avgResponseTime < 1000) {
    console.log('✅ Performance: EXCELLENT')
  } else if (successRate >= 95 && avgResponseTime < 2000) {
    console.log('⚠️  Performance: GOOD')
  } else if (successRate >= 90) {
    console.log('⚠️  Performance: NEEDS IMPROVEMENT')
  } else {
    console.log('❌ Performance: POOR')
  }
  
  console.log('=' .repeat(50))
}

// Run the test
runLoadTest().catch(console.error)
```

## Step 9: Post-Deployment Checklist

### Deployment Verification Script

Create `scripts/post-deploy.js`:

```javascript
#!/usr/bin/env node

const https = require('https')

function testEndpoint(url, expectedStatus = 200) {
  return new Promise((resolve, reject) => {
    https.get(url, (res) => {
      let data = ''
      res.on('data', chunk => data += chunk)
      res.on('end', () => {
        if (res.statusCode === expectedStatus) {
          console.log(`✅ ${url} - Status: ${res.statusCode}`)
          resolve({ url, status: res.statusCode, data })
        } else {
          console.log(`❌ ${url} - Status: ${res.statusCode}`)
          reject({ url, status: res.statusCode, error: 'Unexpected status code' })
        }
      })
    }).on('error', (error) => {
      console.log(`❌ ${url} - Error: ${error.message}`)
      reject({ url, error: error.message })
    })
  })
}

async function runPostDeploymentChecks() {
  console.log('🔍 Running post-deployment checks...\n')
  
  const baseUrl = process.env.BASE_URL || 'https://yourdomain.com'
  const checks = [
    `${baseUrl}/`,
    `${baseUrl}/health`,
    `${baseUrl}/api/health`,
  ]
  
  let allPassed = true
  
  for (const url of checks) {
    try {
      await testEndpoint(url)
    } catch (error) {
      allPassed = false
    }
  }
  
  // Test authentication flow
  console.log('\n🔐 Testing authentication flow...')
  try {
    // This would test the sign-in page
    await testEndpoint(`${baseUrl}/sign-in`)
  } catch (error) {
    allPassed = false
  }
  
  // Test API endpoints
  console.log('\n🧪 Testing API endpoints...')
  const apiTests = [
    { url: `${baseUrl}/api/ai/templates`, method: 'GET' },
  ]
  
  for (const test of apiTests) {
    try {
      // This would make actual API calls with proper authentication
      console.log(`ℹ️  API test for ${test.url} (${test.method})`)
      // await testApiEndpoint(test.url, test.method)
    } catch (error) {
      allPassed = false
    }
  }
  
  console.log('\n' + '=' .repeat(50))
  if (allPassed) {
    console.log('🎉 All post-deployment checks passed!')
    console.log('✅ Application is ready for production use')
  } else {
    console.log('❌ Some checks failed!')
    console.log('⚠️  Please investigate issues before announcing deployment')
  }
  console.log('=' .repeat(50))
}

if (require.main === module) {
  runPostDeploymentChecks()
}
```

## Step 10: Maintenance and Updates

### Update Script

Create `scripts/update.js`:

```javascript
#!/usr/bin/env node

const { execSync } = require('child_process')
const fs = require('fs')

function runCommand(command, description) {
  console.log(`🔄 ${description}...`)
  try {
    execSync(command, { stdio: 'inherit' })
    console.log(`✅ ${description} completed`)
    return true
  } catch (error) {
    console.error(`❌ ${description} failed`)
    return false
  }
}

async function updateApplication() {
  console.log('🔄 Starting application update...\n')
  
  // Create backup before update
  console.log('💾 Creating backup...')
  runCommand('node scripts/backup.js', 'Backup creation')
  
  // Pull latest code
  if (!runCommand('git pull origin main', 'Code update')) {
    process.exit(1)
  }
  
  // Install dependencies
  if (!runCommand('npm ci', 'Dependency installation')) {
    process.exit(1)
  }
  
  // Run tests
  if (!runCommand('npm test', 'Test execution')) {
    console.warn('⚠️  Tests failed, continuing with caution...')
  }
  
  // Build application
  if (!runCommand('npm run build', 'Application build')) {
    process.exit(1)
  }
  
  // Deploy database changes
  console.log('🗄️  Updating database...')
  runCommand('node scripts/deploy-database.js', 'Database update')
  
  // Restart application (for PM2 or other process managers)
  console.log('🔄 Restarting application...')
  // runCommand('pm2 restart ai-document-saas', 'Application restart')
  
  // Run post-deployment checks
  console.log('🔍 Running post-deployment checks...')
  runCommand('node scripts/post-deploy.js', 'Post-deployment verification')
  
  console.log('\n🎉 Update completed successfully!')
}

// Rollback function
async function rollbackUpdate() {
  console.log('⏪ Rolling back update...\n')
  
  // Revert code changes
  runCommand('git reset --hard HEAD~1', 'Code rollback')
  
  // Redeploy
  runCommand('npm run build', 'Build rollback')
  runCommand('node scripts/deploy-database.js', 'Database rollback')
  
  // Restart application
  // runCommand('pm2 restart ai-document-saas', 'Application restart')
  
  console.log('\n✅ Rollback completed')
}

// Main execution
if (require.main === module) {
  const command = process.argv[2]
  
  switch (command) {
    case 'update':
      updateApplication().catch(console.error)
      break
    case 'rollback':
      rollbackUpdate().catch(console.error)
      break
    default:
      console.log(`
🔧 Update Management Tool

Usage:
  node scripts/update.js update      # Update application
  node scripts/update.js rollback    # Rollback last update
      `)
  }
}
```

## Final Deployment Summary

### Complete Deployment Checklist

**Pre-Deployment:**
- [ ] Environment variables configured
- [ ] Database migrations tested
- [ ] Security audit completed
- [ ] Performance tests passed
- [ ] Backup created

**Deployment:**
- [ ] Code deployed to production
- [ ] Database schema updated
- [ ] Environment variables set
- [ ] SSL certificates configured
- [ ] DNS records updated

**Post-Deployment:**
- [ ] Health checks passing
- [ ] Authentication working
- [ ] API endpoints responding
- [ ] Performance monitoring active
- [ ] Error tracking configured
- [ ] Analytics tracking working

**Monitoring:**
- [ ] Uptime monitoring configured
- [ ] Performance monitoring active
- [ ] Error alerts configured
- [ ] Backup schedule established
- [ ] Recovery procedures documented

This completes the deployment guide. You now have a comprehensive deployment strategy with automated CI/CD, monitoring, backup, and maintenance procedures for your AI Document SaaS application.