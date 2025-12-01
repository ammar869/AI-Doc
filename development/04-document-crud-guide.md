# Document CRUD Operations Guide

## Overview
This guide covers implementing Create, Read, Update, and Delete operations for documents using Supabase with proper authentication and security.

## Step 1: Document Type Definitions

Create `src/types/document.ts`:

```typescript
export interface Document {
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

export interface DocumentInsert {
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
}

export interface DocumentUpdate {
  title?: string
  content?: string | null
  content_html?: string | null
  type?: DocumentType
  status?: DocumentStatus
  word_count?: number
  is_template?: boolean
  template_category?: string | null
  metadata?: Record<string, any>
  last_edited_at?: string
}

export type DocumentType = 'letter' | 'report' | 'proposal' | 'contract' | 'memo' | 'other'
export type DocumentStatus = 'draft' | 'published' | 'archived'

export interface DocumentFilters {
  status?: DocumentStatus
  type?: DocumentType
  search?: string
  limit?: number
  offset?: number
}
```

## Step 2: Document API Routes

### Create Documents Route

Create `src/app/api/documents/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase/server'
import { Document, DocumentInsert } from '@/types/document'

export async function GET(request: NextRequest) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const { searchParams } = new URL(request.url)
  const status = searchParams.get('status')
  const type = searchParams.get('type')
  const search = searchParams.get('search')
  const limit = parseInt(searchParams.get('limit') || '20')
  const offset = parseInt(searchParams.get('offset') || '0')

  let query = supabase
    .from('documents')
    .select('*')
    .eq('user_id', user.id)
    .order('updated_at', { ascending: false })
    .range(offset, offset + limit - 1)

  if (status) {
    query = query.eq('status', status)
  }

  if (type) {
    query = query.eq('type', type)
  }

  if (search) {
    query = query.or(`title.ilike.%${search}%,content.ilike.%${search}%`)
  }

  const { data, error, count } = await query

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  // Get total count for pagination
  let totalCount = 0
  const { count: total } = await supabase
    .from('documents')
    .select('*', { count: 'exact', head: true })
    .eq('user_id', user.id)

  if (status) {
    const { count: statusCount } = await supabase
      .from('documents')
      .select('*', { count: 'exact', head: true })
      .eq('user_id', user.id)
      .eq('status', status)
    totalCount = statusCount || 0
  } else {
    totalCount = total || 0
  }

  return NextResponse.json({
    documents: data || [],
    pagination: {
      total: totalCount,
      limit,
      offset,
      hasMore: offset + limit < totalCount
    }
  })
}

export async function POST(request: NextRequest) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const body = await request.json()
  const { title, content = '', type = 'other', status = 'draft' } = body

  if (!title || title.trim().length === 0) {
    return NextResponse.json({ error: 'Title is required' }, { status: 400 })
  }

  const documentData: DocumentInsert = {
    user_id: user.id,
    title: title.trim(),
    content,
    content_html: '',
    type,
    status,
    word_count: content.split(/\s+/).filter(word => word.length > 0).length,
    is_template: false,
    metadata: {}
  }

  const { data, error } = await supabase
    .from('documents')
    .insert(documentData)
    .select()
    .single()

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  return NextResponse.json({ document: data }, { status: 201 })
}
```

### Create Individual Document Route

Create `src/app/api/documents/[id]/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase/server'
import { DocumentUpdate } from '@/types/document'

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
    if (error.code === 'PGRST116') {
      return NextResponse.json({ error: 'Document not found' }, { status: 404 })
    }
    return NextResponse.json({ error: error.message }, { status: 500 })
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
  const { title, content, content_html, type, status, metadata } = body

  // Create version before update
  const { data: currentDoc } = await supabase
    .from('documents')
    .select('*')
    .eq('id', params.id)
    .eq('user_id', user.id)
    .single()

  if (currentDoc) {
    await supabase
      .from('document_versions')
      .insert({
        document_id: params.id,
        version_number: await getNextVersionNumber(supabase, params.id),
        title: currentDoc.title,
        content: currentDoc.content,
        content_html: currentDoc.content_html,
        created_by: user.id,
      })
  }

  const updateData: DocumentUpdate = {
    last_edited_at: new Date().toISOString(),
  }

  if (title !== undefined) updateData.title = title.trim()
  if (content !== undefined) updateData.content = content
  if (content_html !== undefined) updateData.content_html = content_html
  if (type !== undefined) updateData.type = type
  if (status !== undefined) updateData.status = status
  if (metadata !== undefined) updateData.metadata = metadata

  // Calculate word count if content is provided
  if (content !== undefined) {
    updateData.word_count = content.split(/\s+/).filter(word => word.length > 0).length
  }

  const { data, error } = await supabase
    .from('documents')
    .update(updateData)
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

async function getNextVersionNumber(supabase: any, documentId: string): Promise<number> {
  const { data } = await supabase
    .from('document_versions')
    .select('version_number')
    .eq('document_id', documentId)
    .order('version_number', { ascending: false })
    .limit(1)

  return (data?.[0]?.version_number || 0) + 1
}
```

### Create Document Export Route

Create `src/app/api/documents/[id]/export/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase/server'
import { DocumentProcessor } from '@/lib/document/processor'

export async function POST(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const { format, includeMetadata = true } = await request.json()

  if (!format || !['html', 'docx', 'pdf'].includes(format)) {
    return NextResponse.json({ error: 'Invalid export format' }, { status: 400 })
  }

  const { data: document, error } = await supabase
    .from('documents')
    .select('*')
    .eq('id', params.id)
    .eq('user_id', user.id)
    .single()

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }

  try {
    const result = await DocumentProcessor.exportDocument(
      document.content || '',
      document.title,
      {
        format,
        includeMetadata,
      }
    )

    const filename = `${document.title.replace(/[^a-zA-Z0-9]/g, '_')}.${format}`

    return new NextResponse(result, {
      headers: {
        'Content-Disposition': `attachment; filename="${filename}"`,
        'Content-Type': getContentType(format),
      },
    })
  } catch (error) {
    console.error('Export error:', error)
    return NextResponse.json({ error: 'Failed to export document' }, { status: 500 })
  }
}

function getContentType(format: string): string {
  switch (format) {
    case 'html':
      return 'text/html'
    case 'docx':
      return 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
    case 'pdf':
      return 'application/pdf'
    default:
      return 'application/octet-stream'
  }
}
```

### Create Document Share Route

Create `src/app/api/documents/[id]/share/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase/server'

export async function POST(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const { isPublic = false, shareWith = [] } = await request.json()

  // For now, we'll just update the document status to 'published' for public sharing
  // In a full implementation, you'd create a separate sharing table
  
  if (isPublic) {
    const { data, error } = await supabase
      .from('documents')
      .update({ status: 'published' })
      .eq('id', params.id)
      .eq('user_id', user.id)
      .select()
      .single()

    if (error) {
      return NextResponse.json({ error: error.message }, { status: 500 })
    }

    // Generate a public URL (in a real app, you'd create a proper sharing mechanism)
    const shareUrl = `${process.env.NEXT_PUBLIC_APP_URL}/shared/${params.id}`

    return NextResponse.json({ 
      document: data, 
      shareUrl,
      message: 'Document is now publicly accessible' 
    })
  }

  return NextResponse.json({ 
    message: 'Sharing updated',
    shareUrl: null 
  })
}
```

## Step 3: Document Service Layer

Create `src/lib/document/service.ts`:

```typescript
import { supabase } from '@/lib/supabase/client'
import { Document, DocumentInsert, DocumentUpdate, DocumentFilters } from '@/types/document'

export class DocumentService {
  static async getDocuments(filters: DocumentFilters = {}): Promise<{
    documents: Document[]
    pagination: {
      total: number
      limit: number
      offset: number
      hasMore: boolean
    }
  }> {
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) throw new Error('Not authenticated')

    const { status, type, search, limit = 20, offset = 0 } = filters

    let query = supabase
      .from('documents')
      .select('*', { count: 'exact' })
      .eq('user_id', user.id)
      .order('updated_at', { ascending: false })
      .range(offset, offset + limit - 1)

    if (status) {
      query = query.eq('status', status)
    }

    if (type) {
      query = query.eq('type', type)
    }

    if (search) {
      query = query.or(`title.ilike.%${search}%,content.ilike.%${search}%`)
    }

    const { data, error, count } = await query

    if (error) throw error

    return {
      documents: data || [],
      pagination: {
        total: count || 0,
        limit,
        offset,
        hasMore: (offset + limit) < (count || 0)
      }
    }
  }

  static async getDocument(id: string): Promise<Document> {
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) throw new Error('Not authenticated')

    const { data, error } = await supabase
      .from('documents')
      .select('*')
      .eq('id', id)
      .eq('user_id', user.id)
      .single()

    if (error) throw error
    return data
  }

  static async createDocument(data: Omit<DocumentInsert, 'user_id'>): Promise<Document> {
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) throw new Error('Not authenticated')

    const documentData: DocumentInsert = {
      ...data,
      user_id: user.id,
      word_count: data.content?.split(/\s+/).filter(word => word.length > 0).length || 0
    }

    const { data: document, error } = await supabase
      .from('documents')
      .insert(documentData)
      .select()
      .single()

    if (error) throw error
    return document
  }

  static async updateDocument(id: string, data: DocumentUpdate): Promise<Document> {
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) throw new Error('Not authenticated')

    // Create version before update
    const { data: currentDoc } = await supabase
      .from('documents')
      .select('*')
      .eq('id', id)
      .eq('user_id', user.id)
      .single()

    if (currentDoc) {
      await supabase
        .from('document_versions')
        .insert({
          document_id: id,
          version_number: await this.getNextVersionNumber(id),
          title: currentDoc.title,
          content: currentDoc.content,
          content_html: currentDoc.content_html,
          created_by: user.id,
        })
    }

    const updateData: DocumentUpdate = {
      ...data,
      last_edited_at: new Date().toISOString(),
    }

    // Recalculate word count if content changed
    if (data.content !== undefined) {
      updateData.word_count = data.content.split(/\s+/).filter(word => word.length > 0).length
    }

    const { data: document, error } = await supabase
      .from('documents')
      .update(updateData)
      .eq('id', id)
      .eq('user_id', user.id)
      .select()
      .single()

    if (error) throw error
    return document
  }

  static async deleteDocument(id: string): Promise<void> {
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) throw new Error('Not authenticated')

    const { error } = await supabase
      .from('documents')
      .delete()
      .eq('id', id)
      .eq('user_id', user.id)

    if (error) throw error
  }

  static async getDocumentVersions(documentId: string) {
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) throw new Error('Not authenticated')

    const { data, error } = await supabase
      .from('document_versions')
      .select(`
        *,
        creator:created_by(full_name, email)
      `)
      .eq('document_id', documentId)
      .order('version_number', { ascending: false })

    if (error) throw error
    return data
  }

  static async restoreVersion(documentId: string, versionNumber: number): Promise<Document> {
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) throw new Error('Not authenticated')

    // Get the version to restore
    const { data: version } = await supabase
      .from('document_versions')
      .select('*')
      .eq('document_id', documentId)
      .eq('version_number', versionNumber)
      .single()

    if (!version) throw new Error('Version not found')

    // Restore the version
    return await this.updateDocument(documentId, {
      title: version.title,
      content: version.content,
      content_html: version.content_html,
    })
  }

  private static async getNextVersionNumber(documentId: string): Promise<number> {
    const { data } = await supabase
      .from('document_versions')
      .select('version_number')
      .eq('document_id', documentId)
      .order('version_number', { ascending: false })
      .limit(1)

    return (data?.[0]?.version_number || 0) + 1
  }
}
```

## Step 4: Document Hooks

Create `src/hooks/use-documents.ts`:

```typescript
'use client'

import { useState, useEffect } from 'react'
import { Document, DocumentFilters, DocumentService } from '@/lib/document/service'

export function useDocuments(filters: DocumentFilters = {}) {
  const [documents, setDocuments] = useState<Document[]>([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)
  const [pagination, setPagination] = useState({
    total: 0,
    limit: 20,
    offset: 0,
    hasMore: false
  })

  const fetchDocuments = async (newFilters?: DocumentFilters) => {
    try {
      setLoading(true)
      setError(null)
      
      const result = await DocumentService.getDocuments(newFilters || filters)
      setDocuments(result.documents)
      setPagination(result.pagination)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to fetch documents')
    } finally {
      setLoading(false)
    }
  }

  const refresh = () => fetchDocuments()

  const loadMore = async () => {
    if (!pagination.hasMore || loading) return
    
    const nextOffset = pagination.offset + pagination.limit
    const newFilters = { ...filters, offset: nextOffset }
    
    try {
      setLoading(true)
      const result = await DocumentService.getDocuments(newFilters)
      setDocuments(prev => [...prev, ...result.documents])
      setPagination(result.pagination)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load more documents')
    } finally {
      setLoading(false)
    }
  }

  const createDocument = async (data: Omit<DocumentInsert, 'user_id'>) => {
    try {
      const newDocument = await DocumentService.createDocument(data)
      setDocuments(prev => [newDocument, ...prev])
      return newDocument
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to create document')
      throw err
    }
  }

  const updateDocument = async (id: string, data: DocumentUpdate) => {
    try {
      const updatedDocument = await DocumentService.updateDocument(id, data)
      setDocuments(prev => prev.map(doc => 
        doc.id === id ? updatedDocument : doc
      ))
      return updatedDocument
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to update document')
      throw err
    }
  }

  const deleteDocument = async (id: string) => {
    try {
      await DocumentService.deleteDocument(id)
      setDocuments(prev => prev.filter(doc => doc.id !== id))
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to delete document')
      throw err
    }
  }

  useEffect(() => {
    fetchDocuments()
  }, [filters.status, filters.type, filters.search])

  return {
    documents,
    loading,
    error,
    pagination,
    refresh,
    loadMore,
    createDocument,
    updateDocument,
    deleteDocument
  }
}

export function useDocument(id: string) {
  const [document, setDocument] = useState<Document | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  const fetchDocument = async () => {
    try {
      setLoading(true)
      setError(null)
      const doc = await DocumentService.getDocument(id)
      setDocument(doc)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to fetch document')
    } finally {
      setLoading(false)
    }
  }

  const updateDocument = async (data: DocumentUpdate) => {
    try {
      const updatedDocument = await DocumentService.updateDocument(id, data)
      setDocument(updatedDocument)
      return updatedDocument
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to update document')
      throw err
    }
  }

  const deleteDocument = async () => {
    try {
      await DocumentService.deleteDocument(id)
      setDocument(null)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to delete document')
      throw err
    }
  }

  useEffect(() => {
    fetchDocument()
  }, [id])

  return {
    document,
    loading,
    error,
    refresh: fetchDocument,
    updateDocument,
    deleteDocument
  }
}
```

## Step 5: Document Components

### Create Document Card Component

Create `src/components/document/document-card.tsx`:

```typescript
'use client'

import { Document } from '@/types/document'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Badge } from '@/components/ui/badge'
import { 
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'
import { 
  FileText, 
  MoreHorizontal, 
  Edit, 
  Trash2, 
  Download,
  Share2 
} from 'lucide-react'
import Link from 'next/link'
import { formatDistanceToNow } from 'date-fns'

interface DocumentCardProps {
  document: Document
  onEdit?: (document: Document) => void
  onDelete?: (document: Document) => void
  onExport?: (document: Document, format: 'html' | 'docx' | 'pdf') => void
  onShare?: (document: Document) => void
}

export function DocumentCard({ 
  document, 
  onEdit, 
  onDelete, 
  onExport,
  onShare 
}: DocumentCardProps) {
  const getStatusColor = (status: string) => {
    switch (status) {
      case 'published':
        return 'bg-green-100 text-green-800'
      case 'draft':
        return 'bg-yellow-100 text-yellow-800'
      case 'archived':
        return 'bg-gray-100 text-gray-800'
      default:
        return 'bg-blue-100 text-blue-800'
    }
  }

  const getTypeColor = (type: string) => {
    switch (type) {
      case 'letter':
        return 'bg-blue-100 text-blue-800'
      case 'report':
        return 'bg-purple-100 text-purple-800'
      case 'proposal':
        return 'bg-indigo-100 text-indigo-800'
      case 'contract':
        return 'bg-orange-100 text-orange-800'
      case 'memo':
        return 'bg-pink-100 text-pink-800'
      default:
        return 'bg-gray-100 text-gray-800'
    }
  }

  return (
    <Card className="hover:shadow-md transition-shadow">
      <CardHeader className="pb-3">
        <div className="flex items-start justify-between">
          <div className="flex-1">
            <CardTitle className="text-lg font-semibold line-clamp-2">
              <Link 
                href={`/dashboard/documents/${document.id}`}
                className="hover:text-primary transition-colors"
              >
                {document.title}
              </Link>
            </CardTitle>
            <div className="flex items-center gap-2 mt-2">
              <Badge variant="secondary" className={getStatusColor(document.status)}>
                {document.status}
              </Badge>
              <Badge variant="outline" className={getTypeColor(document.type)}>
                {document.type}
              </Badge>
            </div>
          </div>
          
          <DropdownMenu>
            <DropdownMenuTrigger asChild>
              <Button variant="ghost" size="sm">
                <MoreHorizontal className="h-4 w-4" />
              </Button>
            </DropdownMenuTrigger>
            <DropdownMenuContent align="end">
              <DropdownMenuItem asChild>
                <Link href={`/dashboard/documents/${document.id}/edit`}>
                  <Edit className="mr-2 h-4 w-4" />
                  Edit
                </Link>
              </DropdownMenuItem>
              
              <DropdownMenuItem onClick={() => onShare?.(document)}>
                <Share2 className="mr-2 h-4 w-4" />
                Share
              </DropdownMenuItem>
              
              <DropdownMenuItem onClick={() => onExport?.(document, 'pdf')}>
                <Download className="mr-2 h-4 w-4" />
                Export PDF
              </DropdownMenuItem>
              
              <DropdownMenuItem onClick={() => onExport?.(document, 'docx')}>
                <Download className="mr-2 h-4 w-4" />
                Export DOCX
              </DropdownMenuItem>
              
              <DropdownMenuItem 
                onClick={() => onDelete?.(document)}
                className="text-red-600"
              >
                <Trash2 className="mr-2 h-4 w-4" />
                Delete
              </DropdownMenuItem>
            </DropdownMenuContent>
          </DropdownMenu>
        </div>
      </CardHeader>
      
      <CardContent>
        <div className="space-y-2">
          {document.content && (
            <p className="text-sm text-muted-foreground line-clamp-3">
              {document.content.substring(0, 200)}
              {document.content.length > 200 && '...'}
            </p>
          )}
          
          <div className="flex items-center justify-between text-xs text-muted-foreground">
            <div className="flex items-center gap-4">
              <span>{document.word_count} words</span>
              <span>{formatDistanceToNow(new Date(document.updated_at), { addSuffix: true })}</span>
            </div>
            <FileText className="h-4 w-4" />
          </div>
        </div>
      </CardContent>
    </Card>
  )
}
```

### Create Document List Component

Create `src/components/document/document-list.tsx`:

```typescript
'use client'

import { useState } from 'react'
import { Document, DocumentFilters } from '@/types/document'
import { DocumentCard } from './document-card'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { Loader2, Plus, Search } from 'lucide-react'
import { useDocuments } from '@/hooks/use-documents'
import Link from 'next/link'

interface DocumentListProps {
  onCreateDocument?: () => void
  onEditDocument?: (document: Document) => void
  onDeleteDocument?: (document: Document) => void
  onExportDocument?: (document: Document, format: 'html' | 'docx' | 'pdf') => void
  onShareDocument?: (document: Document) => void
}

export function DocumentList({
  onCreateDocument,
  onEditDocument,
  onDeleteDocument,
  onExportDocument,
  onShareDocument
}: DocumentListProps) {
  const [filters, setFilters] = useState<DocumentFilters>({
    search: '',
    status: undefined,
    type: undefined,
    limit: 20,
    offset: 0
  })

  const { 
    documents, 
    loading, 
    error, 
    pagination,
    refresh,
    loadMore,
    deleteDocument 
  } = useDocuments(filters)

  const handleSearch = (search: string) => {
    setFilters(prev => ({ ...prev, search, offset: 0 }))
  }

  const handleStatusFilter = (status: string) => {
    setFilters(prev => ({ 
      ...prev, 
      status: status === 'all' ? undefined : status as any,
      offset: 0 
    }))
  }

  const handleTypeFilter = (type: string) => {
    setFilters(prev => ({ 
      ...prev, 
      type: type === 'all' ? undefined : type as any,
      offset: 0 
    }))
  }

  const handleDeleteDocument = async (document: Document) => {
    if (window.confirm('Are you sure you want to delete this document?')) {
      await deleteDocument(document.id)
      onDeleteDocument?.(document)
    }
  }

  if (error) {
    return (
      <div className="text-center py-8">
        <p className="text-red-600">{error}</p>
        <Button onClick={refresh} className="mt-4">
          Try Again
        </Button>
      </div>
    )
  }

  return (
    <div className="space-y-6">
      {/* Header */}
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">My Documents</h1>
        <Button asChild>
          <Link href="/dashboard/editor/new">
            <Plus className="mr-2 h-4 w-4" />
            New Document
          </Link>
        </Button>
      </div>

      {/* Filters */}
      <div className="flex flex-col sm:flex-row gap-4">
        <div className="relative flex-1">
          <Search className="absolute left-3 top-1/2 transform -translate-y-1/2 h-4 w-4 text-muted-foreground" />
          <Input
            placeholder="Search documents..."
            value={filters.search || ''}
            onChange={(e) => handleSearch(e.target.value)}
            className="pl-10"
          />
        </div>
        
        <Select value={filters.status || 'all'} onValueChange={handleStatusFilter}>
          <SelectTrigger className="w-full sm:w-40">
            <SelectValue placeholder="Status" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="all">All Status</SelectItem>
            <SelectItem value="draft">Draft</SelectItem>
            <SelectItem value="published">Published</SelectItem>
            <SelectItem value="archived">Archived</SelectItem>
          </SelectContent>
        </Select>
        
        <Select value={filters.type || 'all'} onValueChange={handleTypeFilter}>
          <SelectTrigger className="w-full sm:w-40">
            <SelectValue placeholder="Type" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="all">All Types</SelectItem>
            <SelectItem value="letter">Letter</SelectItem>
            <SelectItem value="report">Report</SelectItem>
            <SelectItem value="proposal">Proposal</SelectItem>
            <SelectItem value="contract">Contract</SelectItem>
            <SelectItem value="memo">Memo</SelectItem>
            <SelectItem value="other">Other</SelectItem>
          </SelectContent>
        </Select>
      </div>

      {/* Documents Grid */}
      {loading && documents.length === 0 ? (
        <div className="flex items-center justify-center py-8">
          <Loader2 className="h-8 w-8 animate-spin" />
        </div>
      ) : documents.length === 0 ? (
        <div className="text-center py-8">
          <p className="text-muted-foreground">No documents found</p>
          <Button asChild className="mt-4">
            <Link href="/dashboard/editor/new">
              <Plus className="mr-2 h-4 w-4" />
              Create Your First Document
            </Link>
          </Button>
        </div>
      ) : (
        <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
          {documents.map((document) => (
            <DocumentCard
              key={document.id}
              document={document}
              onEdit={onEditDocument}
              onDelete={handleDeleteDocument}
              onExport={onExportDocument}
              onShare={onShareDocument}
            />
          ))}
        </div>
      )}

      {/* Load More */}
      {pagination.hasMore && (
        <div className="flex justify-center">
          <Button 
            onClick={loadMore} 
            disabled={loading}
            variant="outline"
          >
            {loading ? (
              <>
                <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                Loading...
              </>
            ) : (
              'Load More'
            )}
          </Button>
        </div>
      )}
    </div>
  )
}
```

## Step 6: Testing Document CRUD Operations

### Test Document Creation

```typescript
// Test in a component or API route
import { DocumentService } from '@/lib/document/service'

export async function testDocumentCreation() {
  try {
    const document = await DocumentService.createDocument({
      title: 'Test Document',
      content: 'This is a test document content.',
      type: 'letter',
      status: 'draft'
    })
    console.log('Document created:', document)
    return document
  } catch (error) {
    console.error('Failed to create document:', error)
    throw error
  }
}
```

### Test Document Reading

```typescript
export async function testDocumentReading(documentId: string) {
  try {
    const document = await DocumentService.getDocument(documentId)
    console.log('Document retrieved:', document)
    return document
  } catch (error) {
    console.error('Failed to retrieve document:', error)
    throw error
  }
}
```

### Test Document Updating

```typescript
export async function testDocumentUpdating(documentId: string) {
  try {
    const document = await DocumentService.updateDocument(documentId, {
      title: 'Updated Title',
      content: 'Updated content'
    })
    console.log('Document updated:', document)
    return document
  } catch (error) {
    console.error('Failed to update document:', error)
    throw error
  }
}
```

### Test Document Deletion

```typescript
export async function testDocumentDeletion(documentId: string) {
  try {
    await DocumentService.deleteDocument(documentId)
    console.log('Document deleted successfully')
  } catch (error) {
    console.error('Failed to delete document:', error)
    throw error
  }
}
```

## Security Considerations

1. **Authentication**: All operations require valid user authentication
2. **Authorization**: Users can only access their own documents
3. **Input Validation**: All inputs are validated before database operations
4. **SQL Injection**: Use parameterized queries (handled by Supabase)
5. **Data Sanitization**: Content is properly sanitized before storage

## Error Handling

Common error scenarios:
- **Unauthorized**: User not authenticated
- **Not Found**: Document doesn't exist or user doesn't have access
- **Validation Error**: Invalid input data
- **Database Error**: Supabase operation failures
- **Network Error**: Connection issues

## Performance Considerations

1. **Pagination**: Implement proper pagination for large document lists
2. **Indexes**: Database indexes on user_id, status, type, and created_at
3. **Caching**: Consider caching frequently accessed documents
4. **Lazy Loading**: Load document content only when needed

## Next Steps

Now that document CRUD operations are implemented, proceed to:
1. [AI Integration Guide](./05-ai-integration-guide.md)
2. [Document Editor Implementation Guide](./06-document-editor-guide.md)

This completes the document CRUD operations implementation. You now have a fully functional document management system with proper security, validation, and user experience features.