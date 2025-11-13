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

## Build Mode Pipeline

**Назначение:** Режим активной разработки, где AI создает и модифицирует код в реальном времени.

**System Prompt:** `BUILD_SYSTEM_PROMPT` + `AI_RULES` + `TURBO_EDITS_V2_SYSTEM_PROMPT`?

### ASCII Схема

```
┌─────────────────────────────────────────────────────────────────────┐
│            BUILD MODE - CODE GENERATION PIPELINE                     │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 USER REQUEST (Build Mode)                            │
├─────────────────────────────────────────────────────────────────────┤
│ Examples:                                                            │
│ • "Add dark mode toggle"                                             │
│ • "Create a login page with email/password"                         │
│ • "Fix the broken navigation menu"                                  │
│ • "Refactor the UserProfile component"                              │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│             SYSTEM PROMPT COMPOSITION                                │
├─────────────────────────────────────────────────────────────────────┤
│ THINKING_PROMPT (optional)                                           │
│ + BUILD_SYSTEM_PREFIX                                                │
│   "You are Dyad, an AI editor..."                                   │
│   - Commands: rebuild, restart, refresh                             │
│   - Guidelines for code changes                                      │
│   - Examples of dyad-write, dyad-rename, dyad-delete               │
│ + AI_RULES (from AI_RULES.md or DEFAULT)                            │
│   - Tech stack (React, TypeScript, etc.)                            │
│   - File structure (src/pages, src/components)                      │
│   - Package info (shadcn/ui, lucide-react)                          │
│ + BUILD_SYSTEM_POSTFIX                                               │
│   "NEVER use markdown code blocks"                                  │
│   "ONLY use <dyad-write> tags"                                      │
│ + TURBO_EDITS_V2_SYSTEM_PROMPT (if enabled)                         │
│   "Use <dyad-search-replace> for surgical edits"                   │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│              MESSAGES WITH CODE CONTEXT                              │
├─────────────────────────────────────────────────────────────────────┤
│ [System Message]                                                     │
│   → Full system prompt from above                                   │
│                                                                      │
│ [User Message 1 - Codebase Context]                                 │
│   "This is my codebase:                                              │
│    - src/App.tsx (entry point)                                       │
│    - src/pages/Index.tsx (main page)                                │
│    - src/components/Button.tsx                                       │
│    - package.json (dependencies)                                     │
│    [... all project files with content]"                            │
│                                                                      │
│ [Previous messages from history]                                     │
│   User: "Create a todo list component"                              │
│   Assistant: <dyad-write path="...">...</dyad-write>                │
│                                                                      │
│ [Current User Request]                                               │
│   "Add dark mode toggle"                                             │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   AI PROCESSING                                      │
├─────────────────────────────────────────────────────────────────────┤
│ AI uses THINKING_PROMPT to plan:                                     │
│ <think>                                                              │
│ • **Identify what needs to be changed**                             │
│   - Need to add dark mode state management                          │
│   - Create toggle component                                          │
│   - Update theme context                                             │
│                                                                      │
│ • **Plan the approach**                                              │
│   - **File #1**: Create DarkModeToggle component                    │
│   - **File #2**: Update ThemeContext with dark mode state          │
│   - **File #3**: Update App.tsx to use dark mode                   │
│                                                                      │
│ • **Consider best practices**                                        │
│   - Use localStorage to persist preference                          │
│   - Follow existing project structure                               │
│   - Keep component small and focused                                │
│ </think>                                                             │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│              AI GENERATES RESPONSE WITH TAGS                         │
├─────────────────────────────────────────────────────────────────────┤
│ "I'll add a dark mode toggle to your app..."                        │
│                                                                      │
│ <dyad-write path="src/components/DarkModeToggle.tsx"                │
│              description="Creating dark mode toggle component">     │
│ import { Moon, Sun } from 'lucide-react';                           │
│ import { useTheme } from '../contexts/ThemeContext';                │
│                                                                      │
│ export function DarkModeToggle() {                                   │
│   const { theme, toggleTheme } = useTheme();                        │
│   return (                                                           │
│     <button onClick={toggleTheme}>                                  │
│       {theme === 'dark' ? <Sun /> : <Moon />}                       │
│     </button>                                                        │
│   );                                                                 │
│ }                                                                    │
│ </dyad-write>                                                        │
│                                                                      │
│ <dyad-search-replace path="src/App.tsx"                             │
│                       description="Adding dark mode toggle">        │
│ <<<<<<< SEARCH                                                       │
│ import { Navbar } from './components/Navbar';                       │
│ =======                                                              │
│ import { Navbar } from './components/Navbar';                       │
│ import { DarkModeToggle } from './components/DarkModeToggle';       │
│ >>>>>>> REPLACE                                                      │
│ </dyad-search-replace>                                               │
│                                                                      │
│ <dyad-chat-summary>Adding dark mode toggle</dyad-chat-summary>      │
│                                                                      │
│ "Dark mode toggle has been added to your app..."                    │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   PARSE & EXECUTE ACTIONS                            │
├─────────────────────────────────────────────────────────────────────┤
│ processFullResponseActions() extracts:                              │
│                                                                      │
│ fileWrites: [                                                        │
│   {                                                                  │
│     path: "src/components/DarkModeToggle.tsx",                      │
│     content: "import { Moon, Sun }...",                             │
│     description: "Creating dark mode toggle component"             │
│   }                                                                  │
│ ]                                                                    │
│                                                                      │
│ searchReplaces: [                                                    │
│   {                                                                  │
│     path: "src/App.tsx",                                            │
│     operations: [{                                                   │
│       search: "import { Navbar }...",                               │
│       replace: "import { Navbar }...\nimport { DarkModeToggle }..." │
│     }],                                                              │
│     description: "Adding dark mode toggle"                          │
│   }                                                                  │
│ ]                                                                    │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  APPLY CHANGES TO CODEBASE                           │
├─────────────────────────────────────────────────────────────────────┤
│ Frontend receives actions and:                                       │
│ 1. Updates VirtualFileSystem with new files                         │
│ 2. Applies search-replace operations                                │
│ 3. Triggers hot reload if dev server running                        │
│ 4. Shows visual feedback in UI                                       │
│                                                                      │
│ User sees live preview update immediately                           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│               CHECK FOR TYPESCRIPT ERRORS                            │
├─────────────────────────────────────────────────────────────────────┤
│ IF TypeScript errors detected:                                       │
│   → Generate problem report                                          │
│   → Create fix prompt                                                │
│   → Optionally auto-trigger fix cycle                               │
│                                                                      │
│ IF no errors:                                                        │
│   → Success! Changes applied                                         │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
                        ┌────────┐
                        │  DONE  │
                        └────────┘
```

### Ключевые особенности Build Mode

**1. Полная функциональность:**
- NO placeholders, NO TODO comments
- Complete, working code
- All imports resolved

**2. Immediate component creation:**
- New file for every component/hook
- Components ≤ 100 lines
- Small, focused files

**3. Запрещенные действия:**
- ❌ Markdown code blocks (\`\`\`)
- ❌ Partial implementations
- ❌ Shell commands for user to run
- ❌ Manual file operations

**4. Разрешенные теги:**
- ✅ `<dyad-write>` - создание/обновление файлов
- ✅ `<dyad-rename>` - переименование файлов
- ✅ `<dyad-delete>` - удаление файлов
- ✅ `<dyad-add-dependency>` - установка пакетов
- ✅ `<dyad-search-replace>` - точечные правки (если Turbo Edits включен)
- ✅ `<dyad-command>` - rebuild/restart/refresh
- ✅ `<dyad-chat-summary>` - заголовок чата

---

## Ask Mode Pipeline

**Назначение:** Консультационный режим без генерации кода. AI объясняет концепции, отвечает на вопросы, предлагает архитектурные решения.

**System Prompt:** `ASK_MODE_SYSTEM_PROMPT` + `AI_RULES`

### ASCII Схема

```
┌─────────────────────────────────────────────────────────────────────┐
│              ASK MODE - CONSULTATION PIPELINE                        │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 USER QUESTION (Ask Mode)                             │
├─────────────────────────────────────────────────────────────────────┤
│ Examples:                                                            │
│ • "How should I structure my authentication flow?"                  │
│ • "What's the best way to handle state in React?"                   │
│ • "Explain the differences between SSR and CSR"                     │
│ • "What are the security implications of storing JWT in localStorage?"│
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│             SYSTEM PROMPT COMPOSITION                                │
├─────────────────────────────────────────────────────────────────────┤
│ THINKING_PROMPT (optional)                                           │
│ + ASK_MODE_SYSTEM_PROMPT                                             │
│   "You are a helpful AI assistant..."                               │
│   - Specializes in web development guidance                         │
│   - Provides clear explanations                                      │
│   - Focus on best practices                                          │
│   - Use analogies and conceptual explanations                       │
│ + AI_RULES (for context about tech stack)                           │
│ + **CRITICAL CONSTRAINTS:**                                          │
│   "ABSOLUTE PRIMARY DIRECTIVE:                                       │
│    YOU MUST NOT, UNDER ANY CIRCUMSTANCES,                           │
│    WRITE OR GENERATE CODE"                                           │
│   - No code snippets                                                 │
│   - No syntax examples                                               │
│   - No <dyad-*> tags                                                │
│   - ONLY conceptual explanations                                     │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│              MESSAGES WITHOUT CODE GENERATION                        │
├─────────────────────────────────────────────────────────────────────┤
│ [System Message]                                                     │
│   → ASK_MODE_SYSTEM_PROMPT with strict no-code rules               │
│                                                                      │
│ [User Message 1 - Codebase Context]                                 │
│   "This is my codebase: [read-only reference]"                      │
│   NOTE: Even with codebase access, NO code generation allowed       │
│                                                                      │
│ [Previous Q&A history]                                               │
│   User: "How does React state work?"                                │
│   Assistant: "React state is like..."                               │
│                                                                      │
│ [Current Question]                                                   │
│   "What's the best way to handle auth?"                             │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   AI PROCESSING                                      │
├─────────────────────────────────────────────────────────────────────┤
│ AI uses THINKING_PROMPT to analyze:                                  │
│ <think>                                                              │
│ • **Understand the question**                                        │
│   - User asking about authentication best practices                 │
│   - Need to cover security, UX, implementation approaches          │
│                                                                      │
│ • **Structure the explanation**                                      │
│   - Start with high-level concepts                                  │
│   - Explain different approaches (JWT, sessions, OAuth)            │
│   - Discuss trade-offs of each                                      │
│   - Security considerations                                          │
│                                                                      │
│ • **Use analogies for clarity**                                      │
│   - Compare JWT to a "badge" you carry                              │
│   - Sessions like "checking in at reception"                        │
│ </think>                                                             │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│          AI GENERATES CONCEPTUAL EXPLANATION                         │
├─────────────────────────────────────────────────────────────────────┤
│ "There are several approaches to handle authentication:              │
│                                                                      │
│ **1. JWT (JSON Web Tokens)**                                         │
│ Think of JWT like a badge you wear at a conference. Once you        │
│ check in, you get a badge that proves who you are. Every time      │
│ you enter a session, security just glances at your badge -          │
│ no need to check the registration desk again.                       │
│                                                                      │
│ Benefits:                                                            │
│ - Stateless - server doesn't need to store session data            │
│ - Scalable - works well with microservices                          │
│ - Can include user claims in the token                              │
│                                                                      │
│ Trade-offs:                                                          │
│ - Cannot invalidate tokens early (until expiry)                     │
│ - Token size can grow if you include много claims                   │
│ - Storage location matters for security                             │
│                                                                      │
│ **2. Session-based Authentication**                                  │
│ This is like checking in at a hotel reception. They give you       │
│ a room key (session ID) that corresponds to your reservation        │
│ in their system. Each time you use the key, they can verify        │
│ your reservation is still valid.                                     │
│                                                                      │
│ Benefits:                                                            │
│ - Easy to invalidate (just delete server session)                  │
│ - Smaller client-side footprint (just session ID)                  │
│ - More control over active sessions                                 │
│                                                                      │
│ Trade-offs:                                                          │
│ - Requires server-side session storage                              │
│ - Scaling requires session sharing (Redis, etc.)                    │
│ - Additional server overhead                                         │
│                                                                      │
│ **Security Considerations:**                                         │
│ - HTTPS is mandatory for both approaches                            │
│ - Consider CSRF protection for cookie-based auth                    │
│ - Use HttpOnly cookies to prevent XSS attacks                       │
│ - Implement rate limiting on auth endpoints                         │
│ - Add MFA for sensitive applications                                │
│                                                                      │
│ **Recommendation for your stack:**                                   │
│ Given you're using [tech stack from AI_RULES], I'd suggest...       │
│ [contextual recommendation based on project]"                       │
│                                                                      │
│ ❌ NO CODE EXAMPLES                                                  │
│ ❌ NO <dyad-write> or other tags                                    │
│ ✅ ONLY conceptual explanations                                      │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   RESPONSE VALIDATION                                │
├─────────────────────────────────────────────────────────────────────┤
│ System checks for forbidden content:                                 │
│                                                                      │
│ ❌ Contains ```code blocks```? → VIOLATION                          │
│ ❌ Contains <dyad-*> tags? → VIOLATION                              │
│ ❌ Contains function/class definitions? → VIOLATION                 │
│                                                                      │
│ ✅ Only explanatory text? → ALLOWED                                 │
│ ✅ Uses analogies and concepts? → ALLOWED                           │
│ ✅ Discusses approaches and trade-offs? → ALLOWED                   │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  SAVE & DISPLAY TO USER                              │
├─────────────────────────────────────────────────────────────────────┤
│ • Save explanation to database                                       │
│ • Display formatted explanation to user                             │
│ • No code changes to codebase                                        │
│ • No file operations                                                 │
│ • Pure knowledge transfer                                            │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
                        ┌────────┐
                        │  DONE  │
                        └────────┘
```

### Ключевые особенности Ask Mode

**1. Строгий запрет на код:**
- Абсолютно никакого кода
- Никаких синтаксических примеров
- Никаких file operations
- Только концептуальные объяснения

**2. Образовательный фокус:**
- Объяснение "почему", а не "как"
- Использование аналогий
- Обсуждение trade-offs
- Multiple solution approaches

**3. Доступ к кодовой базе:**
- Read-only access
- Для контекста и понимания
- НЕ для генерации кода

**4. Разрешенный контент:**
- ✅ Текстовые объяснения
- ✅ Архитектурные рекомендации
- ✅ Best practices
- ✅ Концептуальные диаграммы (в тексте)
- ✅ Trade-off analysis

**5. Запрещенный контент:**
- ❌ Код любого вида
- ❌ Все `<dyad-*>` теги
- ❌ Markdown code blocks
- ❌ Syntax examples

---

## Agent Mode Pipeline

**Назначение:** Исследовательский режим, где AI собирает информацию и определяет требуемые инструменты/API/ресурсы перед началом разработки.

**System Prompt:** `AGENT_MODE_SYSTEM_PROMPT`

### ASCII Схема

```
┌─────────────────────────────────────────────────────────────────────┐
│           AGENT MODE - RESEARCH & PREPARATION PIPELINE               │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│               USER REQUEST (Agent Mode)                              │
├─────────────────────────────────────────────────────────────────────┤
│ Examples:                                                            │
│ • "Build a weather app"                                              │
│ • "Create a payment processing system"                              │
│ • "Make a real-time chat application"                               │
│ • "Build an e-commerce store"                                        │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│             SYSTEM PROMPT (Agent Specific)                           │
├─────────────────────────────────────────────────────────────────────┤
│ AGENT_MODE_SYSTEM_PROMPT                                             │
│   "You are an AI App Builder Agent..."                              │
│   - Analyze development requests                                     │
│   - Determine required tools/APIs/data                              │
│   - Research available resources                                     │
│   - NO CODE GENERATION                                               │
│                                                                      │
│ **Core Mission:**                                                    │
│ Gather all necessary information before coding phase begins         │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 TOOL USAGE DECISION                                  │
├─────────────────────────────────────────────────────────────────────┤
│ AI analyzes request and determines if tools needed:                  │
│                                                                      │
│ ┌─────────────────────────────────────────────────┐                 │
│ │ REQUEST TYPE CHECK                              │                 │
│ ├─────────────────────────────────────────────────┤                 │
│ │ Simple app (calculator, form, etc.)?            │                 │
│ │   → NO TOOLS NEEDED                             │                 │
│ │   → Response: "Ok, looks like I don't need any  │                 │
│ │                tools, I can start building."    │                 │
│ │                                                 │                 │
│ │ Complex app (external APIs, real-time data)?    │                 │
│ │   → TOOLS NEEDED                                │                 │
│ │   → Continue to research phase                  │                 │
│ └─────────────────────────────────────────────────┘                 │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                  ┌──────────┴──────────┐
                  │                     │
                  ▼                     ▼
         ┌────────────────┐    ┌────────────────────┐
         │ NO TOOLS       │    │ RESEARCH WITH      │
         │ NEEDED         │    │ TOOLS              │
         └───────┬────────┘    └─────────┬──────────┘
                 │                       │
                 │                       ▼
                 │          ┌────────────────────────────────────┐
                 │          │  RESEARCH PHASE                    │
                 │          ├────────────────────────────────────┤
                 │          │ MCP Tools available:               │
                 │          │ • Web search                       │
                 │          │ • API documentation lookup         │
                 │          │ • Package registry search          │
                 │          │                                    │
                 │          │ AI investigates:                   │
                 │          │ 1. Available APIs                  │
                 │          │    "Search for weather APIs..."    │
                 │          │    → OpenWeather, WeatherAPI.com   │
                 │          │                                    │
                 │          │ 2. Authentication methods          │
                 │          │    "Check API auth requirements"   │
                 │          │    → API key, OAuth, etc.          │
                 │          │                                    │
                 │          │ 3. Library options                 │
                 │          │    "Search npm for weather libs"   │
                 │          │    → axios, fetch, etc.            │
                 │          │                                    │
                 │          │ 4. Best practices                  │
                 │          │    "Look up weather app patterns"  │
                 │          │    → Caching, error handling       │
                 │          └────────────┬───────────────────────┘
                 │                       │
                 │                       ▼
                 │          ┌────────────────────────────────────┐
                 │          │  SUMMARIZE FINDINGS                │
                 │          ├────────────────────────────────────┤
                 │          │ "For the weather app, I found:     │
                 │          │                                    │
                 │          │ **APIs Available:**                │
                 │          │ - OpenWeather API (free tier)      │
                 │          │ - WeatherAPI.com (1M requests/mo)  │
                 │          │                                    │
                 │          │ **Auth Requirements:**             │
                 │          │ - API key (sign up required)       │
                 │          │                                    │
                 │          │ **Libraries Recommended:**         │
                 │          │ - axios for HTTP requests          │
                 │          │ - react-query for caching          │
                 │          │                                    │
                 │          │ **Additional Considerations:**     │
                 │          │ - Rate limiting implementation     │
                 │          │ - Geolocation API for user location│
                 │          │ - Error handling for offline       │
                 │          │                                    │
                 │          │ Ready to build with these insights"│
                 │          └────────────┬───────────────────────┘
                 │                       │
                 └───────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   FINAL RESPONSE                                     │
├─────────────────────────────────────────────────────────────────────┤
│ SIMPLE APP:                                                          │
│   "Ok, looks like I don't need any tools, I can start building."    │
│   → System switches to Build Mode automatically                     │
│                                                                      │
│ COMPLEX APP (after research):                                        │
│   "Here's what I found: [research summary]                          │
│    Ready to build with this information."                           │
│   → User can review findings                                         │
│   → Then switch to Build Mode to implement                          │
│                                                                      │
│ ❌ NO CODE GENERATION in Agent Mode                                 │
│ ❌ NO <dyad-*> tags                                                 │
│ ✅ ONLY information gathering and analysis                           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│              TRANSITION TO BUILD MODE                                │
├─────────────────────────────────────────────────────────────────────┤
│ After Agent Mode completes research:                                 │
│   • Information is available in chat history                        │
│   • User (or system) switches to Build Mode                         │
│   • Build Mode AI has full context from research                    │
│   • Begins implementation with informed decisions                   │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
                        ┌────────┐
                        │  DONE  │
                        └────────┘
```

### Ключевые особенности Agent Mode

**1. Information Gathering:**
- Определяет нужные API и сервисы
- Исследует документацию
- Находит библиотеки и фреймворки
- Анализирует best practices

**2. Tool Usage Decision Framework:**

**Use Tools When:**
- External APIs needed (payment, auth, maps, social)
- Real-time data required (weather, stocks, news)
- Third-party integrations (Firebase, Supabase, cloud)
- Current framework documentation needed

**Don't Use Tools When:**
- Simple standalone app
- No external dependencies
- Standard web tech sufficient
- Examples: calculator, form, static page

**3. MCP Tools Integration:**
- Web search for API discovery
- Documentation lookup
- Package registry search
- Best practices research

**4. Constraints:**
- ❌ Absolutely NO code generation
- ❌ NO implementation details
- ❌ NO <dyad-*> tags
- ✅ ONLY information and analysis

**5. Output Format:**

**When no tools needed:**
```
"Ok, looks like I don't need any tools, I can start building."
```

**When tools used:**
```
"Here's what I found:

**Available APIs:**
- [API name]: [description, pricing, limits]

**Authentication:**
- [Auth method and requirements]

**Libraries:**
- [Recommended packages]

**Considerations:**
- [Security, performance, UX notes]

Ready to build with this information."
```

---
