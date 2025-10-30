# 🏗️ Enhanced AI Chat - Architecture Diagram

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          USER BROWSER                                    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                  AIChat.Enhanced.astro                          │    │
│  │                     (Main Component)                            │    │
│  │                                                                  │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │    │
│  │  │   History    │  │   Messages   │  │  Export      │         │    │
│  │  │   Sidebar    │  │   Display    │  │  Modal       │         │    │
│  │  │              │  │              │  │              │         │    │
│  │  │ • List       │  │ • User       │  │ • JSON       │         │    │
│  │  │ • Search     │  │ • AI         │  │ • TXT        │         │    │
│  │  │ • Click      │  │ • Streaming  │  │ • Markdown   │         │    │
│  │  │              │  │ • Copy       │  │ • HTML       │         │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘         │    │
│  │                                                                  │    │
│  │  ┌────────────────────────────────────────────────────┐        │    │
│  │  │           Client-side State Manager                 │        │    │
│  │  │                                                      │        │    │
│  │  │  • currentConversation: Conversation                │        │    │
│  │  │  • conversations: Conversation[]                    │        │    │
│  │  │  • history: ChatHistoryEntry[]                      │        │    │
│  │  │  • isProcessing: boolean                            │        │    │
│  │  └────────────────────────────────────────────────────┘        │    │
│  │                          ↕                                       │    │
│  │  ┌────────────────────────────────────────────────────┐        │    │
│  │  │           localStorage Integration                  │        │    │
│  │  │                                                      │        │    │
│  │  │  • mybonzo-ai-conversations                         │        │    │
│  │  │  • mybonzo-ai-current-conversation                  │        │    │
│  │  │  • mybonzo-ai-sidebar-state                         │        │    │
│  │  └────────────────────────────────────────────────────┘        │    │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↕
                            HTTP / Server-Sent Events
                                    ↕
┌─────────────────────────────────────────────────────────────────────────┐
│                     CLOUDFLARE WORKERS RUNTIME                           │
│                                                                          │
│  ┌──────────────────────────┐  ┌──────────────────────────┐            │
│  │  POST /api/ai/chat       │  │ POST /api/ai/chat-stream │            │
│  │  (Standard Mode)         │  │ (Streaming Mode)         │            │
│  │                          │  │                          │            │
│  │  1. Rate Limit Check     │  │  1. Rate Limit Check     │            │
│  │  2. Validate Input       │  │  2. Validate Input       │            │
│  │  3. Sanitize History     │  │  3. Sanitize History     │            │
│  │  4. Check KV Cache       │  │  4. Open SSE Stream      │            │
│  │  5. Call AI (if miss)    │  │  5. Call AI Streaming    │            │
│  │  6. Save to Cache        │  │  6. Forward Chunks       │            │
│  │  7. Return JSON          │  │  7. Close Stream         │            │
│  └──────────────────────────┘  └──────────────────────────┘            │
│                ↓                              ↓                          │
│  ┌────────────────────────────────────────────────────┐                │
│  │            Shared Services Layer                    │                │
│  │                                                      │                │
│  │  • rateLimiter: Map<IP, RateInfo>                   │                │
│  │  • sanitiseHistory(history)                         │                │
│  │  • buildSystemPrompt(model)                         │                │
│  │  • getClientIP(request)                             │                │
│  └────────────────────────────────────────────────────┘                │
│                          ↓                                               │
│  ┌────────────────────────────────────────────────────┐                │
│  │            Cloudflare Services                      │                │
│  │                                                      │                │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐         │                │
│  │  │   AI     │  │  CACHE   │  │ SESSION  │         │                │
│  │  │ Binding  │  │   (KV)   │  │   (KV)   │         │                │
│  │  │          │  │          │  │          │         │                │
│  │  │ 4 Models │  │ 1h TTL   │  │ Session  │         │                │
│  │  └──────────┘  └──────────┘  └──────────┘         │                │
│  └────────────────────────────────────────────────────┘                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                      CLOUDFLARE WORKERS AI                               │
│                                                                          │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐           │
│  │ Gemma 3 12B IT │  │ Qwen QWQ 32B   │  │ Phi-2          │           │
│  │ (Default)      │  │ (Reasoning)    │  │ (Fast)         │           │
│  └────────────────┘  └────────────────┘  └────────────────┘           │
│                                                                          │
│  ┌────────────────┐                                                     │
│  │ OpenChat 3.5   │                                                     │
│  │ (Conversational│                                                     │
│  └────────────────┘                                                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Diagrams

### Standard Mode (Non-streaming)

```
┌─────────┐
│  USER   │
└────┬────┘
     │ 1. Types message
     ↓
┌────────────────┐
│ Input Textarea │
└────┬───────────┘
     │ 2. Submit form
     ↓
┌─────────────────────┐
│ JavaScript Handler  │
│ • Validate input    │
│ • Add to display    │
│ • Update history    │
└─────┬───────────────┘
     │ 3. POST /api/ai/chat
     ↓
┌──────────────────────┐
│ API Endpoint         │
│ • Rate limit check   │
│ • Sanitize history   │
└─────┬────────────────┘
     │ 4. Check cache
     ↓
┌─────────────────┐    Cache Hit
│ Cloudflare KV   │───────────┐
│ CACHE namespace │           │
└─────┬───────────┘           │
     │ Cache Miss            │
     │ 5. Call AI            │
     ↓                        │
┌──────────────────┐          │
│ Cloudflare AI    │          │
│ • Run model      │          │
│ • Generate text  │          │
└─────┬────────────┘          │
     │ 6. Response           │
     ↓                        │
┌──────────────────┐          │
│ Save to cache    │          │
│ TTL: 1 hour      │          │
└─────┬────────────┘          │
     │                        │
     └────────┬───────────────┘
              │ 7. Return JSON
              ↓
┌─────────────────────┐
│ Client receives     │
│ • Parse response    │
│ • Render markdown   │
│ • Add to display    │
│ • Save to localStorage │
└──────────┬──────────┘
           │ 8. User sees response
           ↓
      ┌─────────┐
      │  USER   │
      └─────────┘
```

### Streaming Mode (SSE)

```
┌─────────┐
│  USER   │
└────┬────┘
     │ 1. Types message + Streaming enabled
     ↓
┌────────────────┐
│ Input Textarea │
└────┬───────────┘
     │ 2. Submit form
     ↓
┌─────────────────────┐
│ JavaScript Handler  │
│ • Validate input    │
│ • Add to display    │
│ • Update history    │
└─────┬───────────────┘
     │ 3. POST /api/ai/chat-stream
     ↓
┌──────────────────────┐
│ Streaming Endpoint   │
│ • Rate limit check   │
│ • Sanitize history   │
│ • Open SSE stream    │
└─────┬────────────────┘
     │ 4. Call AI with stream=true
     ↓
┌──────────────────────┐
│ Cloudflare AI        │
│ • Run model          │
│ • Stream tokens      │
└─────┬────────────────┘
     │
     │ ┌─ Continuous stream ─┐
     │ │                     │
     ↓ ↓                     ↑
┌─────────────────────┐      │
│ SSE Stream          │      │
│ data: {chunk: "W "} │──────┘
│ data: {chunk: "2025"}│
│ data: {chunk: " "}  │
│ data: {done: true}  │
└─────┬───────────────┘
     │
     │ Each chunk forwarded to client
     ↓
┌─────────────────────┐
│ Client EventSource  │
│ • Receive chunk     │
│ • Append to buffer  │
└─────┬───────────────┘
     │
     ↓ For each chunk
┌─────────────────────┐
│ Progressive Render  │
│ • Update DOM        │
│ • Render markdown   │
│ • Scroll to bottom  │
└──────────┬──────────┘
           │
           ↓ On stream complete
┌─────────────────────┐
│ Finalize Message    │
│ • Add copy button   │
│ • Remove "streaming"│
│ • Save to localStorage │
└──────────┬──────────┘
           │ User sees complete response
           ↓
      ┌─────────┐
      │  USER   │
      └─────────┘
```

---

## Component Interaction Flow

```
┌──────────────────────────────────────────────────────────────┐
│                     User Actions                              │
└───┬──────────┬──────────┬──────────┬──────────┬─────────────┘
    │          │          │          │          │
    │          │          │          │          │
    ↓          ↓          ↓          ↓          ↓
┌───────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌────────┐
│ Send  │  │Toggle│  │Search│  │Export│  │Bookmark│
│Message│  │Model │  │ Conv │  │      │  │        │
└───┬───┘  └───┬──┘  └───┬──┘  └───┬──┘  └───┬────┘
    │          │          │          │          │
    └──────────┴──────────┴──────────┴──────────┘
                          │
                          ↓
            ┌─────────────────────────┐
            │   Event Handlers         │
            │   (TypeScript)           │
            └─────────────┬────────────┘
                          │
                          ↓
            ┌─────────────────────────┐
            │   State Updates          │
            │   • currentConversation  │
            │   • conversations[]      │
            │   • history[]            │
            └─────────────┬────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ↓                 ↓                 ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ localStorage │  │     API      │  │     UI       │
│   Write      │  │   Request    │  │   Update     │
└──────────────┘  └──────────────┘  └──────────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                          ↓
                ┌─────────────────┐
                │   User sees      │
                │   updated UI     │
                └─────────────────┘
```

---

## localStorage Schema

```
localStorage
│
├─ "mybonzo-ai-conversations"
│  │
│  └─ Array<Conversation>
│     │
│     ├─ Conversation #1
│     │  ├─ id: "conv-1706544000000"
│     │  ├─ title: "Jak działa TypeScript?"
│     │  ├─ messages: [
│     │  │   {
│     │  │     role: "user",
│     │  │     content: "Jak działa TypeScript?",
│     │  │     timestamp: 1706544001000
│     │  │   },
│     │  │   {
│     │  │     role: "ai",
│     │  │     content: "TypeScript to...",
│     │  │     timestamp: 1706544003000,
│     │  │     cached: false
│     │  │   }
│     │  │ ]
│     │  ├─ model: "@cf/google/gemma-3-12b-it"
│     │  ├─ createdAt: 1706544000000
│     │  ├─ updatedAt: 1706544010000
│     │  └─ bookmarked: true
│     │
│     ├─ Conversation #2
│     │  └─ ...
│     │
│     └─ Conversation #N
│        └─ ...
│
├─ "mybonzo-ai-current-conversation"
│  └─ "conv-1706544000000"  (ID aktywnej konwersacji)
│
└─ "mybonzo-ai-sidebar-state"
   └─ "open"  (stan sidebaru: "open" | "closed")
```

---

## Export Flow

```
┌─────────┐
│  USER   │
│ Clicks  │
│ Export  │
└────┬────┘
     │
     ↓
┌────────────────────┐
│ Show Export Modal  │
│ • JSON             │
│ • TXT              │
│ • Markdown         │
│ • HTML             │
└────┬───────────────┘
     │
     ↓ User selects format
┌────────────────────┐
│ Export Engine      │
└────┬───────────────┘
     │
     ├─ JSON ──────→ JSON.stringify(conversation)
     │
     ├─ TXT ───────→ formatConversationAsText(conversation)
     │               └─ Plain text with timestamps
     │
     ├─ Markdown ──→ formatConversationAsMarkdown(conversation)
     │               └─ Markdown with formatting
     │
     └─ HTML ──────→ formatConversationAsHTML(conversation)
                     └─ Standalone HTML with CSS
     │
     ↓ Generate blob
┌────────────────────┐
│ Download File      │
│ • Create Blob      │
│ • Generate URL     │
│ • Trigger download │
│ • Cleanup          │
└────┬───────────────┘
     │
     ↓
┌─────────┐
│  FILE   │
│ Saved   │
└─────────┘
```

---

## Rate Limiting Architecture

```
┌───────────────────────────────────────────┐
│          Request arrives                   │
└───────────────┬───────────────────────────┘
                │
                ↓
┌───────────────────────────────────────────┐
│      Extract Client IP                     │
│      • cf-connecting-ip                    │
│      • x-forwarded-for                     │
└───────────────┬───────────────────────────┘
                │
                ↓
┌───────────────────────────────────────────┐
│   Check rateLimiter Map<IP, RateInfo>     │
└───────────────┬───────────────────────────┘
                │
        ┌───────┴───────┐
        │               │
        ↓               ↓
┌──────────────┐  ┌──────────────┐
│  New IP      │  │ Existing IP  │
│  (no record) │  │ (has record) │
└──────┬───────┘  └──────┬───────┘
       │                 │
       ↓                 ↓
┌──────────────┐  ┌─────────────────┐
│ Create       │  │ Check timestamp │
│ new record   │  └──────┬──────────┘
│              │         │
│ count: 1     │  ┌──────┴──────┐
│ resetTime:   │  │             │
│ now + 60s    │  ↓             ↓
└──────┬───────┘  Past     Within
       │          reset    window
       │          time
       │            │        │
       │            ↓        ↓
       │       ┌─────────┐  Check
       │       │ Reset   │  count
       │       │ record  │    │
       │       └────┬────┘    │
       │            │         │
       └────────────┴─────────┘
                    │
            ┌───────┴───────┐
            │               │
            ↓               ↓
    ┌──────────────┐  ┌──────────────┐
    │ count < 15   │  │ count >= 15  │
    │ ALLOW        │  │ DENY         │
    └──────┬───────┘  └──────┬───────┘
           │                 │
           │                 ↓
           │         ┌──────────────┐
           │         │ Return 429   │
           │         │ Retry-After  │
           │         └──────────────┘
           │
           ↓
    ┌──────────────┐
    │ Increment    │
    │ count        │
    └──────┬───────┘
           │
           ↓
    ┌──────────────┐
    │ Process      │
    │ request      │
    └──────────────┘
```

---

## Security Layers

```
┌────────────────────────────────────────────────────┐
│                   User Input                        │
└─────────────────────┬──────────────────────────────┘
                      │
                      ↓
┌────────────────────────────────────────────────────┐
│            Layer 1: Client Validation               │
│            • Max length check (800-1000 chars)      │
│            • Required field validation              │
│            • Type checking                          │
└─────────────────────┬──────────────────────────────┘
                      │
                      ↓
┌────────────────────────────────────────────────────┐
│            Layer 2: HTML Escaping                   │
│            • escapeHtml() function                  │
│            • Prevent XSS in user messages           │
│            • Safe innerHTML usage                   │
└─────────────────────┬──────────────────────────────┘
                      │
                      ↓
┌────────────────────────────────────────────────────┐
│            Layer 3: API Validation                  │
│            • Type checking                          │
│            • Length limits (2000 chars)             │
│            • History sanitization                   │
└─────────────────────┬──────────────────────────────┘
                      │
                      ↓
┌────────────────────────────────────────────────────┐
│            Layer 4: Rate Limiting                   │
│            • IP-based throttling                    │
│            • 15 requests per 60 seconds             │
│            • 429 response with Retry-After          │
└─────────────────────┬──────────────────────────────┘
                      │
                      ↓
┌────────────────────────────────────────────────────┐
│            Layer 5: AI Inference                    │
│            • Model validation                       │
│            • Parameter bounds checking              │
│            • Error handling                         │
└─────────────────────┬──────────────────────────────┘
                      │
                      ↓
┌────────────────────────────────────────────────────┐
│            Layer 6: Output Sanitization             │
│            • Markdown rendering with safe HTML      │
│            • Link target="_blank" with noopener     │
│            • No script execution in exports         │
└────────────────────────────────────────────────────┘
```

---

## Mobile Responsive Layout

```
Desktop (>1024px):
┌─────────────────────────────────────────────┐
│  ┌─────────┐  ┌──────────────────────────┐ │
│  │         │  │                          │ │
│  │ Sidebar │  │     Main Chat            │ │
│  │         │  │                          │ │
│  │ • Conv1 │  │  ┌────────────────────┐  │ │
│  │ • Conv2 │  │  │   Messages Area    │  │ │
│  │ • Conv3 │  │  │                    │  │ │
│  │ • Search│  │  │   • User messages  │  │ │
│  │         │  │  │   • AI responses   │  │ │
│  │ (300px) │  │  └────────────────────┘  │ │
│  │         │  │                          │ │
│  │         │  │  ┌────────────────────┐  │ │
│  │         │  │  │   Input Form       │  │ │
│  │         │  │  └────────────────────┘  │ │
│  └─────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────┘

Mobile (<768px):
┌──────────────────────────┐
│ ☰ Toggle    MyBonzo AI   │ ← Header
├──────────────────────────┤
│                          │
│  ┌────────────────────┐  │
│  │   Messages Area    │  │
│  │                    │  │
│  │   • User messages  │  │
│  │   • AI responses   │  │
│  │                    │  │
│  └────────────────────┘  │
│                          │
│  ┌────────────────────┐  │
│  │   Input Form       │  │
│  └────────────────────┘  │
│                          │
└──────────────────────────┘

When sidebar toggled:
┌──────────────────────────┐
│ Sidebar (overlay)        │
│                          │
│ • Conversation 1         │
│ • Conversation 2         │
│ • Conversation 3         │
│ • Search                 │
│                          │
│ [Tap outside to close]   │
└──────────────────────────┘
     ↓ (z-index: 1000)
┌──────────────────────────┐
│ Main chat (dimmed)       │
│                          │
│ (background overlay)     │
│                          │
└──────────────────────────┘
```

---

## Error Handling Flow

```
┌──────────────┐
│    Error     │
│   Occurs     │
└──────┬───────┘
       │
       ↓
┌──────────────────────────────┐
│  Error Type Detection         │
└──────┬───────────────────────┘
       │
       ├─ Validation Error ──→ Show inline message
       │                       • Red border on input
       │                       • Status text update
       │                       • No API call
       │
       ├─ Rate Limit Error ─→ 429 Response
       │                       • Show retry time
       │                       • Disable input temporarily
       │                       • Auto-retry after timeout
       │
       ├─ Network Error ────→ Retry Logic
       │                       • Show error message
       │                       • Offer manual retry
       │                       • Preserve user input
       │
       ├─ AI API Error ─────→ Graceful Degradation
       │                       • Show error in chat
       │                       • Suggest model switch
       │                       • Don't save to history
       │
       └─ Streaming Error ──→ Fallback to Standard
                               • Close SSE connection
                               • Retry with POST
                               • Show standard loading
       │
       ↓
┌──────────────────────────────┐
│  User Notification            │
│  • Error message (Polish)     │
│  • Actionable suggestions     │
│  • Recovery options           │
└───────────────────────────────┘
```

---

## Performance Optimization Layers

```
┌────────────────────────────────────────────┐
│         Layer 1: Client-side Cache          │
│         • localStorage (instant load)       │
│         • No API calls for local data       │
└─────────────────┬──────────────────────────┘
                  │
                  ↓
┌────────────────────────────────────────────┐
│         Layer 2: Server Cache (KV)          │
│         • 1-hour TTL                        │
│         • Identical requests cached         │
│         • Base64 cache key                  │
└─────────────────┬──────────────────────────┘
                  │
                  ↓
┌────────────────────────────────────────────┐
│         Layer 3: Cloudflare CDN             │
│         • Edge caching                      │
│         • Static assets                     │
│         • Global distribution               │
└─────────────────┬──────────────────────────┘
                  │
                  ↓
┌────────────────────────────────────────────┐
│         Layer 4: Code Optimization          │
│         • Debounced search (300ms)          │
│         • Lazy rendering                    │
│         • Minimal re-renders                │
└─────────────────┬──────────────────────────┘
                  │
                  ↓
┌────────────────────────────────────────────┐
│         Layer 5: Bundle Optimization        │
│         • Tree shaking                      │
│         • Code splitting                    │
│         • Minification                      │
└────────────────────────────────────────────┘
```

---

**Architecture Version:** 1.0.0
**Last Updated:** 2025-01-30
**Diagram Type:** ASCII Text Architecture
