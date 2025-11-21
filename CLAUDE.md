# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Core Commands

### Development
- `pnpm build` - Build all packages and apps
- `pnpm lint` - Run ESLint across all packages
- `pnpm test` - Run tests across all packages
- `pnpm prettier` - Check code formatting
- `pnpm prettier:fix` - Fix code formatting

### Package-specific commands
- `pnpm --filter=@assistant-ui/docs dev` - Start docs development server
- Individual packages have their own `build`, `test`, `lint` scripts

## Repository Architecture

This is a pnpm workspace monorepo with the following structure:

### Core Packages (`/packages/`)
- **`@assistant-ui/react`** - Main React library with primitives for building AI chat UIs
- **`assistant-stream`** - Core streaming utilities and assistant transport protocol
- **`@assistant-ui/cloud`** - Assistant Cloud integration for chat persistence and analytics
- **CLI tools**: `@assistant-ui/cli`, `create-assistant-ui` for project scaffolding

### Runtime Integrations
- **`@assistant-ui/react-ai-sdk`** - Vercel AI SDK integration
- **`@assistant-ui/react-langgraph`** - LangGraph integration
- **`@assistant-ui/react-data-stream`** - Data stream protocol integration
- **`@assistant-ui/react-ag-ui`** - AG Grid UI integration

### UI Components
- **`@assistant-ui/react-markdown`** - Markdown rendering with syntax highlighting
- **`@assistant-ui/react-syntax-highlighter`** - Code syntax highlighting
- **`@assistant-ui/styles`** - Base CSS styles and Tailwind components

### Applications (`/apps/`)
- **`docs`** - Documentation site (Next.js + Fumadocs)
- **`devtools-extension`** - Browser extension for debugging
- **`devtools-frame`** - DevTools UI frame
- **`registry`** - Component registry for copying components

### Examples (`/examples/`)
Multiple example integrations showing different runtime configurations and use cases.

## Key Architecture Concepts

### Runtime Pattern
The library uses a "runtime" pattern where different backends (AI SDK, LangGraph, custom) are abstracted behind a common interface. Each runtime manages:
- Message state and streaming
- Tool execution and UI rendering
- Threading and conversation persistence

### Primitive Components
Following Radix UI patterns, the library provides unstyled primitives that compose together:
- `Thread` - Main conversation container
- `Message` - Individual message display
- `Composer` - Message input interface
- `ActionBar` - Message actions (copy, retry, etc.)

### Transport Protocol
Uses `assistant-stream` for standardized streaming protocol with support for:
- Text streaming
- Tool calls and results
- Reasoning steps
- Error handling

## Development Guidelines

### Package Management
- Use `pnpm` for all package operations
- Workspace dependencies use `workspace:*` protocol
- All packages must build successfully before publishing

### Testing
- Run `pnpm test` to execute all package tests
- Individual packages use Vitest for testing
- Main React package includes mutation testing with Stryker

### Code Style
- ESLint configuration extends Next.js + TypeScript rules
- Prettier with Tailwind plugin for formatting
- TypeScript strict mode enabled across all packages

### Build System
- Turbo for build orchestration with dependency tracking
- Individual packages use custom build scripts (typically with tsx)
- Build outputs go to `dist/` directories

## Environment Variables
Required for certain features (documented in turbo.json):
- `OPENAI_*` - OpenAI API configuration
- `ASSISTANT_*` - Assistant Cloud configuration
- `NEXT_PUBLIC_*` - Public environment variables for Next.js apps