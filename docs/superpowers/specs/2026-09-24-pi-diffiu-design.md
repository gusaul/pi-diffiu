# Pi Diffiu Design

## Purpose

Pi Diffiu is a public Pi extension that provides an interactive, side-by-side Git working-tree and staged-diff viewer. Its differentiator is structured Tree-sitter syntax highlighting that distinguishes common syntactic entities (including member properties, methods, parameters, types, and variables) without relying on a user’s local Neovim configuration.

## Scope

Version 1 ships as the public MIT-licensed npm package `@gusaul/pi-diffiu` and Pi package `gusaul/pi-diffiu`. It supports Go, JavaScript, TypeScript, TSX, JSON, Python, and Rust with bundled Tree-sitter grammar dependencies. Unsupported languages and parser failures retain Pi’s built-in `highlightCode()` fallback.

LSP semantic-token enrichment is explicitly out of scope for v1. Tree-sitter supplies syntax-level distinctions; LSP can be added later as an optional enhancement.

## Package and Distribution

The repository is a standard Pi package with its extension under `extensions/` and source code under `src/`. `package.json` declares the Pi extension in its `pi.extensions` manifest, declares Pi runtime packages as `peerDependencies` with `"*"`, and declares Tree-sitter and all shipped grammar packages as runtime `dependencies`.

The package is ESM TypeScript. A build step emits distributable JavaScript and declaration files under `dist/`. npm uses public scoped access and provenance when credentials permit. GitHub Actions runs install, typecheck, test, and package-content checks on supported Node versions.

## Syntax Pipeline

For every selected diff file and each side independently, the viewer reconstructs a virtual source document from the diff side’s displayed text. It preserves line order and uses blank lines for unavailable sides so parser line coordinates remain aligned with the display. It parses the complete virtual document once with the grammar selected by file path.

The syntax layer returns line-indexed spans:

```ts
export type SyntaxRole =
  | "comment" | "keyword" | "type" | "function" | "method"
  | "property" | "parameter" | "variable" | "string" | "number"
  | "operator" | "punctuation";

export type SyntaxSpan = {
  start: number;
  end: number;
  role: SyntaxRole;
};

export type HighlightedFile = SyntaxSpan[][];
```

The Tree-sitter walker maps grammar node types and parent context to roles. Parent-aware mapping classifies property/field identifiers and method declarations independently from ordinary identifiers. Child traversal avoids overlapping output: a recognized node creates one span and is not recursed into; structural nodes recurse into children.

The viewer preserves plain source chunks and `SyntaxSpan`s through wrapping. It clips and splits spans to each wrapped segment, applies role foreground ANSI styling only at render time, then layers line and changed-character diff backgrounds without discarding foreground styling. This replaces per-line `highlightCode()` calls.

## Compatibility and Fallback

The language registry maps paths to bundled grammars. It selects the TypeScript grammar for `.ts`, the TSX grammar for `.tsx`, and equivalents for the remaining v1 set. It accepts case-insensitive extensions. `Dockerfile`-like filename detection is not v1 scope.

If there is no registered grammar, the parser cannot load, or parsing throws, the caller calls Pi’s `getLanguageFromPath()`/`highlightCode()` fallback. The extension never blocks opening a diff because syntax enhancement fails.

## Viewer Behavior

`/diff` shows unstaged tracked changes plus untracked files. `/diff --staged` (and `--cached`) shows staged changes. Optional path arguments scope the Git commands. The overlay requires Pi fullscreen TUI, provides tree navigation and code scrolling by keyboard and mouse, displays old/new panes with line numbers and change markers, uses a sticky file header, and preserves the existing changed-character emphasis behavior.

## Testing

Tests use Node’s native test runner against compiled JavaScript. Unit tests cover language selection, parent-aware syntax classification in every supported grammar, complete-file multiline context, span clipping across wrapped cells, unsupported-file fallback, and parser-failure fallback. Parser integration tests use real grammars, never mocked token output.

A viewer-focused test verifies both sides receive complete virtual documents with aligned line coordinates, including additions/deletions. A smoke test exercises package installation from the packed tarball and confirms Pi can resolve the declared extension entrypoint.

## Documentation

README documents installation through `pi install npm:@gusaul/pi-diffiu`, manual local loading for development, commands and controls, supported languages, fallback behavior, requirements, and development/test commands. It also states that syntax classification is Tree-sitter syntax rather than LSP semantic-token highlighting.
