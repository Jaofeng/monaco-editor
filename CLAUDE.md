# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Monaco Editor is the browser-based code editor that powers VS Code. The **core editor** (`monaco-editor-core`) is built directly from the VS Code source — this repo wraps that core and adds language definitions (87 languages) and editor features on top.

## Build & Development Commands

```bash
npm install                        # Install dependencies
npm run build                      # Full build (LSP client + editor)
npm run build-monaco-editor        # Build editor only (outputs to out/monaco-editor/)
npm run watch                      # TypeScript watch mode (src/ only)

npm run test                       # Run all tests (sample checks + grammar tests)
npm run test:grammars              # Grammar tests only (Monarch tokenizers)
# Single language grammar test:
node --import tsx --require ./test/test-setup.js --test "src/languages/definitions/python/python.test.ts"

npm run prettier-check             # Check formatting
npm run prettier                   # Fix formatting

# Smoke tests (require build + packaging first)
npm run build-monaco-editor
npm run package-for-smoketest      # Package for webpack, esbuild, vite
npm run smoketest                  # Run Playwright smoke tests
npm run smoketest-debug            # Debug smoke tests with Playwright inspector
```

## Architecture

### Entry Point Chain

`src/index.ts` is the main entry point:
- Re-exports `monaco-editor-core` (the VS Code editor engine)
- Imports `src/languages/register.all.ts` — registers all 87 language tokenizers
- Imports `src/features/register.all.ts` — registers all 65+ editor features
- Exports `lsp` (Language Server Protocol client from `monaco-lsp-client/`)

### Three Layers

1. **Core** (`monaco-editor-core` npm package) — Editor rendering, viewport, DOM, core APIs. Not modified in this repo; debug core issues in the [VS Code repo](https://github.com/microsoft/vscode) directly.

2. **Languages** (`src/languages/definitions/{lang}/`) — Each language has:
   - `register.ts` — Lazy-loads the grammar via `LazyLanguageLoader`
   - `{lang}.ts` — Monarch tokenizer definition
   - `{lang}.test.ts` — Grammar tests
   - Four built-in languages have full IntelliSense (CSS, HTML, JSON, TypeScript/JavaScript); the rest use Monarch tokenization only.

3. **Features** (`src/features/{feature}/`) — Editor features (find, suggest, folding, hover, etc.) Each has a `register.ts` or `.js` that imports from `monaco-editor-core`.

### Build System

- **ESM** (primary): Rollup with esbuild transpiler → `out/monaco-editor/esm/`
- **AMD** (deprecated): Vite → `out/monaco-editor/min/` (prod) and `out/monaco-editor/dev/`
- **Types**: Rollup with DTS plugin → `out/monaco-editor/monaco.d.ts`
- Build orchestration: `build/build-monaco-editor.ts`
- Path mapping in `build/shared.mjs` remaps `src/` → `vs/`, `node_modules/monaco-editor-core/esm/` → `.`

### Deprecated API Layer

`src/deprecated/` contains backward-compatible re-exports mapped to the `vs/` namespace for consumers using older import paths.

### Webpack Plugin

`/webpack-plugin/` is a separate npm package (`monaco-editor-webpack-plugin`) that auto-configures web workers and allows cherry-picking languages/features.

## Code Style

- **Prettier**: tabs, single quotes, no trailing commas, 100 char width
- **Pre-commit hook**: `pretty-quick` auto-formats staged files (via Husky)
- **TypeScript**: strict mode, ES5 target, `emitDeclarationOnly` in src

## Adding a New Language

1. Create `src/languages/definitions/{lang}/register.ts`, `{lang}.ts`, `{lang}.test.ts`
2. Register in `src/languages/register.all.ts`
3. Add sample: `website/index/samples/sample.{lang}.txt`

## Key Dependencies

- `monaco-editor-core` — VS Code editor engine (dev version tied to specific VS Code commit via `vscodeRef` in package.json)
- `vscode-*-languageservice` — Language servers for CSS, HTML, JSON
- Node version: 22.21.1 (see `.nvmrc`)
