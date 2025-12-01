# AI Integration Guide

## Overview
This guide covers implementing AI-powered document features using OpenAI API, including content generation, improvement suggestions, and document enhancement.

## Step 1: AI Service Configuration

### Install Dependencies

```bash
# Install OpenAI SDK
npm install openai

# Install other AI-related dependencies
npm install zod  # For input validation
npm install rate-limiter-flexible  # For rate limiting
```

### Create OpenAI Service

Create `src/lib/ai/openai.ts`:

```typescript
import OpenAI from 'openai'

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
})

export interface AIGenerationRequest {
  prompt: string
  type: 'generate' | 'improve' | 'suggest' | 'expand' | 'summarize'
  documentType?: string
  context?: string
  tone?: string
  length?: 'short' | 'medium' | 'long'
  language?: string
}

export interface AIGenerationResponse {
  content: string
  wordCount: number
  tokensUsed: number
  suggestions?: string[]
  improvements?: string[]
}

export interface AIImproveRequest {
  content: string
  instructions: string
  focus?: 'clarity' | 'style' | 'grammar' | 'structure' | 'tone'
}

export interface AISuggestion {
  type: 'improvement' | 'expansion' | 'rewrite' | 'style'
  description: string
  suggestion: string
  confidence: number
}

export class AIService {
  private static async callOpenAI(
    messages: any[], 
    model = 'gpt-3.5-turbo',
    maxTokens = 2000
  ) {
    try {
      const response = await openai.chat.completions.create({
        model,
        messages,
        max_tokens: maxTokens,
        temperature: 0.7,
        presence_penalty: 0.1,
        frequency_penalty: 0.1,
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
      content: content.trim(),
      wordCount: content.split(/\s+/).filter(word => word.length > 0).length,
      tokensUsed: response.usage?.total_tokens || 0,
    }
  }

  static async improveContent(request: AIImproveRequest): Promise<AIGenerationResponse> {
    const prompt = this.buildImprovementPrompt(request)
    
    const messages = [
      { 
        role: 'system', 
        content: 'You are a professional writing assistant. Improve the given text while maintaining the original meaning and tone. Provide specific, actionable improvements.' 
      },
      { role: 'user', content: prompt }
    ]

    const response = await this.callOpenAI(messages, 'gpt-4', 1500)
    const improvedContent = response.choices[0]?.message?.content || ''

    return {
      content: improvedContent.trim(),
      wordCount: improvedContent.split(/\s+/).filter(word => word.length > 0).length,
      tokensUsed: response.usage?.total_tokens || 0,
    }
  }

  static async suggestImprovements(content: string): Promise<AISuggestion[]> {
    const prompt = `Analyze the following text and provide specific improvement suggestions:

"${content}"

Please provide 3-5 concrete suggestions focusing on clarity, style, grammar, and effectiveness.`

    const messages = [
      { 
        role: 'system', 
        content: 'Provide specific, actionable writing improvement suggestions. Focus on one aspect per suggestion.' 
      },
      { role: 'user', content: prompt }
    ]

    const response = await this.callOpenAI(messages, 'gpt-4', 1000)
    const suggestionsText = response.choices[0]?.message?.content || ''
    
    return this.parseSuggestions(suggestionsText)
  }

  static async expandContent(content: string, expansionType: 'paragraph' | 'section' | 'detail' = 'paragraph'): Promise<AIGenerationResponse> {
    const prompt = `Expand the following ${expansionType} with relevant details while maintaining the original style and tone:

"${content}"

Please provide a natural expansion that adds value and depth to the content.`

    const messages = [
      { 
        role: 'system', 
        content: 'You are a skilled content expander. Add relevant details and depth while maintaining the original writing style and tone.' 
      },
      { role: 'user', content: prompt }
    ]

    const response = await this.callOpenAI(messages, 'gpt-4', 1500)
    const expandedContent = response.choices[0]?.message?.content || ''

    return {
      content: expandedContent.trim(),
      wordCount: expandedContent.split(/\s+/).filter(word => word.length > 0).length,
      tokensUsed: response.usage?.total_tokens || 0,
    }
  }

  static async summarizeContent(content: string, summaryLength: 'brief' | 'medium' | 'detailed' = 'medium'): Promise<AIGenerationResponse> {
    let maxLength = 100
    switch (summaryLength) {
      case 'brief':
        maxLength = 50
        break
      case 'medium':
        maxLength = 150
        break
      case 'detailed':
        maxLength = 300
        break
    }

    const prompt = `Summarize the following content in approximately ${maxLength} words:

"${content}"

Provide a concise summary that captures the key points and main ideas.`

    const messages = [
      { 
        role: 'system', 
        content: 'Create clear, concise summaries that capture the essence of the original content.' 
      },
      { role: 'user', content: prompt }
    ]

    const response = await this.callOpenAI(messages, 'gpt-3.5-turbo', 500)
    const summary = response.choices[0]?.message?.content || ''

    return {
      content: summary.trim(),
      wordCount: summary.split(/\s+/).filter(word => word.length > 0).length,
      tokensUsed: response.usage?.total_tokens || 0,
    }
  }

  static async rewriteContent(
    content: string, 
    style: 'professional' | 'casual' | 'academic' | 'creative' | 'persuasive' = 'professional'
  ): Promise<AIGenerationResponse> {
    const stylePrompts = {
      professional: 'Rewrite in a professional, business-appropriate tone.',
      casual: 'Rewrite in a friendly, conversational tone.',
      academic: 'Rewrite in an academic, scholarly tone with proper citations.',
      creative: 'Rewrite in a creative, engaging style.',
      persuasive: 'Rewrite to be more persuasive and compelling.'
    }

    const prompt = `${stylePrompts[style]}

Original content:
"${content}"`

    const messages = [
      { 
        role: 'system', 
        content: 'You are a skilled content rewriter. Transform the writing style while preserving the core message and meaning.' 
      },
      { role: 'user', content: prompt }
    ]

    const response = await this.callOpenAI(messages, 'gpt-4', 1500)
    const rewrittenContent = response.choices[0]?.message?.content || ''

    return {
      content: rewrittenContent.trim(),
      wordCount: rewrittenContent.split(/\s+/).filter(word => word.length > 0).length,
      tokensUsed: response.usage?.total_tokens || 0,
    }
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

    if (request.language && request.language !== 'en') {
      prompt += `Write in ${request.language}. `
    }
    
    prompt += 'Create well-structured, professional content that is clear and engaging.'
    
    return prompt
  }

  private static buildImprovementPrompt(request: AIImproveRequest): string {
    let prompt = `Improve the following text based on these instructions: ${request.instructions}\n\n`
    
    if (request.focus) {
      prompt += `Focus specifically on ${request.focus}.\n\n`
    }
    
    prompt += `Original text:\n${request.content}`
    
    return prompt
  }

  private static parseSuggestions(suggestionsText: string): AISuggestion[] {
    const suggestions: AISuggestion[] = []
    const lines = suggestionsText.split('\n').filter(line => line.trim().length > 0)
    
    lines.forEach((line, index) => {
      if (line.match(/^\d+\./) || line.match(/^[-*]/)) {
        suggestions.push({
          type: 'improvement',
          description: line.replace(/^\d+\.\s*/, '').replace(/^[-*]\s*/, ''),
          suggestion: line,
          confidence: 0.8
        })
      }
    })

    // If no structured format found, create general suggestions
    if (suggestions.length === 0) {
      suggestions.push({
        type: 'improvement',
        description: 'General improvement suggestion',
        suggestion: suggestionsText,
        confidence: 0.6
      })
    }

    return suggestions.slice(0, 5) // Limit to 5 suggestions
  }

  // Rate limiting helper
  static checkRateLimit(userId: string): boolean {
    // Implement rate limiting logic here
    // This is a simple example - in production, use a proper rate limiter
    return true
  }

  // Token cost estimation
  static estimateCost(tokensUsed: number, model: string = 'gpt-3.5-turbo'): number {
    const costs = {
      'gpt-3.5-turbo': 0.002, // $0.002 per 1K tokens
      'gpt-4': 0.03, // $0.03 per 1K tokens
      'gpt-4-turbo': 0.01 // $0.01 per 1K tokens
    }
    
    const costPerToken = costs[model as keyof typeof costs] || costs['gpt-3.5-turbo']
    return (tokensUsed / 1000) * costPerToken
  }
}
```

## Step 2: AI API Routes

### Create AI Generation Route

Create `src/app/api/ai/generate/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { AIService } from '@/lib/ai/openai'
import { createServerSupabaseClient } from '@/lib/supabase/server'
import { z } from 'zod'

const generateSchema = z.object({
  prompt: z.string().min(1).max(2000),
  type: z.enum(['generate', 'improve', 'suggest', 'expand', 'summarize']),
  documentType: z.string().optional(),
  context: z.string().optional(),
  tone: z.string().optional(),
  length: z.enum(['short', 'medium', 'long']).optional(),
  language: z.string().optional(),
  documentId: z.string().uuid().optional(),
})

export async function POST(request: NextRequest) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  // Check rate limiting
  if (!AIService.checkRateLimit(user.id)) {
    return NextResponse.json({ 
      error: 'Rate limit exceeded. Please try again later.' 
    }, { status: 429 })
  }

  const body = await request.json()
  
  const validation = generateSchema.safeParse(body)
  if (!validation.success) {
    return NextResponse.json({ 
      error: 'Invalid request data',
      details: validation.error.errors 
    }, { status: 400 })
  }

  const { prompt, type, documentType, context, tone, length, language, documentId } = validation.data

  try {
    let result
    
    switch (type) {
      case 'generate':
        result = await AIService.generateContent({ 
          prompt, 
          type, 
          documentType, 
          tone, 
          length,
          language 
        })
        break
        
      case 'improve':
        if (!context) {
          return NextResponse.json({ 
            error: 'Context is required for improvement requests' 
          }, { status: 400 })
        }
        result = await AIService.improveContent({
          content: context,
          instructions: prompt
        })
        break
        
      case 'suggest':
        if (!context) {
          return NextResponse.json({ 
            error: 'Context is required for suggestion requests' 
          }, { status: 400 })
        }
        const suggestions = await AIService.suggestImprovements(context)
        return NextResponse.json({ suggestions })
        
      case 'expand':
        if (!context) {
          return NextResponse.json({ 
            error: 'Context is required for expansion requests' 
          }, { status: 400 })
        }
        result = await AIService.expandContent(context)
        break
        
      case 'summarize':
        if (!context) {
          return NextResponse.json({ 
            error: 'Context is required for summarization requests' 
          }, { status: 400 })
        }
        result = await AIService.summarizeContent(context, length as any)
        break
        
      default:
        return NextResponse.json({ error: 'Invalid AI request type' }, { status: 400 })
    }

    // Log AI interaction
    if (documentId) {
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
    }

    // Calculate estimated cost
    const estimatedCost = AIService.estimateCost(result.tokensUsed)

    return NextResponse.json({
      ...result,
      estimatedCost,
      model: 'gpt-3.5-turbo'
    })
    
  } catch (error) {
    console.error('AI generation error:', error)
    return NextResponse.json(
      { error: 'AI service temporarily unavailable' }, 
      { status: 500 }
    )
  }
}
```

### Create AI Rewrite Route

Create `src/app/api/ai/rewrite/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { AIService } from '@/lib/ai/openai'
import { createServerSupabaseClient } from '@/lib/supabase/server'
import { z } from 'zod'

const rewriteSchema = z.object({
  content: z.string().min(1),
  style: z.enum(['professional', 'casual', 'academic', 'creative', 'persuasive']),
  documentId: z.string().uuid().optional(),
})

export async function POST(request: NextRequest) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const body = await request.json()
  const validation = rewriteSchema.safeParse(body)
  
  if (!validation.success) {
    return NextResponse.json({ 
      error: 'Invalid request data',
      details: validation.error.errors 
    }, { status: 400 })
  }

  const { content, style, documentId } = validation.data

  try {
    const result = await AIService.rewriteContent(content, style)

    // Log AI interaction
    if (documentId) {
      await supabase
        .from('ai_interactions')
        .insert({
          document_id: documentId,
          user_id: user.id,
          prompt: `Rewrite in ${style} style`,
          response: result.content,
          model: 'gpt-4',
          tokens_used: result.tokensUsed,
          interaction_type: 'rewrite',
        })
    }

    const estimatedCost = AIService.estimateCost(result.tokensUsed, 'gpt-4')

    return NextResponse.json({
      ...result,
      estimatedCost,
      model: 'gpt-4',
      originalStyle: style
    })
    
  } catch (error) {
    console.error('AI rewrite error:', error)
    return NextResponse.json(
      { error: 'AI service temporarily unavailable' }, 
      { status: 500 }
    )
  }
}
```

### Create AI Templates Route

Create `src/app/api/ai/templates/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase/server'

// Predefined AI prompts for different document types
export const AI_TEMPLATES = {
  business_letter: {
    name: 'Business Letter',
    prompt: 'Write a professional business letter with proper formatting. Include: current date, recipient address, subject line, professional greeting, clear purpose, call to action, and formal closing.',
    category: 'Letter',
    variables: ['recipient_name', 'company_name', 'purpose', 'deadline']
  },
  project_proposal: {
    name: 'Project Proposal',
    prompt: 'Create a comprehensive project proposal including: executive summary, objectives, methodology, timeline, budget, risks, and success criteria.',
    category: 'Proposal',
    variables: ['project_name', 'client_name', 'budget', 'timeline']
  },
  meeting_minutes: {
    name: 'Meeting Minutes',
    prompt: 'Format meeting minutes with: meeting details, attendees, agenda items, decisions made, action items, and next steps.',
    category: 'Memo',
    variables: ['meeting_date', 'attendees', 'agenda', 'decisions']
  },
  contract_section: {
    name: 'Contract Section',
    prompt: 'Draft a professional contract section covering: parties involved, terms and conditions, payment terms, deliverables, and legal clauses.',
    category: 'Contract',
    variables: ['party_a', 'party_b', 'terms', 'payment_terms']
  },
  report_executive_summary: {
    name: 'Executive Summary',
    prompt: 'Write a compelling executive summary that highlights key findings, recommendations, and business impact in a concise format.',
    category: 'Report',
    variables: ['findings', 'recommendations', 'business_impact']
  }
}

export async function GET(request: NextRequest) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  return NextResponse.json({ templates: AI_TEMPLATES })
}

export async function POST(request: NextRequest) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const body = await request.json()
  const { templateId, variables = {} } = body

  const template = AI_TEMPLATES[templateId as keyof typeof AI_TEMPLATES]
  
  if (!template) {
    return NextResponse.json({ error: 'Template not found' }, { status: 404 })
  }

  // Replace variables in the prompt
  let prompt = template.prompt
  Object.entries(variables).forEach(([key, value]) => {
    prompt = prompt.replace(new RegExp(`\\{${key}\\}`, 'g'), value as string)
  })

  return NextResponse.json({
    template: {
      ...template,
      processedPrompt: prompt
    }
  })
}
```

## Step 3: AI Service Layer

Create `src/lib/ai/service.ts`:

```typescript
import { AIService, AIGenerationRequest, AIGenerationResponse } from './openai'

export interface AIRequestOptions {
  documentId?: string
  userId: string
  type: string
  prompt: string
  context?: string
  metadata?: Record<string, any>
}

export class AIServiceWrapper {
  static async generateContent(options: AIRequestOptions): Promise<AIGenerationResponse> {
    const request: AIGenerationRequest = {
      prompt: options.prompt,
      type: options.type as any,
      documentType: options.metadata?.documentType,
      context: options.context,
      tone: options.metadata?.tone,
      length: options.metadata?.length,
      language: options.metadata?.language
    }

    const result = await AIService.generateContent(request)
    
    // Log usage for analytics
    this.logUsage(options.userId, options.type, result.tokensUsed)
    
    return result
  }

  static async improveContent(
    userId: string,
    documentId: string,
    content: string,
    instructions: string,
    focus?: string
  ): Promise<AIGenerationResponse> {
    const result = await AIService.improveContent({
      content,
      instructions,
      focus: focus as any
    })

    this.logUsage(userId, 'improve', result.tokensUsed)
    return result
  }

  static async getSuggestions(
    userId: string,
    content: string
  ): Promise<any[]> {
    const suggestions = await AIService.suggestImprovements(content)
    this.logUsage(userId, 'suggest', 0) // Suggestions use fewer tokens
    return suggestions
  }

  static async expandContent(
    userId: string,
    content: string,
    expansionType: 'paragraph' | 'section' | 'detail' = 'paragraph'
  ): Promise<AIGenerationResponse> {
    const result = await AIService.expandContent(content, expansionType)
    this.logUsage(userId, 'expand', result.tokensUsed)
    return result
  }

  static async summarizeContent(
    userId: string,
    content: string,
    summaryLength: 'brief' | 'medium' | 'detailed' = 'medium'
  ): Promise<AIGenerationResponse> {
    const result = await AIService.summarizeContent(content, summaryLength)
    this.logUsage(userId, 'summarize', result.tokensUsed)
    return result
  }

  static async rewriteContent(
    userId: string,
    content: string,
    style: 'professional' | 'casual' | 'academic' | 'creative' | 'persuasive'
  ): Promise<AIGenerationResponse> {
    const result = await AIService.rewriteContent(content, style)
    this.logUsage(userId, 'rewrite', result.tokensUsed)
    return result
  }

  private static logUsage(userId: string, type: string, tokensUsed: number): void {
    // Log to analytics service or database
    console.log(`AI Usage: User ${userId}, Type: ${type}, Tokens: ${tokensUsed}`)
    
    // In a real app, you might want to:
    // 1. Store in database for user analytics
    // 2. Send to analytics service
    // 3. Update user quota/limits
  }

  static calculateUsageStats(interactions: any[]): {
    totalRequests: number
    totalTokens: number
    totalCost: number
    averageTokensPerRequest: number
    typeBreakdown: Record<string, number>
  } {
    const stats = {
      totalRequests: interactions.length,
      totalTokens: interactions.reduce((sum, interaction) => sum + (interaction.tokens_used || 0), 0),
      totalCost: 0,
      averageTokensPerRequest: 0,
      typeBreakdown: {} as Record<string, number>
    }

    stats.totalCost = interactions.reduce((sum, interaction) => {
      return sum + AIService.estimateCost(interaction.tokens_used || 0, interaction.model)
    }, 0)

    stats.averageTokensPerRequest = stats.totalRequests > 0 
      ? Math.round(stats.totalTokens / stats.totalRequests) 
      : 0

    interactions.forEach(interaction => {
      const type = interaction.interaction_type
      stats.typeBreakdown[type] = (stats.typeBreakdown[type] || 0) + 1
    })

    return stats
  }
}
```

## Step 4: AI Hook

Create `src/hooks/use-ai-assistant.ts`:

```typescript
'use client'

import { useState } from 'react'

export interface AIRequest {
  type: 'generate' | 'improve' | 'suggest' | 'expand' | 'summarize' | 'rewrite'
  prompt: string
  context?: string
  documentType?: string
  tone?: string
  length?: 'short' | 'medium' | 'long'
  language?: string
  style?: 'professional' | 'casual' | 'academic' | 'creative' | 'persuasive'
}

export interface AIResponse {
  content?: string
  suggestions?: any[]
  wordCount?: number
  tokensUsed?: number
  estimatedCost?: number
  model?: string
}

export function useAIAssistant() {
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const request = async (data: AIRequest & { documentId?: string }): Promise<AIResponse> => {
    setLoading(true)
    setError(null)

    try {
      const response = await fetch('/api/ai/generate', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(data),
      })

      if (!response.ok) {
        const errorData = await response.json()
        throw new Error(errorData.error || 'AI request failed')
      }

      const result = await response.json()
      return result
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'An error occurred'
      setError(errorMessage)
      throw new Error(errorMessage)
    } finally {
      setLoading(false)
    }
  }

  const rewrite = async (
    content: string, 
    style: 'professional' | 'casual' | 'academic' | 'creative' | 'persuasive',
    documentId?: string
  ): Promise<AIResponse> => {
    setLoading(true)
    setError(null)

    try {
      const response = await fetch('/api/ai/rewrite', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ content, style, documentId }),
      })

      if (!response.ok) {
        const errorData = await response.json()
        throw new Error(errorData.error || 'AI rewrite failed')
      }

      const result = await response.json()
      return result
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'An error occurred'
      setError(errorMessage)
      throw new Error(errorMessage)
    } finally {
      setLoading(false)
    }
  }

  const getTemplates = async () => {
    setLoading(true)
    setError(null)

    try {
      const response = await fetch('/api/ai/templates')
      
      if (!response.ok) {
        throw new Error('Failed to fetch AI templates')
      }

      const result = await response.json()
      return result.templates
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'An error occurred'
      setError(errorMessage)
      throw new Error(errorMessage)
    } finally {
      setLoading(false)
    }
  }

  return {
    loading,
    error,
    request,
    rewrite,
    getTemplates,
  }
}
```

## Step 5: AI Assistant Component

Create `src/components/document/ai-assistant.tsx`:

```typescript
'use client'

import { useState, useEffect } from 'react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { Loader2, Sparkles, Lightbulb, PenTool, FileText, Wand2 } from 'lucide-react'
import { useAIAssistant } from '@/hooks/use-ai-assistant'

interface AIAssistantProps {
  documentId?: string
  currentContent?: string
  onContentGenerated?: (content: string) => void
  onClose?: () => void
}

export function AIAssistant({ 
  documentId, 
  currentContent, 
  onContentGenerated, 
  onClose 
}: AIAssistantProps) {
  const { loading, error, request, getTemplates } = useAIAssistant()
  const [activeTab, setActiveTab] = useState('generate')
  const [templates, setTemplates] = useState<any[]>([])
  
  // Form states
  const [prompt, setPrompt] = useState('')
  const [documentType, setDocumentType] = useState('')
  const [tone, setTone] = useState('')
  const [length, setLength] = useState<'short' | 'medium' | 'long'>('medium')
  const [style, setStyle] = useState<'professional' | 'casual' | 'academic' | 'creative' | 'persuasive'>('professional')
  const [instructions, setInstructions] = useState('')
  const [results, setResults] = useState<string[]>([])
  const [suggestions, setSuggestions] = useState<any[]>([])

  useEffect(() => {
    loadTemplates()
  }, [])

  const loadTemplates = async () => {
    try {
      const templateData = await getTemplates()
      setTemplates(Object.entries(templateData).map(([id, template]: [string, any]) => ({
        id,
        ...template
      })))
    } catch (error) {
      console.error('Failed to load templates:', error)
    }
  }

  const handleGenerate = async () => {
    if (!prompt.trim()) return

    try {
      const response = await request({
        type: 'generate',
        prompt,
        documentType: documentType || undefined,
        tone: tone || undefined,
        length,
        documentId,
        context: currentContent
      })

      if (response.content) {
        setResults(prev => [response.content!, ...prev])
        onContentGenerated?.(response.content)
      }
    } catch (error) {
      console.error('Generation failed:', error)
    }
  }

  const handleImprove = async () => {
    if (!instructions.trim() || !currentContent) return

    try {
      const response = await request({
        type: 'improve',
        prompt: instructions,
        context: currentContent,
        documentId
      })

      if (response.content) {
        setResults(prev => [response.content!, ...prev])
        onContentGenerated?.(response.content)
      }
    } catch (error) {
      console.error('Improvement failed:', error)
    }
  }

  const handleSuggest = async () => {
    if (!currentContent) return

    try {
      const response = await request({
        type: 'suggest',
        prompt: 'Analyze and suggest improvements',
        context: currentContent,
        documentId
      })

      if (response.suggestions) {
        setSuggestions(response.suggestions)
      }
    } catch (error) {
      console.error('Suggestions failed:', error)
    }
  }

  const handleRewrite = async () => {
    if (!currentContent) return

    try {
      const response = await request({
        type: 'rewrite',
        prompt: `Rewrite in ${style} style`,
        style,
        context: currentContent,
        documentId
      })

      if (response.content) {
        setResults(prev => [response.content!, ...prev])
        onContentGenerated?.(response.content)
      }
    } catch (error) {
      console.error('Rewrite failed:', error)
    }
  }

  const handleTemplateUse = (template: any) => {
    setPrompt(template.processedPrompt || template.prompt)
    setActiveTab('generate')
  }

  return (
    <div className="w-80 h-full bg-white border-l border-gray-200 flex flex-col">
      {/* Header */}
      <div className="p-4 border-b border-gray-200">
        <div className="flex items-center justify-between">
          <h3 className="font-semibold flex items-center">
            <Sparkles className="mr-2 h-4 w-4" />
            AI Assistant
          </h3>
          {onClose && (
            <Button variant="ghost" size="sm" onClick={onClose}>
              ×
            </Button>
          )}
        </div>
      </div>

      {/* Tabs */}
      <Tabs value={activeTab} onValueChange={setActiveTab} className="flex-1 flex flex-col">
        <TabsList className="grid w-full grid-cols-4 m-4 mb-0">
          <TabsTrigger value="generate" className="text-xs">
            <Wand2 className="h-3 w-3 mr-1" />
            Generate
          </TabsTrigger>
          <TabsTrigger value="improve" className="text-xs">
            <PenTool className="h-3 w-3 mr-1" />
            Improve
          </TabsTrigger>
          <TabsTrigger value="suggest" className="text-xs">
            <Lightbulb className="h-3 w-3 mr-1" />
            Suggest
          </TabsTrigger>
          <TabsTrigger value="rewrite" className="text-xs">
            <FileText className="h-3 w-3 mr-1" />
            Rewrite
          </TabsTrigger>
        </TabsList>

        <div className="flex-1 overflow-y-auto">
          {/* Generate Tab */}
          <TabsContent value="generate" className="p-4 space-y-4">
            <div>
              <label className="text-sm font-medium">What would you like to create?</label>
              <Textarea
                placeholder="Describe the document you want to create..."
                value={prompt}
                onChange={(e) => setPrompt(e.target.value)}
                className="mt-1"
                rows={3}
              />
            </div>

            <div className="grid grid-cols-2 gap-2">
              <div>
                <label className="text-sm font-medium">Type</label>
                <Select value={documentType} onValueChange={setDocumentType}>
                  <SelectTrigger className="mt-1">
                    <SelectValue placeholder="Select type" />
                  </SelectTrigger>
                  <SelectContent>
                    <SelectItem value="letter">Letter</SelectItem>
                    <SelectItem value="report">Report</SelectItem>
                    <SelectItem value="proposal">Proposal</SelectItem>
                    <SelectItem value="contract">Contract</SelectItem>
                    <SelectItem value="memo">Memo</SelectItem>
                  </SelectContent>
                </Select>
              </div>

              <div>
                <label className="text-sm font-medium">Length</label>
                <Select value={length} onValueChange={(value: any) => setLength(value)}>
                  <SelectTrigger className="mt-1">
                    <SelectValue />
                  </SelectTrigger>
                  <SelectContent>
                    <SelectItem value="short">Short</SelectItem>
                    <SelectItem value="medium">Medium</SelectItem>
                    <SelectItem value="long">Long</SelectItem>
                  </SelectContent>
                </Select>
              </div>
            </div>

            <Button 
              onClick={handleGenerate} 
              disabled={loading || !prompt.trim()}
              className="w-full"
            >
              {loading ? (
                <>
                  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                  Generating...
                </>
              ) : (
                <>
                  <Wand2 className="mr-2 h-4 w-4" />
                  Generate Content
                </>
              )}
            </Button>

            {/* Templates */}
            {templates.length > 0 && (
              <div>
                <label className="text-sm font-medium">Quick Start Templates</label>
                <div className="mt-2 space-y-2">
                  {templates.slice(0, 3).map((template) => (
                    <Button
                      key={template.id}
                      variant="outline"
                      size="sm"
                      className="w-full justify-start text-left"
                      onClick={() => handleTemplateUse(template)}
                    >
                      {template.name}
                    </Button>
                  ))}
                </div>
              </div>
            )}
          </TabsContent>

          {/* Improve Tab */}
          <TabsContent value="improve" className="p-4 space-y-4">
            <div>
              <label className="text-sm font-medium">How would you like to improve it?</label>
              <Textarea
                placeholder="e.g., Make it more professional, improve clarity, add more detail..."
                value={instructions}
                onChange={(e) => setInstructions(e.target.value)}
                className="mt-1"
                rows={3}
              />
            </div>

            <Button 
              onClick={handleImprove} 
              disabled={loading || !instructions.trim() || !currentContent}
              className="w-full"
            >
              {loading ? (
                <>
                  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                  Improving...
                </>
              ) : (
                <>
                  <PenTool className="mr-2 h-4 w-4" />
                  Improve Content
                </>
              )}
            </Button>
          </TabsContent>

          {/* Suggest Tab */}
          <TabsContent value="suggest" className="p-4 space-y-4">
            <p className="text-sm text-muted-foreground">
              Get AI-powered suggestions to improve your document.
            </p>

            <Button 
              onClick={handleSuggest} 
              disabled={loading || !currentContent}
              className="w-full"
            >
              {loading ? (
                <>
                  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                  Analyzing...
                </>
              ) : (
                <>
                  <Lightbulb className="mr-2 h-4 w-4" />
                  Get Suggestions
                </>
              )}
            </Button>

            {suggestions.length > 0 && (
              <div className="space-y-2">
                <h4 className="font-medium">Suggestions:</h4>
                {suggestions.map((suggestion, index) => (
                  <Card key={index} className="p-3">
                    <p className="text-sm">{suggestion.description}</p>
                  </Card>
                ))}
              </div>
            )}
          </TabsContent>

          {/* Rewrite Tab */}
          <TabsContent value="rewrite" className="p-4 space-y-4">
            <div>
              <label className="text-sm font-medium">Rewrite Style</label>
              <Select value={style} onValueChange={(value: any) => setStyle(value)}>
                <SelectTrigger className="mt-1">
                  <SelectValue />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="professional">Professional</SelectItem>
                  <SelectItem value="casual">Casual</SelectItem>
                  <SelectItem value="academic">Academic</SelectItem>
                  <SelectItem value="creative">Creative</SelectItem>
                  <SelectItem value="persuasive">Persuasive</SelectItem>
                </SelectContent>
              </Select>
            </div>

            <Button 
              onClick={handleRewrite} 
              disabled={loading || !currentContent}
              className="w-full"
            >
              {loading ? (
                <>
                  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                  Rewriting...
                </>
              ) : (
                <>
                  <FileText className="mr-2 h-4 w-4" />
                  Rewrite Content
                </>
              )}
            </Button>
          </TabsContent>
        </div>
      </Tabs>

      {/* Results */}
      {results.length > 0 && (
        <div className="border-t border-gray-200 p-4 max-h-32 overflow-y-auto">
          <h4 className="font-medium mb-2">Recent Results:</h4>
          {results.slice(0, 3).map((result, index) => (
            <div key={index} className="text-sm text-muted-foreground truncate">
              {result.substring(0, 100)}...
            </div>
          ))}
        </div>
      )}

      {/* Error Display */}
      {error && (
        <div className="p-4 border-t border-red-200 bg-red-50">
          <p className="text-sm text-red-600">{error}</p>
        </div>
      )}
    </div>
  )
}
```

## Step 6: Testing AI Integration

### Test AI Generation

```typescript
import { AIService } from '@/lib/ai/openai'

export async function testAIGeneration() {
  try {
    const result = await AIService.generateContent({
      prompt: 'Write a professional business letter',
      type: 'generate',
      documentType: 'letter',
      tone: 'professional',
      length: 'medium'
    })
    
    console.log('Generated content:', result.content)
    console.log('Word count:', result.wordCount)
    console.log('Tokens used:', result.tokensUsed)
    
    return result
  } catch (error) {
    console.error('AI generation failed:', error)
    throw error
  }
}
```

### Test AI Improvement

```typescript
export async function testAIImprovement() {
  const content = 'This document needs improvement.'
  
  try {
    const result = await AIService.improveContent({
      content,
      instructions: 'Make this more professional and detailed'
    })
    
    console.log('Improved content:', result.content)
    return result
  } catch (error) {
    console.error('AI improvement failed:', error)
    throw error
  }
}
```

## Rate Limiting Implementation

Create `src/lib/ai/rate-limiter.ts`:

```typescript
import { RateLimiterMemory } from 'rate-limiter-flexible'

// Different limits for different AI operations
const limits = {
  generate: { points: 10, duration: 60 }, // 10 requests per minute
  improve: { points: 20, duration: 60 }, // 20 requests per minute
  suggest: { points: 30, duration: 60 }, // 30 requests per minute
  rewrite: { points: 5, duration: 60 },  // 5 requests per minute
}

const limiters = Object.entries(limits).reduce((acc, [key, config]) => {
  acc[key] = new RateLimiterMemory(config)
  return acc
}, {} as Record<string, RateLimiterMemory>)

export async function checkRateLimit(userId: string, operation: keyof typeof limits): Promise<boolean> {
  try {
    await limiters[operation].consume(userId)
    return true
  } catch (rejRes) {
    return false
  }
}

export async function getRateLimitStatus(userId: string) {
  const status: Record<string, any> = {}
  
  for (const [operation, limiter] of Object.entries(limiters)) {
    try {
      const res = await limiter.get(userId)
      status[operation] = {
        remaining: res ? Math.max(0, limits[operation as keyof typeof limits].points - res.totalHits) : limits[operation as keyof typeof limits].points,
        resetTime: res ? res.msBeforeNext : 0
      }
    } catch {
      status[operation] = {
        remaining: 0,
        resetTime: 0
      }
    }
  }
  
  return status
}
```

## Security Considerations

1. **API Key Security**: Store OpenAI API key securely in environment variables
2. **Input Validation**: Validate all AI request inputs
3. **Rate Limiting**: Implement rate limiting to prevent API abuse
4. **Content Filtering**: Consider implementing content filtering for generated content
5. **Cost Monitoring**: Track API usage and costs
6. **Error Handling**: Handle API failures gracefully

## Monitoring and Analytics

1. **Usage Tracking**: Log AI interactions for analytics
2. **Cost Monitoring**: Track API costs per user
3. **Performance Monitoring**: Monitor API response times
4. **Error Tracking**: Log and monitor AI service errors

## Next Steps

Now that AI integration is implemented, proceed to:
1. [Document Editor Implementation Guide](./06-document-editor-guide.md)
2. [Export Functionality Guide](./07-export-functionality-guide.md)

This completes the AI integration implementation. You now have a comprehensive AI-powered document assistance system with content generation, improvement, suggestions, and rewriting capabilities.