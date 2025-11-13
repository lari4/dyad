# Документация Пайплайнов Агента Dyad

Этот документ описывает все возможные схемы работы AI агента в Dyad, включая пайплайны обработки сообщений, вызовов промтов и интеграций. Каждый пайплайн представлен в виде ASCII схемы с подробным описанием потока данных.

## Оглавление

1. [Основной Chat Pipeline](#основной-chat-pipeline)
2. [Build Mode Pipeline](#build-mode-pipeline)
3. [Ask Mode Pipeline](#ask-mode-pipeline)
4. [Agent Mode Pipeline](#agent-mode-pipeline)
5. [Security Review Pipeline](#security-review-pipeline)
6. [Supabase Integration Pipeline](#supabase-integration-pipeline)
7. [TypeScript Error Fix Pipeline](#typescript-error-fix-pipeline)
8. [Chat Summarization Pipeline](#chat-summarization-pipeline)

---

## Основной Chat Pipeline

**Файл:** `src/ipc/handlers/chat_stream_handlers.ts`

**Entry Point:** `ipcMain.handle("chat:stream")`

Это основной пайплайн обработки всех сообщений в Dyad. Обрабатывает входящие запросы пользователя, собирает контекст, генерирует промты, стримит ответ AI и обрабатывает результаты.

### ASCII Схема

```
┌─────────────────────────────────────────────────────────────────────┐
│                      USER SENDS MESSAGE                              │
│                    (chat:stream IPC call)                           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   INITIALIZATION PHASE                               │
├─────────────────────────────────────────────────────────────────────┤
│ 1. Create AbortController for stream cancellation                   │
│ 2. Fetch chat from database (with messages + app info)             │
│ 3. Handle redo option (delete last user/assistant messages)        │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   ATTACHMENTS PROCESSING                             │
├─────────────────────────────────────────────────────────────────────┤
│ FOR EACH attachment:                                                 │
│   • Generate unique filename (MD5 hash)                             │
│   • Save to temp directory                                           │
│   • Type: upload-to-codebase?                                       │
│     YES → Store mapping with DYAD_ATTACHMENT_N id                   │
│     NO  → Add to chat context (read text files)                     │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  USER PROMPT ENRICHMENT                              │
├─────────────────────────────────────────────────────────────────────┤
│ 1. Add attachment info to prompt                                     │
│ 2. Expand @prompt:<id> references                                   │
│    • Query prompts table from DB                                     │
│    • Replace @prompt:123 with actual content                        │
│ 3. Add selected component context                                    │
│    • Read file content                                               │
│    • Extract lines around selection                                  │
│    • Add "// <-- EDIT HERE" marker                                  │
│ 4. Save enriched user message to DB                                 │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 CREATE PLACEHOLDER MESSAGE                           │
├─────────────────────────────────────────────────────────────────────┤
│ • Insert empty assistant message to DB                              │
│ • Generate dyadRequestId (if Pro enabled)                           │
│ • Store current git commit hash                                      │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  TEST MODE CHECK                                     │
├─────────────────────────────────────────────────────────────────────┤
│ Check for [dyad-qa=<key>] in prompt?                               │
│   YES → Return canned test response                                 │
│   NO  → Continue to normal flow                                      │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│              LOAD CONFIGURATION & SETTINGS                           │
├─────────────────────────────────────────────────────────────────────┤
│ • Read user settings (model, provider, keys)                        │
│ • Read AI_RULES.md or use DEFAULT_AI_RULES                          │
│ • Check Turbo Edits v2 enabled                                       │
│ • Get max tokens and temperature                                     │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 DETERMINE CHAT INTENT                                │
├─────────────────────────────────────────────────────────────────────┤
│ Check prompt for special intents:                                    │
│   • Contains "/security-review"? → Security Review Mode             │
│   • Contains "Summarize from chat-id="? → Summarization Mode       │
│   • Otherwise → Use chat.mode (build/ask/agent)                     │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                 ┌───────────┴──────────┬─────────────────────┐
                 │                      │                     │
                 ▼                      ▼                     ▼
        ┌──────────────┐      ┌─────────────────┐   ┌────────────────┐
        │ SECURITY     │      │ SUMMARIZATION   │   │ NORMAL CHAT    │
        │ REVIEW       │      │ MODE            │   │ MODE           │
        └──────┬───────┘      └────────┬────────┘   └───────┬────────┘
               │                       │                    │
               └───────────────────────┴────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   BUILD SYSTEM PROMPT                                │
├─────────────────────────────────────────────────────────────────────┤
│ constructSystemPrompt({                                              │
│   aiRules,        ← from AI_RULES.md                                │
│   chatMode,       ← build/ask/agent/security/summarize             │
│   enableTurboEditsV2  ← from settings                               │
│ })                                                                   │
│                                                                      │
│ ASSEMBLED SYSTEM PROMPT:                                             │
│ ┌────────────────────────────────────────────────────────┐          │
│ │ 1. THINKING_PROMPT (optional)                          │          │
│ │ 2. Mode-specific system prompt:                        │          │
│ │    - BUILD_SYSTEM_PROMPT                               │          │
│ │    - ASK_MODE_SYSTEM_PROMPT                            │          │
│ │    - AGENT_MODE_SYSTEM_PROMPT                          │          │
│ │    - SECURITY_REVIEW_SYSTEM_PROMPT                     │          │
│ │    - SUMMARIZE_CHAT_SYSTEM_PROMPT                      │          │
│ │ 3. [[AI_RULES]] replacement                            │          │
│ │ 4. TURBO_EDITS_V2_SYSTEM_PROMPT (if enabled)           │          │
│ └────────────────────────────────────────────────────────┘          │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│              ADD INTEGRATION-SPECIFIC PROMPTS                        │
├─────────────────────────────────────────────────────────────────────┤
│ IF Supabase connected:                                               │
│   • Add SUPABASE_AVAILABLE_SYSTEM_PROMPT                            │
│   • Add dynamic Supabase context:                                    │
│     - Project ID, anon key                                           │
│     - Database schema (query information_schema)                    │
│     - Available secrets                                              │
│ ELSE IF request needs Supabase:                                      │
│   • Add SUPABASE_NOT_AVAILABLE_SYSTEM_PROMPT                        │
│                                                                      │
│ IF SECURITY_RULES.md exists:                                         │
│   • Append file content to system prompt                            │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  BUILD USER MESSAGES ARRAY                           │
├─────────────────────────────────────────────────────────────────────┤
│ STEP 1: Add Codebase Context (first user message)                   │
│   • extractCodebase() - get all project files                       │
│   • createCodebasePrompt() - format as message                      │
│   • For versioned context: getVersionedFiles()                      │
│                                                                      │
│ STEP 2: Add Referenced Apps Context (@app:)                         │
│   • Parse @app: mentions from user prompt                           │
│   • extractMentionedAppsCodebases()                                 │
│   • createOtherAppsCodebasePrompt() (READ-ONLY)                     │
│                                                                      │
│ STEP 3: Add Chat History                                            │
│   • Get last N messages (MAX_CHAT_TURNS_IN_CONTEXT)                │
│   • For each message:                                                │
│     - Format role (user/assistant)                                   │
│     - Handle multimodal content (text + images)                     │
│     - For image attachments: convert to ImagePart                   │
│                                                                      │
│ STEP 4: Add Current User Message                                    │
│   • Already enriched with:                                           │
│     - Attachments info                                               │
│     - @prompt: expansions                                            │
│     - Selected component context                                     │
│   • Handle image attachments (multimodal)                           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   SETUP MCP TOOLS (if enabled)                       │
├─────────────────────────────────────────────────────────────────────┤
│ IF MCP servers configured:                                           │
│   • Query mcpServers table from DB                                   │
│   • mcpManager.ensureServer(server) for each                        │
│   • Collect all available tools from MCP servers                    │
│   • Wrap each tool with consent requirement:                        │
│     - requireMcpToolConsent(tool)                                    │
│     - User approval needed before execution                         │
│   • Add tools to toolSet                                             │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    GET MODEL CLIENT                                  │
├─────────────────────────────────────────────────────────────────────┤
│ getModelClient(settings) → ModelClient                              │
│   Supports:                                                          │
│   • Anthropic (Claude)                                               │
│   • OpenAI (GPT-4, etc.)                                             │
│   • Google (Gemini)                                                  │
│   • Local models (Ollama, LM Studio)                                │
│   • Dyad Pro custom endpoint                                         │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  START AI STREAMING                                  │
├─────────────────────────────────────────────────────────────────────┤
│ streamText({                                                         │
│   model: modelClient,                                                │
│   system: systemPrompt,                                              │
│   messages: messagesArray,                                           │
│   tools: toolSet (MCP tools),                                        │
│   maxTokens,                                                         │
│   temperature,                                                       │
│   abortSignal                                                        │
│ })                                                                   │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│               PROCESS STREAMING RESPONSE                             │
├─────────────────────────────────────────────────────────────────────┤
│ FOR EACH chunk in stream:                                            │
│                                                                      │
│   IF chunk is text:                                                  │
│     • Append to fullResponse                                         │
│     • Send to frontend via safeSend()                               │
│                                                                      │
│   IF chunk is tool call:                                             │
│     • Extract tool name and args                                     │
│     • Execute MCP tool (after user consent)                         │
│     • Send tool result back to AI                                    │
│                                                                      │
│   IF chunk is finish:                                                │
│     • Get finish reason (stop/tool-calls/etc)                       │
│     • Break loop                                                     │
│                                                                      │
│ Store partialResponses for cancellation recovery                    │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                CLEAN & PROCESS RESPONSE                              │
├─────────────────────────────────────────────────────────────────────┤
│ 1. cleanFullResponse(fullResponse)                                   │
│    • Remove markdown code blocks (```)                              │
│    • Clean up malformed XML tags                                     │
│                                                                      │
│ 2. processFullResponseActions()                                      │
│    • Parse <dyad-write> tags                                        │
│    • Parse <dyad-rename> tags                                       │
│    • Parse <dyad-delete> tags                                       │
│    • Parse <dyad-add-dependency> tags                               │
│    • Parse <dyad-execute-sql> tags (Supabase)                       │
│    • Parse <dyad-search-replace> tags (Turbo Edits)                │
│    • Parse <dyad-security-finding> tags                             │
│    • Extract file operations and commands                           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  SAVE TO DATABASE                                    │
├─────────────────────────────────────────────────────────────────────┤
│ • Update assistant message in DB                                     │
│   - Set content = cleanedResponse                                    │
│   - Set requestId (if Pro)                                           │
│   - Set sourceCommitHash                                             │
│                                                                      │
│ • Update chat table                                                  │
│   - Set updatedAt = NOW()                                            │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   EXECUTE ACTIONS                                    │
├─────────────────────────────────────────────────────────────────────┤
│ Send parsed actions to frontend:                                     │
│ • File writes → update virtual filesystem                           │
│ • File renames → update file system                                 │
│ • File deletes → remove from filesystem                             │
│ • Dependencies → trigger npm install                                │
│ • SQL commands → execute on Supabase                                │
│ • Security findings → display in UI                                 │
│                                                                      │
│ Frontend processes actions and updates UI                           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    CHECK FOR ERRORS                                  │
├─────────────────────────────────────────────────────────────────────┤
│ IF TypeScript errors detected:                                       │
│   • generateProblemReport()                                          │
│   • createProblemFixPrompt(errors)                                  │
│   • Automatically trigger fix cycle (optional)                      │
│                                                                      │
│ IF Build errors:                                                     │
│   • Report to user                                                   │
│   • Suggest fixes                                                    │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      CLEANUP                                         │
├─────────────────────────────────────────────────────────────────────┤
│ • Remove from activeStreams map                                      │
│ • Delete temporary attachment files                                  │
│ • Clear file uploads state                                           │
│ • Send completion signal to frontend                                │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
                     ┌───────────────┐
                     │   COMPLETE    │
                     └───────────────┘
```

### Данные, передаваемые между этапами

**ChatStreamParams (входные):**
```typescript
{
  chatId: number,
  prompt: string,
  attachments?: Array<{
    name: string,
    type: string,
    data: string,  // base64
    attachmentType: "chat-context" | "upload-to-codebase"
  }>,
  selectedComponent?: {
    name: string,
    relativePath: string,
    lineNumber: number
  },
  redo?: boolean
}
```

**System Prompt (собранный):**
```typescript
string = THINKING_PROMPT? +
         BASE_SYSTEM_PROMPT +
         AI_RULES +
         TURBO_EDITS? +
         SUPABASE_CONTEXT? +
         SECURITY_RULES?
```

**Messages Array:**
```typescript
Array<{
  role: "user" | "assistant",
  content: string | Array<TextPart | ImagePart>
}>
```

**Parsed Actions (из ответа AI):**
```typescript
{
  fileWrites: Array<{ path: string, content: string, description: string }>,
  fileRenames: Array<{ from: string, to: string }>,
  fileDeletes: Array<{ path: string }>,
  dependencies: Array<{ packages: string[] }>,
  sqlCommands: Array<{ sql: string, description: string }>,
  searchReplaces: Array<{ path: string, operations: SearchReplaceOp[] }>,
  securityFindings: Array<{ title: string, level: string, content: string }>
}
```

**ChatResponseEnd (выходные):**
```typescript
{
  type: "end",
  fullResponse: string,
  messageId: number,
  requestId?: string
}
```

---
