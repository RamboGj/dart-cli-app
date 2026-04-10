# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Dartpedia is a Dart CLI app that fetches articles from the Wikipedia API. It is structured as a Dart workspace with three packages.

## Commands

All commands should be run from the workspace root unless noted.

**Run the CLI:**
```sh
dart run cli/bin/cli.dart <command> [args]
# Examples:
dart run cli/bin/cli.dart search "dart programming"
dart run cli/bin/cli.dart search --im-feeling-lucky "cats"
dart run cli/bin/cli.dart article "Cat"
dart run cli/bin/cli.dart help
dart run cli/bin/cli.dart help -v
```

**Run all tests (workspace):**
```sh
dart test
```

**Run tests for a single package:**
```sh
dart test wikipedia/
dart test command_runner/
dart test cli/
```

**Run a single test file:**
```sh
dart test wikipedia/test/model_test.dart
```

**Analyze (lint):**
```sh
dart analyze
```

## Architecture

The workspace has three packages with a clear dependency hierarchy:

```
cli  →  command_runner  (custom CLI framework)
cli  →  wikipedia       (Wikipedia API client)
```

### `wikipedia/` — API client library
- `lib/src/api/` — three top-level async functions: `search()`, `getArticleByTitle()`, `getRandomArticleSummary()`, `getArticleSummaryByTitle()`. Each creates its own `http.Client`, calls the Wikipedia REST or MediaWiki API, and deserializes the response.
- `lib/src/model/` — plain Dart model classes (`Article`, `SearchResults`, `Summary`, `TitleSet`) with hand-written `fromJson`/`toJson`. `Article.listFromJson` uses Dart pattern matching to navigate the nested MediaWiki JSON structure.
- Tests use fixture JSON files in `test/test_data/` to test deserialization without network calls.

### `command_runner/` — custom CLI framework (no third-party deps)
- `CommandRunner` parses `List<String>` input, dispatches to registered `Command` objects, and invokes optional `onOutput`/`onError` callbacks.
- `Command` (abstract) declares `name`, `description`, `run(ArgResults)`, and supports `addFlag`/`addOption` for defining accepted arguments.
- `ArgResults` carries the resolved command, a single positional `commandArg`, and a `Map<Option, Object?>` for named options/flags.
- `HelpCommand` is not auto-added; callers must register it explicitly.

### `cli/` — executable that wires everything together
- `bin/cli.dart` — entry point: instantiates `CommandRunner` with file-based error logging, then registers `HelpCommand`, `SearchCommand`, and `GetArticleCommand`.
- `lib/src/commands/` — two commands: `search` (supports `--im-feeling-lucky` flag) and `article` (defaults to `"cat"` if no arg given).
- `lib/src/logger.dart` — `initFileLogger(name)` creates a date-stamped log file under `cli/logs/` using the `logging` package; only errors/warnings are written there at runtime.

## Key Conventions

- The `wikipedia` API functions throw `HttpException` (non-200 response) or `FormatException` (bad JSON). Commands catch both and log them via the injected `Logger` rather than crashing.
- `Command.run()` returns `FutureOr<Object?>`. The runner calls `.toString()` on the result before passing it to `onOutput`.
- Each API function opens and closes its own `http.Client` in a `try/finally` block.
