# Session Context: Review Project Codebase
**Session ID**: review-project-codebase-01JfPcZCsDG5sh1BcS1oc7k3
**Branch**: claude/review-project-codebase-01JfPcZCsDG5sh1BcS1oc7k3
**Created**: 2025-12-10
**Purpose**: Comprehensive codebase review to understand project architecture and implementation

---

## Project Overview

This is a **Next.js 13 AI Chat Application** with real-time streaming capabilities, demonstrating:
- Real-time streaming chat with OpenAI GPT-4o using Vercel's Data Stream Protocol v1
- Tool calling/function calling support (e.g., weather service)
- Multi-conversation management with persistence
- Dark/Light theme support
- Multimodal input with attachments
- Pure hexagonal architecture with Domain-Driven Design

---

## Architecture Summary

### Hexagonal Architecture (Ports & Adapters)

The codebase strictly follows hexagonal architecture with three primary layers:

```
src/
├── domain/                # Core business logic (ZERO framework dependencies)
│   ├── entities/         # Conversation, Message, ToolInvocation, StreamingResponse
│   ├── value-objects/    # MessageRole, MessageContent, ToolName, Coordinates, Attachment
│   ├── services/         # ConversationOrchestrator, MessageValidator
│   ├── repositories/     # IConversationRepository interface
│   ├── exceptions/       # Domain-specific errors
│   └── __test-helpers__/ # Builders and factories for tests
│
├── application/          # Use cases and application services
│   ├── use-cases/       # SendMessageUseCase, StreamChatCompletionUseCase, etc.
│   ├── ports/
│   │   ├── inbound/    # IChatService (exposed to presentation)
│   │   └── outbound/   # IAIProvider, IStreamAdapter, IToolRegistry
│   ├── dto/            # Data transfer objects
│   └── mappers/        # Entity <-> DTO conversions
│
└── infrastructure/       # Adapters and external integrations
    ├── adapters/        # OpenAI, Vercel Stream, Tools, MongoDB
    ├── repositories/    # In-memory and MongoDB implementations
    └── config/          # DependencyContainer (IoC)

app/                      # Next.js App Router (Frontend)
├── api/conversations/   # API routes (thin controllers)
├── features/conversation/ # Feature-based UI organization
│   ├── components/      # React components
│   ├── hooks/          # Business logic hooks (useConversation)
│   └── data/           # Frontend services and schemas
└── layout.tsx          # Root layout with theme provider
```

---

## Tech Stack

### Backend
- **Framework**: Next.js 13 with App Router
- **Language**: TypeScript (strict mode)
- **AI**: OpenAI SDK (GPT-4o)
- **Streaming**: Vercel AI SDK + custom Data Stream Protocol v1 adapter
- **Database**: MongoDB with in-memory fallback
- **Testing**: Vitest + React Testing Library

### Frontend
- **UI**: React 18.2.0
- **State**: React Query (TanStack) + Vercel AI SDK's useChat
- **Components**: shadcn/ui (new-york style, zinc colors)
- **Styling**: Tailwind CSS + next-themes
- **Markdown**: react-markdown with remark-gfm
- **Icons**: Lucide React

---

## Domain Model

### Core Entities

#### Conversation (Aggregate Root)
- **Location**: `/src/domain/entities/Conversation.ts`
- **Responsibility**: Conversation lifecycle and business rules
- **Features**:
  - Message ordering validation (user → assistant → tool)
  - Status management: ACTIVE → WAITING_FOR_RESPONSE → COMPLETED/ARCHIVED
  - Message limit enforcement (max 1000)
  - Auto-title generation
  - Metadata storage
  - Tool invocation tracking

#### Message (Entity)
- **Location**: `/src/domain/entities/Message.ts`
- **Features**:
  - Role validation (user, assistant, system, tool)
  - Content and attachments management
  - Tool invocation tracking
  - Sequence validation

#### ToolInvocation (Entity)
- **Location**: `/src/domain/entities/ToolInvocation.ts`
- **State Machine**: PENDING → EXECUTING → COMPLETED|FAILED
- Immutable arguments (frozen)

#### StreamingResponse (Entity)
- **Location**: `/src/domain/entities/StreamingResponse.ts`
- Tracks streaming state and token usage
- States: Created → Started → Completed/Failed

### Value Objects
- MessageRole, MessageContent, ToolName, Attachment, Coordinates

### Domain Services
- **ConversationOrchestrator**: Complex conversation operations
- **MessageValidator**: Message validation logic

---

## Application Layer

### Use Cases

#### StreamChatCompletionUseCase
- **Location**: `/src/application/use-cases/StreamChatCompletionUseCase.ts`
- Orchestrates AI streaming with tool execution
- Flow: retrieve → validate → stream → execute tools → save

#### SendMessageUseCase
- **Location**: `/src/application/use-cases/SendMessageUseCase.ts`
- Adds messages to conversations
- Methods: execute, executeMultiple, canSendMessage

#### ManageConversationUseCase
- Create/retrieve/list/archive conversations

#### ExecuteToolUseCase
- Executes individual tools with validation

### Ports

**Inbound**: IChatService (exposed to API routes)
**Outbound**: IAIProvider, IStreamAdapter, IToolRegistry, IWeatherService, IConversationRepository

---

## Infrastructure

### Adapters

- **OpenAIAdapter**: Implements IAIProvider, streams from GPT-4o
- **VercelStreamAdapter**: Implements IStreamAdapter, Vercel Data Stream Protocol v1
- **ToolRegistry**: Manages tool registration and execution
- **WeatherTool**: Sample tool using Open-Meteo API
- **MongoDBConversationRepository**: MongoDB persistence
- **InMemoryConversationRepository**: Fallback repository

### Dependency Injection

**DependencyContainer** (`/src/infrastructure/config/DependencyContainer.ts`):
- Singleton pattern with async initialization
- Wires all adapters and use cases
- Supports configuration (API keys, DB URL)
- Fallback strategy: MongoDB → In-Memory
- Health check endpoint

---

## Frontend Architecture

### Main Business Hook: useConversation

**Location**: `/app/features/conversation/hooks/useConversation.tsx:37`

**Primary interface for chat functionality**:
- Wraps Vercel AI SDK's useChat
- Manages conversation storage (localStorage)
- Provides derived state (isThinking, isEmpty, hasMessages)
- Handles errors with user-friendly messages
- Syncs with server after streaming
- Invalidates React Query cache

### Container Pattern

- **ChatContainer.tsx**: Smart component using hooks
- **Chat.tsx**: Pure presentation component (props only)

### State Management

- **Vercel AI SDK**: Core streaming state
- **React Query**: Conversation list queries/mutations
- **localStorage**: Conversation persistence
- **Context API**: Handler exposure

---

## API Communication Flow

```
1. Frontend (useConversation) → POST /api/conversations
2. API route → DependencyContainer
3. ManageConversationUseCase (ensure conversation exists)
4. SendMessageUseCase (add user message)
5. StreamChatCompletionUseCase (stream response)
   ├── OpenAIAdapter (call GPT-4o)
   ├── VercelStreamAdapter (encode stream)
   ├── ToolRegistry (execute tools)
   └── ConversationRepository (save state)
6. Frontend receives stream → updates UI → syncs with DB
```

---

## Testing Strategy

### Configuration
- **File**: `/vitest.config.ts`
- Environment: Node.js
- Coverage: 95% statements, 90% branches, 95% functions, 95% lines
- Only domain layer measured

### Test Coverage
- **Domain Entities**: Comprehensive unit tests (Conversation, Message, ToolInvocation, StreamingResponse)
- **Value Objects**: All tested (MessageRole, MessageContent, ToolName, etc.)
- **Domain Services**: ConversationOrchestrator, MessageValidator
- **Exceptions**: All custom exceptions tested
- **Frontend**: ThemeProvider, ThemeToggle

### Test Helpers
- ConversationBuilder, MessageBuilder, ToolInvocationBuilder (fluent builders)
- MessageFactory (factory methods)

---

## Configuration

### Environment Variables
```
OPENAI_API_KEY=required
REPOSITORY_TYPE=mongodb|inmemory
MONGODB_URL=mongodb://...
DATABASE_NAME=ai_chat_app
```

### Scripts
```bash
yarn dev              # Development server (port 3000)
yarn build            # Production build
yarn start            # Production server
yarn lint             # ESLint
yarn test             # Run tests
yarn test:coverage    # Coverage report
```

### Path Aliases
```
@/*              → root
@/domain/*       → ./src/domain/*
@/application/*  → ./src/application/*
@/infrastructure/* → ./src/infrastructure/*
```

---

## Key Files

### Backend Core
- `/src/domain/entities/Conversation.ts` - Aggregate root
- `/src/application/use-cases/StreamChatCompletionUseCase.ts` - Main streaming logic
- `/src/infrastructure/config/DependencyContainer.ts` - IoC container
- `/app/api/conversations/route.ts` - API endpoint

### Frontend Core
- `/app/features/conversation/hooks/useConversation.tsx` - Main business hook
- `/app/features/conversation/components/chat-container.tsx` - Smart container
- `/app/features/conversation/components/chat.tsx` - Presentation component

### Configuration
- `/CLAUDE.md` - Project guidelines and instructions
- `/vitest.config.ts` - Test configuration
- `/tsconfig.json` - Path aliases
- `/components.json` - shadcn/ui config

---

## Project Statistics

| Metric | Count |
|---|---|
| Total Files | 118 |
| Domain Layer Files | 35+ |
| Application Layer Files | 16 |
| Infrastructure Files | 14+ |
| Frontend Components | 20+ |
| Test Files | 20+ |
| Dependencies | 32 |
| Lines of Code | ~10,000+ |

---

## Architectural Strengths

✅ Pure hexagonal architecture (domain has zero framework dependencies)
✅ Strict TypeScript type safety
✅ 95% domain test coverage
✅ Clear separation of concerns
✅ Extensible (easy to add AI providers, tools, streaming adapters)
✅ Resilient (fallback repositories)
✅ Real-time streaming with tool execution
✅ Modern React patterns with hooks
✅ shadcn/ui for accessible components
✅ Theme support (dark/light)

---

## Next Steps / Notes

- This session completed a comprehensive codebase review
- All major components, architecture layers, and patterns documented
- Ready for feature development, bug fixes, or enhancements
- Consider creating feature-specific context files for future work

---

## Session Log

### 2025-12-10: Initial Review
- ✅ Used Explore agent to perform comprehensive codebase analysis
- ✅ Documented all layers: domain, application, infrastructure, frontend
- ✅ Identified key patterns: hexagonal architecture, DDD, ports & adapters
- ✅ Catalogued all entities, value objects, use cases, and adapters
- ✅ Documented testing strategy and configuration
- ✅ Created this context file for future reference
