# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a multi-agent coding platform template built on Next.js 15 that allows users to create automated coding tasks executed in isolated Vercel Sandboxes. It supports multiple AI coding agents (Claude Code, OpenAI Codex CLI, GitHub Copilot CLI, Cursor CLI, Google Gemini CLI, and opencode) and features user authentication, task management, and real-time execution monitoring.

**Key Technologies:**
- Next.js 15 with React 19 and Turbopack
- PostgreSQL database with Drizzle ORM
- Vercel Sandbox for isolated code execution
- Multi-provider OAuth authentication (GitHub, Vercel)
- AI SDK 5 with Vercel AI Gateway integration
- Real-time streaming with WebSocket connections

## Development Commands

### Running the Application

```bash
# Start development server with Turbopack
pnpm dev

# Build for production
pnpm build

# Start production server
pnpm start
```

### Database Operations

```bash
# Generate new migrations from schema changes
pnpm db:generate

# Push schema changes directly to database (skip migration files)
pnpm db:push

# Apply migrations to database
pnpm db:migrate

# Open Drizzle Studio (database GUI)
pnpm db:studio
```

### Code Quality

```bash
# Format code with Prettier
pnpm format

# Check formatting without modifying files
pnpm format:check

# Run TypeScript type checking
pnpm type-check

# Run ESLint
pnpm lint
```

**CRITICAL:** After editing any `.ts` or `.tsx` files, you MUST run `pnpm format`, `pnpm type-check`, and `pnpm lint` to ensure code quality. Fix all errors before considering work complete.

## High-Level Architecture

### Request Flow: Task Creation → Execution → Completion

1. **User creates task** via web UI (`app/tasks/[taskId]/page.tsx`)
2. **API route** (`app/api/tasks/route.ts`) validates input and creates database record
3. **Branch name generation** happens asynchronously via AI SDK 5 + AI Gateway (non-blocking using Next.js 15's `after()`)
4. **Sandbox creation** (`lib/sandbox/creation.ts`):
   - Validates environment variables
   - Clones repository with GitHub authentication
   - Installs dependencies (npm/pnpm/yarn/pip auto-detected)
   - Configures Git with user credentials
   - Creates or checks out branch
5. **Agent execution** (`lib/sandbox/agents/`):
   - Agent-specific setup (installing CLI tools, authentication)
   - Streams execution output to database (`task_messages` table)
   - Monitors for file changes via git status
   - Captures session IDs for resumption (Claude only)
6. **Git operations** (`lib/sandbox/git.ts`):
   - Commits changes with user's Git identity
   - Pushes to remote branch with authentication
7. **Cleanup**: Sandbox shutdown (immediate if `keepAlive=false`, delayed if `keepAlive=true`)

### Authentication & Authorization Flow

The system uses session-based authentication with JWT encryption:

1. **OAuth Sign-In**:
   - GitHub: `app/api/auth/github/signin/route.ts` → callback at `/api/auth/github/callback`
   - Vercel: `app/api/auth/signin/vercel/route.ts` → callback at `/api/auth/callback/vercel`
2. **Session Creation** (`lib/session/create.ts`):
   - Encrypts OAuth tokens using `ENCRYPTION_KEY`
   - Creates or updates user in `users` table
   - Issues JWE session token (encrypted with `JWE_SECRET`)
   - Stores session token in httpOnly cookie
3. **Request Authorization**:
   - API routes call `getServerSession()` to validate session
   - Decrypts session token to get user ID
   - Queries database for user and linked accounts
   - Returns decrypted GitHub token for repository access

**Important:** Each user uses their own GitHub token for repository operations - there is no shared fallback token. Users who sign in with Vercel must connect GitHub separately to access repositories.

### Database Schema Overview

**Core Tables:**
- `users`: User profiles and primary OAuth account (GitHub or Vercel)
- `accounts`: Additional linked accounts (e.g., Vercel users connecting GitHub)
- `keys`: User-provided API keys (Anthropic, OpenAI, Cursor, Gemini, AI Gateway)
- `tasks`: Task records with status, logs, sandbox info, PR details
- `task_messages`: Conversation history between user and agent (for follow-up messages)
- `connectors`: MCP server configurations (Claude Code only)
- `settings`: Per-user configuration overrides

**Key Relationships:**
- All tables (except `users`) have `userId` foreign key → cascade delete on user removal
- `accounts.userId` → `users.id`: Links additional accounts to users
- `tasks.userId` → `users.id`: User ownership of tasks
- `task_messages.taskId` → `tasks.id`: Message thread for each task
- `connectors.userId` → `users.id`: MCP servers are user-scoped

**Encryption:** All sensitive data (OAuth tokens, API keys, connector env vars) is encrypted at rest using the `ENCRYPTION_KEY` environment variable.

### Multi-Agent Architecture

Each agent has its own execution module in `lib/sandbox/agents/`:
- `claude.ts`: Installs @anthropic-ai/claude-code, configures MCP servers, streams JSON output
- `codex.ts`: Uses AI Gateway with OpenAI models
- `copilot.ts`: Installs and executes GitHub Copilot CLI
- `cursor.ts`: Installs and executes Cursor CLI
- `gemini.ts`: Uses Google Gemini API
- `opencode.ts`: OpenCode integration

All agents return a standardized `AgentExecutionResult` with success status, output, changes detection, and optional session ID for resumption.

### AI-Powered Branch Naming

Branch names are generated using AI SDK 5 with Vercel AI Gateway integration (`lib/utils/branch-name-generator.ts`):
- Non-blocking: Uses Next.js 15's `after()` hook to avoid delaying task creation
- Context-aware: Analyzes task description, repo name, and agent type
- Format: `{type}/{description}-{hash}` (e.g., `feature/user-auth-K3mP9n`)
- Graceful fallback: Uses timestamp-based names if AI generation fails

### MCP Server Support (Claude Only)

MCP (Model Context Protocol) servers extend Claude Code with additional tools. Configuration stored in `connectors` table:
- **Local MCP servers**: Command-based STDIO servers (e.g., `node server.js`)
- **Remote MCP servers**: HTTP/SSE endpoints with optional OAuth
- Environment variables are encrypted before storage
- Servers are added to sandbox via `claude mcp add` commands during agent setup

## Critical Security Rules

### No Dynamic Values in Logs (MUST FOLLOW)

**All log statements MUST use static strings only. NEVER include dynamic values.**

**Bad (DO NOT DO):**
```typescript
await logger.info(`Task created: ${taskId}`)
await logger.error(`Failed to process ${filename}`)
```

**Good (DO THIS):**
```typescript
await logger.info('Task created')
await logger.error('Failed to process file')
```

**Rationale:** Logs are displayed directly in the UI and can expose sensitive information (user IDs, file paths, credentials, API keys, etc.) to end users. This applies to ALL log levels and ALL logger methods.

**Sensitive data that must NEVER appear in logs:**
- Vercel credentials (SANDBOX_VERCEL_TOKEN, SANDBOX_VERCEL_TEAM_ID, SANDBOX_VERCEL_PROJECT_ID)
- User IDs and personal information
- File paths and repository URLs
- Branch names and commit messages
- API keys and OAuth tokens
- Error details containing system internals

### Environment Variables

**Never expose to client:**
- `SANDBOX_VERCEL_TOKEN`
- `SANDBOX_VERCEL_TEAM_ID`
- `SANDBOX_VERCEL_PROJECT_ID`
- `ANTHROPIC_API_KEY`
- `OPENAI_API_KEY`
- `GEMINI_API_KEY`
- `CURSOR_API_KEY`
- `AI_GATEWAY_API_KEY`
- `JWE_SECRET`
- `ENCRYPTION_KEY`

**Safe for client (NEXT_PUBLIC_ prefix):**
- `NEXT_PUBLIC_AUTH_PROVIDERS`
- `NEXT_PUBLIC_GITHUB_CLIENT_ID`
- `NEXT_PUBLIC_VERCEL_CLIENT_ID`

## File Organization

### API Routes Structure

```
app/api/
├── auth/                    # Authentication endpoints
│   ├── github/             # GitHub OAuth flow
│   ├── signin/             # Sign-in endpoints
│   └── callback/           # OAuth callbacks
├── tasks/                  # Task management
│   ├── route.ts           # Create/list tasks
│   └── [taskId]/          # Task-specific operations
│       ├── messages/      # Conversation history
│       ├── continue/      # Follow-up messages
│       ├── files/         # File browser
│       ├── pr/            # Pull request creation
│       └── ...            # Other task operations
├── repos/                  # Repository operations
│   └── [owner]/[repo]/    # Repo-specific endpoints
├── github/                 # GitHub integration
├── api-keys/              # User API key management
└── connectors/            # MCP server management
```

### Library Structure

```
lib/
├── sandbox/               # Sandbox creation and execution
│   ├── creation.ts       # Core sandbox setup logic
│   ├── agents/           # Agent-specific implementations
│   ├── commands.ts       # Command execution helpers
│   ├── git.ts           # Git operations
│   └── package-manager.ts # Dependency installation
├── db/                    # Database layer
│   ├── schema.ts         # Drizzle schema definitions
│   ├── client.ts         # Database client
│   └── users.ts          # User queries
├── session/               # Authentication session management
├── github/                # GitHub API integration
├── utils/                 # Utilities
│   ├── task-logger.ts    # Database-backed logging
│   ├── logging.ts        # Credential redaction
│   └── branch-name-generator.ts # AI branch naming
└── api-keys/              # User API key retrieval
```

## Common Development Patterns

### Adding a New API Route

1. Create route file in appropriate `app/api/` subdirectory
2. Import and call `getServerSession()` to validate authentication
3. Validate request body with Zod schemas from `lib/db/schema.ts`
4. Query database using Drizzle ORM client from `lib/db/client.ts`
5. Filter results by `userId` to ensure user can only access their own data
6. Return JSON response with appropriate status codes

### Working with the Database

1. Modify schema in `lib/db/schema.ts` (add tables, columns, indexes)
2. Run `pnpm db:generate` to create migration files
3. Run `pnpm db:push` to apply changes to database
4. Update TypeScript types are automatically inferred from schema

### Implementing a New Agent

1. Create new file in `lib/sandbox/agents/` (e.g., `myagent.ts`)
2. Export async function matching signature: `execute{Agent}InSandbox(sandbox, instruction, logger, ...)`
3. Install agent CLI if needed (check if already installed for resumed sandboxes)
4. Execute agent with instruction, stream output to `task_messages` if possible
5. Check for file changes via `git status --porcelain`
6. Return `AgentExecutionResult` with success, output, changesDetected, and optional sessionId
7. Add agent to `lib/sandbox/agents/index.ts` export
8. Update task schema in `lib/db/schema.ts` to include new agent in enum

## Testing Changes

### End-to-End Testing Flow

1. Sign in with GitHub or Vercel OAuth
2. Create a new task with a test repository
3. Monitor task execution logs in real-time
4. Verify changes are committed and pushed to branch
5. Check that sandbox shuts down correctly based on keepAlive setting
6. Test follow-up messages if keepAlive is enabled

### Database Testing

Use Drizzle Studio to inspect database state:
```bash
pnpm db:studio
```

### Security Testing

Search for potential credential leaks:
```bash
# Check for dynamic log values
grep -r "logger\.(info|error|success|command)(\`.*\${" .
grep -r "console\.(log|error|warn|info)(\`.*\${" .
```

## Repository Page Architecture

The repository browser uses Next.js nested layouts with separate pages for each tab:

**Routes:**
- `app/repos/[owner]/[repo]/layout.tsx` - Shared layout with tab navigation
- `app/repos/[owner]/[repo]/page.tsx` - Redirects to `/commits`
- `app/repos/[owner]/[repo]/commits/page.tsx` - Commits list
- `app/repos/[owner]/[repo]/issues/page.tsx` - Issues list
- `app/repos/[owner]/[repo]/pull-requests/page.tsx` - Pull requests list

**To add a new tab:**
1. Create directory: `app/repos/[owner]/[repo]/[new-tab]/`
2. Add `page.tsx` with component
3. Create API route: `app/api/repos/[owner]/[repo]/[new-tab]/route.ts`
4. Update `tabs` array in `components/repo-layout.tsx`

## Important Notes

- **Sandbox Timeout:** Controlled by `maxDuration` setting (default 300 minutes). All work must complete within this timeframe.
- **Keep Alive:** If enabled, sandbox stays alive after task completion for follow-up messages until timeout expires.
- **Dependency Installation:** Auto-detects package manager (pnpm, yarn, npm) and Python via presence of lock files.
- **Branch Naming:** AI-generated names are created asynchronously and may not be immediately available when sandbox starts.
- **Session Resumption:** Only Claude Code supports resuming conversations via session IDs.
- **Rate Limiting:** Configurable via `MAX_MESSAGES_PER_DAY` environment variable (default: 5 tasks + follow-ups per user per day).
- **Encryption:** All sensitive data is encrypted with `ENCRYPTION_KEY` before database storage.
- **OAuth Tokens:** User OAuth tokens are automatically refreshed when expired (if refresh token is available).

## Key Files to Understand

- `AGENTS.md` - Critical security and development guidelines (READ FIRST)
- `lib/sandbox/creation.ts` - Core sandbox setup and initialization logic
- `lib/sandbox/agents/claude.ts` - Claude Code integration with MCP support
- `lib/session/server.ts` - Session validation and user authentication
- `lib/db/schema.ts` - Complete database schema with all tables and relationships
- `app/api/tasks/route.ts` - Task creation endpoint with AI branch naming
- `app/api/tasks/[taskId]/continue/route.ts` - Follow-up message handling
- `lib/utils/branch-name-generator.ts` - AI-powered branch name generation
