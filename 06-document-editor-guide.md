# Document Editor Implementation Guide using SuperDoc

## Overview
This guide covers implementing a rich document editor using SuperDoc, including document creation, editing, real-time saving, and AI integration.

## Step 1: SuperDoc Installation and Setup

### Install SuperDoc

```bash
# Install SuperDoc
npm install @harbour-enterprises/superdoc

# Install additional dependencies
npm install date-fns  # For date formatting
```

### Install CSS Styles

Add to your global CSS (`src/app/globals.css`):

```css
@import '@harbour-enterprises/superdoc/style.css';

/* Additional custom styles for SuperDoc */
.superdoc-container {
  height: 100%;
  min-height: 600px;
}

.superdoc-editor {
  height: 100%;
}

/* Custom toolbar styles */
.superdoc-toolbar {
  border-bottom: 1px solid hsl(var(--border));
  background: hsl(var(--background));
}

/* Ensure editor fills available space */
.superdoc-content {
  flex: 1;
  overflow-y: auto;
  padding: 1rem;
  line-height: 1.6;
}

/* Word count and status indicators */
.editor-status {
  position: fixed;
  bottom: 1rem;
  right: 1rem;
  background: hsl(var(--background));
  border: 1px solid hsl(var(--border));
  border-radius: 0.5rem;
  padding: 0.5rem 1rem;
  font-size: 0.875rem;
  box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1);
}

/* AI sidebar styles */
.ai-sidebar {
  width: 350px;
  border-left: 1px solid hsl(var(--border));
  background: hsl(var(--background));
}

/* Loading states */
.editor-loading {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  background: hsl(var(--background));
}
```

## Step 2: Document Editor Component

### Create Main Editor Component

Create `src/components/document/superdoc-editor.tsx`:

```typescript
'use client'

import { useEffect, useRef, useState, forwardRef, useImperativeHandle } from 'react'
import { SuperDoc } from '@harbour-enterprises/superdoc'
import { Button } from '@/components/ui/button'
import { Card, CardContent } from '@/components/ui/card'
import { Loader2, Save, Download, Share2, Settings } from 'lucide-react'
import { useDocument } from '@/hooks/use-documents'
import { AISidebar } from './ai-sidebar'
import { DocumentToolbar } from './document-toolbar'
import { formatDistanceToNow } from 'date-fns'

interface SuperDocEditorProps {
  documentId?: string
  initialContent?: string
  title: string
  user: {
    id: string
    name: string
    email: string
  }
  onSave?: (content: string, metadata: any) => void
  onTitleChange?: (title: string) => void
}

export interface EditorRef {
  export: (options?: ExportOptions) => Promise<Blob>
  getContent: () => string
  getWordCount: () => number
  focus: () => void
  insertContent: (content: string) => void
  setReadOnly: (readOnly: boolean) => void
}

interface ExportOptions {
  isFinalDoc?: boolean
  includeComments?: boolean
  format?: 'pdf' | 'docx' | 'html'
}

export const SuperDocEditor = forwardRef<EditorRef, SuperDocEditorProps>(
  ({ documentId, initialContent = '', title, user, onSave, onTitleChange }, ref) => {
    const containerRef = useRef<HTMLDivElement>(null)
    const superdocRef = useRef<SuperDoc | null>(null)
    const [isLoading, setIsLoading] = useState(true)
    const [isSaving, setIsSaving] = useState(false)
    const [wordCount, setWordCount] = useState(0)
    const [lastSaved, setLastSaved] = useState<Date | null>(null)
    const [isAIOpen, setIsAIOpen] = useState(false)
    const [hasUnsavedChanges, setHasUnsavedChanges] = useState(false)

    const { document, updateDocument, loading } = useDocument(documentId || '')

    useImperativeHandle(ref, () => ({
      export: async (options) => {
        if (!superdocRef.current) throw new Error('Editor not ready')
        return await superdocRef.current.export({
          isFinalDoc: options?.isFinalDoc || false,
          includeComments: options?.includeComments || false,
          format: options?.format || 'pdf'
        })
      },
      getContent: () => {
        return superdocRef.current?.getHTML()?.join('') || ''
      },
      getWordCount: () => {
        return wordCount
      },
      focus: () => {
        superdocRef.current?.focus()
      },
      insertContent: (content: string) => {
        superdocRef.current?.insertContent(content)
      },
      setReadOnly: (readOnly: boolean) => {
        superdocRef.current?.setDocumentMode(readOnly ? 'viewing' : 'editing')
      }
    }))

    useEffect(() => {
      if (!containerRef.current || isLoading) return

      const initEditor = async () => {
        try {
          superdocRef.current = new SuperDoc({
            selector: containerRef.current!,
            document: initialContent,
            user: {
              id: user.id,
              name: user.name,
              email: user.email
            },
            onReady: () => {
              setIsLoading(false)
              updateWordCount()
            },
            onContentChange: (content: string) => {
              setHasUnsavedChanges(true)
              updateWordCount()
              
              // Auto-save after 2 seconds of inactivity
              if (documentId) {
                setTimeout(() => {
                  handleAutoSave(content)
                }, 2000)
              }
            },
            onModeChange: (mode: string) => {
              console.log('Editor mode changed to:', mode)
            }
          })

          // Set up event listeners
          if (superdocRef.current) {
            // Listen for document events
            superdocRef.current.addEventListener('document:changed', () => {
              setHasUnsavedChanges(true)
            })

            superdocRef.current.addEventListener('document:saved', () => {
              setHasUnsavedChanges(false)
              setLastSaved(new Date())
            })
          }
        } catch (error) {
          console.error('Failed to initialize editor:', error)
          setIsLoading(false)
        }
      }

      initEditor()

      return () => {
        if (superdocRef.current) {
          superdocRef.current = null
        }
      }
    }, [documentId, initialContent, user])

    const updateWordCount = () => {
      if (superdocRef.current) {
        const content = superdocRef.current.getHTML()?.join('') || ''
        const words = content.replace(/<[^>]*>/g, '').split(/\s+/).filter(word => word.length > 0)
        setWordCount(words.length)
      }
    }

    const handleAutoSave = async (content: string) => {
      if (!documentId || isSaving) return

      setIsSaving(true)
      try {
        await updateDocument(documentId, {
          content,
          content_html: content,
          word_count: wordCount
        })
        setLastSaved(new Date())
        setHasUnsavedChanges(false)
      } catch (error) {
        console.error('Auto-save failed:', error)
      } finally {
        setIsSaving(false)
      }
    }

    const handleManualSave = async () => {
      if (!superdocRef.current || !documentId) return

      setIsSaving(true)
      try {
        const content = superdocRef.current.getHTML()?.join('') || ''
        
        await updateDocument(documentId, {
          content,
          content_html: content,
          word_count: wordCount
        })
        
        setLastSaved(new Date())
        setHasUnsavedChanges(false)
        onSave?.(content, { wordCount, savedAt: new Date() })
      } catch (error) {
        console.error('Manual save failed:', error)
      } finally {
        setIsSaving(false)
      }
    }

    const handleExport = async (format: 'pdf' | 'docx' | 'html') => {
      if (!superdocRef.current) return

      try {
        const blob = await superdocRef.current.export({
          isFinalDoc: true,
          includeComments: false,
          format
        })

        // Download the file
        const url = URL.createObjectURL(blob)
        const a = document.createElement('a')
        a.href = url
        a.download = `${title}.${format}`
        document.body.appendChild(a)
        a.click()
        document.body.removeChild(a)
        URL.revokeObjectURL(url)
      } catch (error) {
        console.error('Export failed:', error)
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
            context: superdocRef.current?.getHTML()?.join('') || ''
          }),
        })

        if (response.ok) {
          const result = await response.json()
          
          // Insert AI response at current cursor position
          if (result.content) {
            superdocRef.current?.insertContent(result.content)
          }
          
          // Auto-save after AI insertion
          if (documentId) {
            setTimeout(() => handleManualSave(), 1000)
          }
        }
      } catch (error) {
        console.error('AI request failed:', error)
      }
    }

    if (loading || isLoading) {
      return (
        <div className="flex items-center justify-center h-screen">
          <div className="text-center">
            <Loader2 className="h-8 w-8 animate-spin mx-auto mb-4" />
            <p className="text-muted-foreground">Loading editor...</p>
          </div>
        </div>
      )
    }

    return (
      <div className="flex h-screen bg-background">
        {/* Main Editor Area */}
        <div className="flex-1 flex flex-col">
          {/* Toolbar */}
          <DocumentToolbar
            title={title}
            onSave={handleManualSave}
            onExport={handleExport}
            onAI={() => setIsAIOpen(!isAIOpen)}
            onSettings={() => {/* Handle settings */}}
            isSaving={isSaving}
            wordCount={wordCount}
            lastSaved={lastSaved}
            hasUnsavedChanges={hasUnsavedChanges}
          />

          {/* Editor Container */}
          <div className="flex-1 relative">
            <div 
              ref={containerRef} 
              className="superdoc-container h-full"
              style={{ height: 'calc(100vh - 80px)' }}
            />
            
            {/* Status Indicator */}
            <div className="editor-status">
              <div className="flex items-center gap-2 text-sm">
                <div className="flex items-center gap-1">
                  <div className={`w-2 h-2 rounded-full ${isSaving ? 'bg-yellow-500' : hasUnsavedChanges ? 'bg-red-500' : 'bg-green-500'}`} />
                  <span>
                    {isSaving ? 'Saving...' : hasUnsavedChanges ? 'Unsaved changes' : 'Saved'}
                  </span>
                </div>
                {lastSaved && (
                  <span className="text-muted-foreground">
                    • {formatDistanceToNow(lastSaved, { addSuffix: true })}
                  </span>
                )}
              </div>
              <div className="text-xs text-muted-foreground mt-1">
                {wordCount} words
              </div>
            </div>
          </div>
        </div>

        {/* AI Sidebar */}
        {isAIOpen && (
          <AISidebar
            documentId={documentId}
            currentContent={superdocRef.current?.getHTML()?.join('') || ''}
            onAIRequest={handleAIRequest}
            onClose={() => setIsAIOpen(false)}
          />
        )}
      </div>
    )
  }
)

SuperDocEditor.displayName = 'SuperDocEditor'
```

## Step 3: Document Toolbar Component

Create `src/components/document/document-toolbar.tsx`:

```typescript
'use client'

import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { 
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'
import { 
  Save, 
  Download, 
  Share2, 
  Settings, 
  Sparkles,
  FileText,
  MoreHorizontal
} from 'lucide-react'
import { formatDistanceToNow } from 'date-fns'

interface DocumentToolbarProps {
  title: string
  onSave?: () => void
  onExport?: (format: 'pdf' | 'docx' | 'html') => void
  onAI?: () => void
  onSettings?: () => void
  onTitleChange?: (title: string) => void
  isSaving?: boolean
  wordCount?: number
  lastSaved?: Date | null
  hasUnsavedChanges?: boolean
}

export function DocumentToolbar({
  title,
  onSave,
  onExport,
  onAI,
  onSettings,
  onTitleChange,
  isSaving = false,
  wordCount = 0,
  lastSaved,
  hasUnsavedChanges = false
}: DocumentToolbarProps) {
  const handleTitleChange = (newTitle: string) => {
    onTitleChange?.(newTitle)
  }

  return (
    <div className="flex items-center justify-between p-4 border-b border-border bg-background">
      {/* Left Section - Title and Status */}
      <div className="flex items-center gap-4 flex-1">
        <Input
          value={title}
          onChange={(e) => handleTitleChange(e.target.value)}
          className="text-lg font-semibold border-none shadow-none p-0 h-auto focus-visible:ring-0"
          placeholder="Untitled Document"
        />
        
        <div className="flex items-center gap-2 text-sm text-muted-foreground">
          {isSaving && (
            <div className="flex items-center gap-1">
              <div className="w-2 h-2 bg-yellow-500 rounded-full animate-pulse" />
              <span>Saving...</span>
            </div>
          )}
          
          {hasUnsavedChanges && !isSaving && (
            <div className="flex items-center gap-1">
              <div className="w-2 h-2 bg-red-500 rounded-full" />
              <span>Unsaved changes</span>
            </div>
          )}
          
          {!hasUnsavedChanges && !isSaving && lastSaved && (
            <div className="flex items-center gap-1">
              <div className="w-2 h-2 bg-green-500 rounded-full" />
              <span>Saved {formatDistanceToNow(lastSaved, { addSuffix: true })}</span>
            </div>
          )}
        </div>
      </div>

      {/* Right Section - Actions */}
      <div className="flex items-center gap-2">
        {/* Word Count */}
        <div className="text-sm text-muted-foreground hidden sm:block">
          {wordCount.toLocaleString()} words
        </div>

        {/* AI Button */}
        <Button
          variant="outline"
          size="sm"
          onClick={onAI}
          className="hidden sm:flex"
        >
          <Sparkles className="h-4 w-4 mr-1" />
          AI Assist
        </Button>

        {/* Save Button */}
        <Button
          variant="outline"
          size="sm"
          onClick={onSave}
          disabled={isSaving}
        >
          <Save className="h-4 w-4 mr-1" />
          Save
        </Button>

        {/* Export Menu */}
        <DropdownMenu>
          <DropdownMenuTrigger asChild>
            <Button variant="outline" size="sm">
              <Download className="h-4 w-4 mr-1" />
              Export
            </Button>
          </DropdownMenuTrigger>
          <DropdownMenuContent align="end">
            <DropdownMenuItem onClick={() => onExport?.('pdf')}>
              <FileText className="mr-2 h-4 w-4" />
              Export as PDF
            </DropdownMenuItem>
            <DropdownMenuItem onClick={() => onExport?.('docx')}>
              <FileText className="mr-2 h-4 w-4" />
              Export as DOCX
            </DropdownMenuItem>
            <DropdownMenuItem onClick={() => onExport?.('html')}>
              <FileText className="mr-2 h-4 w-4" />
              Export as HTML
            </DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>

        {/* More Actions */}
        <DropdownMenu>
          <DropdownMenuTrigger asChild>
            <Button variant="outline" size="sm">
              <MoreHorizontal className="h-4 w-4" />
            </Button>
          </DropdownMenuTrigger>
          <DropdownMenuContent align="end">
            <DropdownMenuItem onClick={onSettings}>
              <Settings className="mr-2 h-4 w-4" />
              Settings
            </DropdownMenuItem>
            <DropdownMenuItem onClick={() => {/* Handle share */}}>
              <Share2 className="mr-2 h-4 w-4" />
              Share Document
            </DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>
      </div>
    </div>
  )
}
```

## Step 4: AI Sidebar Component

Create `src/components/document/ai-sidebar.tsx`:

```typescript
'use client'

import { useState, useEffect } from 'react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { Loader2, X, Sparkles, Lightbulb, PenTool, FileText, Wand2 } from 'lucide-react'
import { useAIAssistant } from '@/hooks/use-ai-assistant'

interface AISidebarProps {
  documentId?: string
  currentContent?: string
  onAIRequest?: (request: any) => void
  onClose?: () => void
}

export function AISidebar({ 
  documentId, 
  currentContent, 
  onAIRequest, 
  onClose 
}: AISidebarProps) {
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
        onAIRequest?.({
          type: 'generate',
          prompt,
          content: response.content
        })
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
        onAIRequest?.({
          type: 'improve',
          instructions,
          content: response.content
        })
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
        onAIRequest?.({
          type: 'suggest',
          suggestions: response.suggestions
        })
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
        onAIRequest?.({
          type: 'rewrite',
          style,
          content: response.content
        })
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
    <div className="ai-sidebar flex flex-col h-full">
      {/* Header */}
      <div className="p-4 border-b border-border">
        <div className="flex items-center justify-between">
          <h3 className="font-semibold flex items-center">
            <Sparkles className="mr-2 h-4 w-4" />
            AI Assistant
          </h3>
          <Button variant="ghost" size="sm" onClick={onClose}>
            <X className="h-4 w-4" />
          </Button>
        </div>
      </div>

      {/* Content */}
      <div className="flex-1 overflow-y-auto">
        <Tabs value={activeTab} onValueChange={setActiveTab} className="h-full flex flex-col">
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

          <div className="flex-1 p-4 space-y-4">
            {/* Generate Tab */}
            <TabsContent value="generate" className="space-y-4 mt-0">
              <div>
                <label className="text-sm font-medium">What would you like to create?</label>
                <Textarea
                  placeholder="Describe the content you want to generate..."
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
                        className="w-full justify-start text-left text-xs"
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
            <TabsContent value="improve" className="space-y-4 mt-0">
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
            <TabsContent value="suggest" className="space-y-4 mt-0">
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
            <TabsContent value="rewrite" className="space-y-4 mt-0">
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
      </div>

      {/* Results */}
      {results.length > 0 && (
        <div className="border-t border-border p-4 max-h-32 overflow-y-auto">
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

## Step 5: Editor Page Components

### Create Document Editor Page

Create `src/app/(dashboard)/editor/[documentId]/page.tsx`:

```typescript
import { notFound } from 'next/navigation'
import { createServerSupabaseClient } from '@/lib/supabase/server'
import { SuperDocEditor } from '@/components/document/superdoc-editor'
import { ProtectedRoute } from '@/components/auth/protected-route'

interface EditorPageProps {
  params: {
    documentId: string
  }
}

export default async function EditorPage({ params }: EditorPageProps) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    notFound()
  }

  // Handle 'new' document ID
  if (params.documentId === 'new') {
    return (
      <ProtectedRoute>
        <SuperDocEditor
          title="Untitled Document"
          user={{
            id: user.id,
            name: user.user_metadata?.full_name || user.email || '',
            email: user.email || ''
          }}
        />
      </ProtectedRoute>
    )
  }

  const { data: document, error } = await supabase
    .from('documents')
    .select('*')
    .eq('id', params.documentId)
    .eq('user_id', user.id)
    .single()

  if (error || !document) {
    notFound()
  }

  return (
    <ProtectedRoute>
      <SuperDocEditor
        documentId={document.id}
        initialContent={document.content || ''}
        title={document.title}
        user={{
          id: user.id,
          name: user.user_metadata?.full_name || user.email || '',
          email: user.email || ''
        }}
      />
    </ProtectedRoute>
  )
}
```

### Create Document View Page

Create `src/app/(dashboard)/documents/[documentId]/page.tsx`:

```typescript
import { notFound } from 'next/navigation'
import { createServerSupabaseClient } from '@/lib/supabase/server'
import { SuperDocEditor } from '@/components/document/superdoc-editor'
import { ProtectedRoute } from '@/components/auth/protected-route'

interface DocumentViewPageProps {
  params: {
    documentId: string
  }
}

export default async function DocumentViewPage({ params }: DocumentViewPageProps) {
  const supabase = createServerSupabaseClient()
  
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    notFound()
  }

  const { data: document, error } = await supabase
    .from('documents')
    .select('*')
    .eq('id', params.documentId)
    .eq('user_id', user.id)
    .single()

  if (error || !document) {
    notFound()
  }

  return (
    <ProtectedRoute>
      <SuperDocEditor
        documentId={document.id}
        initialContent={document.content || ''}
        title={document.title}
        user={{
          id: user.id,
          name: user.user_metadata?.full_name || user.email || '',
          email: user.email || ''
        }}
      />
    </ProtectedRoute>
  )
}
```

## Step 6: Custom Hooks for Editor

Create `src/hooks/use-editor.ts`:

```typescript
'use client'

import { useState, useEffect, useCallback } from 'react'
import { useDocument } from './use-documents'

interface UseEditorOptions {
  documentId?: string
  autoSave?: boolean
  autoSaveDelay?: number
}

export function useEditor(options: UseEditorOptions = {}) {
  const { documentId, autoSave = true, autoSaveDelay = 2000 } = options
  const { document, updateDocument, loading } = useDocument(documentId || '')
  
  const [content, setContent] = useState('')
  const [wordCount, setWordCount] = useState(0)
  const [isDirty, setIsDirty] = useState(false)
  const [lastSaved, setLastSaved] = useState<Date | null>(null)

  // Auto-save functionality
  useEffect(() => {
    if (!autoSave || !documentId || !isDirty) return

    const timeoutId = setTimeout(async () => {
      try {
        await updateDocument(documentId, {
          content,
          word_count: wordCount
        })
        setIsDirty(false)
        setLastSaved(new Date())
      } catch (error) {
        console.error('Auto-save failed:', error)
      }
    }, autoSaveDelay)

    return () => clearTimeout(timeoutId)
  }, [content, wordCount, isDirty, documentId, autoSave, autoSaveDelay, updateDocument])

  const updateContent = useCallback((newContent: string) => {
    setContent(newContent)
    setIsDirty(true)
    
    // Update word count
    const words = newContent.replace(/<[^>]*>/g, '').split(/\s+/).filter(word => word.length > 0)
    setWordCount(words.length)
  }, [])

  const saveDocument = useCallback(async () => {
    if (!documentId || !isDirty) return

    try {
      await updateDocument(documentId, {
        content,
        word_count: wordCount
      })
      setIsDirty(false)
      setLastSaved(new Date())
      return true
    } catch (error) {
      console.error('Manual save failed:', error)
      return false
    }
  }, [documentId, content, wordCount, isDirty, updateDocument])

  return {
    // State
    document,
    content,
    wordCount,
    isDirty,
    lastSaved,
    loading,
    
    // Actions
    updateContent,
    saveDocument,
    setIsDirty,
    
    // Computed
    hasUnsavedChanges: isDirty,
    isLoading: loading
  }
}
```

## Step 7: Editor Utils

Create `src/lib/editor/utils.ts`:

```typescript
// Editor utility functions

export function calculateWordCount(html: string): number {
  const text = html.replace(/<[^>]*>/g, '') // Remove HTML tags
  const words = text.split(/\s+/).filter(word => word.length > 0)
  return words.length
}

export function estimateReadingTime(wordCount: number, wordsPerMinute: number = 200): number {
  return Math.ceil(wordCount / wordsPerMinute)
}

export function extractTextFromHTML(html: string): string {
  return html.replace(/<[^>]*>/g, '')
}

export function formatFileSize(bytes: number): string {
  if (bytes === 0) return '0 Bytes'
  
  const k = 1024
  const sizes = ['Bytes', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

export function validateDocumentTitle(title: string): { valid: boolean; error?: string } {
  if (!title || title.trim().length === 0) {
    return { valid: false, error: 'Title is required' }
  }
  
  if (title.length > 255) {
    return { valid: false, error: 'Title must be less than 255 characters' }
  }
  
  return { valid: true }
}

export function generateDocumentSlug(title: string): string {
  return title
    .toLowerCase()
    .replace(/[^a-z0-9\s-]/g, '')
    .replace(/\s+/g, '-')
    .trim()
}

export function createDocumentPreview(content: string, maxLength: number = 200): string {
  const text = extractTextFromHTML(content)
  if (text.length <= maxLength) return text
  
  return text.substring(0, maxLength).trim() + '...'
}

// Export format helpers
export function getExportFilename(title: string, format: string): string {
  const cleanTitle = title.replace(/[^a-zA-Z0-9\s-]/g, '').trim()
  const slug = cleanTitle.toLowerCase().replace(/\s+/g, '-')
  return `${slug}.${format}`
}

export function getMimeType(format: string): string {
  const mimeTypes: Record<string, string> = {
    pdf: 'application/pdf',
    docx: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    html: 'text/html',
    txt: 'text/plain'
  }
  
  return mimeTypes[format] || 'application/octet-stream'
}
```

## Testing the Editor

### Test Basic Functionality

```typescript
// Test component
import { SuperDocEditor, EditorRef } from '@/components/document/superdoc-editor'

function TestEditor() {
  const editorRef = useRef<EditorRef>(null)
  
  const handleExport = async () => {
    try {
      const blob = await editorRef.current?.export({ format: 'pdf' })
      // Handle blob...
    } catch (error) {
      console.error('Export failed:', error)
    }
  }
  
  const handleInsertContent = () => {
    editorRef.current?.insertContent('Hello, World!')
  }
  
  return (
    <SuperDocEditor
      ref={editorRef}
      title="Test Document"
      user={{ id: 'user-id', name: 'Test User', email: 'test@example.com' }}
      onSave={(content, metadata) => {
        console.log('Document saved:', metadata)
      }}
    />
  )
}
```

### Test Auto-save

```typescript
import { useEditor } from '@/hooks/use-editor'

function EditorWithAutoSave({ documentId }: { documentId: string }) {
  const { content, isDirty, saveDocument } = useEditor({
    documentId,
    autoSave: true,
    autoSaveDelay: 3000
  })
  
  return (
    <div>
      <div>Content: {content}</div>
      <div>Is Dirty: {isDirty}</div>
      <button onClick={saveDocument}>Manual Save</button>
    </div>
  )
}
```

## Performance Considerations

1. **Lazy Loading**: Load SuperDoc only when needed
2. **Debounced Auto-save**: Prevent excessive save requests
3. **Content Optimization**: Store both HTML and plain text versions
4. **Memory Management**: Clean up event listeners properly
5. **SSR Handling**: Ensure proper client-side initialization

## Accessibility

1. **Keyboard Navigation**: Ensure all editor functions work with keyboard
2. **Screen Reader Support**: Add proper ARIA labels
3. **Focus Management**: Manage focus when switching between editor and sidebar
4. **Color Contrast**: Ensure sufficient contrast in editor UI

## Next Steps

Now that the document editor is implemented, proceed to:
1. [Export Functionality Guide](./07-export-functionality-guide.md)
2. [Security Implementation Guide](./08-security-guide.md)

This completes the document editor implementation using SuperDoc. You now have a fully functional, AI-integrated document editor with real-time saving, export capabilities, and comprehensive editing features.