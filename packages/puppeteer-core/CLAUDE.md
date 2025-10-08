# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **puppeteer-core** package, a high-level API to control headless Chrome or Firefox over the DevTools Protocol (CDP) or WebDriver BiDi. Unlike the main `puppeteer` package, `puppeteer-core` does not bundle a browser and is designed to be used as a library.

The package name has been customized to `@codebyai/puppeteer-core` (private fork).

## Build Commands

### Building the project
```bash
# Build puppeteer-core (from repository root or package directory)
npm run build --workspace puppeteer-core

# Build in watch mode (auto-rebuild on changes)
npm run build --watch --workspace puppeteer-core

# Clean build artifacts
npm run clean --workspace puppeteer-core

# Build using hereby (low-level, from package directory)
hereby build
```

### Code quality
```bash
# Lint code
npm run lint

# Auto-fix linting issues
npm run format

# Check specific linting
npm run lint:eslint
npm run lint:prettier
npm run lint:expectations
```

### Testing
```bash
# Run all tests (from repository root)
npm test

# Run specific test suites
npm run test:chrome:headless
npm run test:chrome:headful
npm run test:chrome:bidi
npm run test:firefox:headless
npm run test:firefox:headful

# Run unit tests only (Node test runner, requires Node 20+)
npm run unit --workspace puppeteer-core

# Run unit tests with only() filter
npm run unit:only --workspace puppeteer-core

# Build test workspace before running tests
npm run build --workspace @puppeteer-test/test && npm test

# Run with custom browser
BINARY=<path-to-executable> npm run test:chrome:headless
```

### Documentation
```bash
# Generate documentation
npm run docs

# Run doctest
npm run doctest
```

## Architecture

### Protocol Implementations

Puppeteer-core supports two browser automation protocols:

1. **CDP (Chrome DevTools Protocol)** - Located in `src/cdp/`
   - Original Puppeteer protocol implementation
   - Chrome/Chromium-specific features
   - Connection via WebSocket to DevTools
   - Classes: `CdpBrowser`, `CdpBrowserContext`, `CdpPage`, `CdpFrame`, etc.

2. **WebDriver BiDi** - Located in `src/bidi/`
   - Modern cross-browser protocol (Chrome and Firefox)
   - Two layers:
     - `src/bidi/core/` - Low-level WebDriver BiDi implementation that strictly follows spec
     - `src/bidi/` - Puppeteer API implementation on top of BiDi core
   - Classes: `BidiBrowser`, `BidiBrowserContext`, `BidiPage`, `BidiFrame`, etc.

### Core Directory Structure

- **`src/api/`** - Public API interfaces and base classes
  - Abstract base classes: `Browser`, `BrowserContext`, `Page`, `Frame`, `ElementHandle`, `JSHandle`, etc.
  - These define the public Puppeteer API contract

- **`src/cdp/`** - CDP protocol implementation
  - Concrete implementations of API classes using Chrome DevTools Protocol
  - `Connection.ts` - WebSocket connection management
  - `NetworkManager.ts`, `FrameManager.ts` - Core subsystems

- **`src/bidi/`** - WebDriver BiDi implementation
  - Puppeteer API implementation using BiDi protocol
  - `src/bidi/core/` - Spec-compliant low-level BiDi layer (see dedicated README)

- **`src/common/`** - Shared utilities and components
  - Query handlers (ARIA, CSS, XPath, text, Pierce)
  - Configuration, connectivity, event handling
  - Locators implementation

- **`src/injected/`** - Code injected into browser pages
  - Built and bundled into `src/generated/injected.ts` during build

- **`src/node/`** - Node.js-specific code
  - Launcher implementations for different browsers

- **`src/util/`** - General utilities
  - Disposables, async helpers, decorators

### BiDi Core Design Principles

The `bidi/core` layer follows strict design rules (see `src/bidi/core/README.md`):
- Required arguments are inlined; optional arguments use options objects
- Session is only exposed on Browser object
- Follows WebDriver BiDi spec comprehensively but minimally
- Always implements the full graph of objects/events (no skipping or composition)
- Never follows Puppeteer's needs - strictly spec-compliant

### Build Process

The build uses **wireit** (similar to GNU Make) for dependency management:

1. **TypeScript compilation** (`hereby build`):
   - Generates both ESM (`lib/esm/`) and CJS (`lib/cjs/`) outputs
   - Compiles from `src/` using separate tsconfigs (src/tsconfig.esm.json, src/tsconfig.cjs.json)

2. **Type definitions** (`build:types`):
   - Uses API Extractor to generate unified `lib/types.d.ts`
   - Rolls up all type definitions into a single file

3. **ES5 bundle** (`build:es5`):
   - Creates browser-compatible bundle via rollup
   - Output: `lib/es5-iife/puppeteer-core-browser.js`

4. **Third-party bundling**:
   - Bundles rxjs, mitt, parsel-js from `third_party/` into both ESM and CJS
   - Embeds license headers

5. **Generated files** (`hereby generate`):
   - `src/generated/version.ts` - From package.json version
   - `src/generated/injected.ts` - Minified browser injection code
   - `lib/esm/package.json` - ESM marker file

### Testing Infrastructure

- **Integration tests**: `test/src/` (repository root) using Mocha + Expect
  - Uses custom test runner: `tools/mocha-runner`
  - Test expectations: `test/TestExpectations.json` (marks known failures/skips)
  - Test suites: `test/TestSuites.json` (defines test configurations)
  - Test helpers: `test/src/mocha-utils.ts` provides `getTestState()`, setup hooks

- **Unit tests**: Colocated with source files (`*.test.ts`)
  - Run with Node's built-in test runner
  - Located alongside implementation files
  - Execute via `npm run unit`

### Key Patterns

1. **Dual protocol support**: Most API classes have both CDP and BiDi implementations
   - Abstract base in `src/api/`
   - CDP implementation in `src/cdp/`
   - BiDi implementation in `src/bidi/`

2. **Event-driven architecture**: Heavy use of RxJS observables (bundled from `third_party/rxjs`)

3. **Query handlers**: Extensible system for element selection
   - Built-in: ARIA, CSS, XPath, text, Pierce (shadow DOM)
   - Custom handlers can be registered

4. **Locators**: Modern API for element interactions with auto-waiting and retries

## Development Workflow

### Making changes

1. Modify source in `src/`
2. Build: `npm run build --workspace puppeteer-core`
3. Run tests: `npm test` (from repo root) or `npm run unit` (for unit tests)
4. For specific test: Add `it.only()` in test file
5. Format code: `npm run format`

### Running single test file

Edit test file and use `it.only()`:
```typescript
it.only('should work', async function() {
  const {server, page} = await getTestState();
  // test code
});
```

Then run: `npm run build --workspace @puppeteer-test/test && npm test`

### Debugging tests in VSCode

1. Copy `.vscode/launch.template.json` to `.vscode/launch.json`
2. Build tests: `npm run build --workspace @puppeteer-test/test`
3. Use VSCode debugger

### Updating test expectations

If tests fail on specific platforms/configurations, update `test/TestExpectations.json` instead of changing test code.

## Documentation

- TSDoc comments are required for all public APIs
- Must include `@public` or `@internal` tag
- Keep lines ≤90 characters
- Docs auto-generated via `npm run docs` using API Extractor
- Never edit `docs/api/*.md` manually (auto-generated)

## Commit Conventions

Follow Conventional Commits format:
```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

For breaking changes, include `BREAKING CHANGE:` in footer.

Example: `feat(chrome): roll to Chrome 90.0.4427.0`

## Package Exports

- Main CJS: `lib/cjs/puppeteer/puppeteer-core.js`
- Main ESM: `lib/esm/puppeteer/puppeteer-core.js`
- Browser: `lib/esm/puppeteer/puppeteer-core-browser.js`
- Types: `lib/types.d.ts`
- Internal exports: `internal/*` maps to `lib/{esm,cjs}/puppeteer/*`

## Dependencies

- **Runtime**: `@puppeteer/browsers`, `chromium-bidi`, `debug`, `devtools-protocol`, `ws`, `webdriver-bidi-protocol`
- **Bundled third-party**: rxjs, mitt, parsel-js (in `third_party/`)
- Minimum Node.js: 18

## Notes

- This is a private fork published as `@codebyai/puppeteer-core`
- Build artifacts in `lib/` are gitignored
- Generated files in `src/generated/` are created during build
- Wire dependencies carefully - wireit handles build orchestration