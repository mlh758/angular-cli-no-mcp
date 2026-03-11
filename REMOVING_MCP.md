# Removing MCP from Angular CLI

This document records every change needed to strip Model Context Protocol (MCP)
features from the Angular CLI. Use it as a checklist when rebasing onto a newer
upstream version.

---

## Why

The `@modelcontextprotocol/sdk` dependency pulls in a Hono-based Node HTTP server
and associated packages (express, jose, cors, etc.) that inflate the install size
and have produced CVEs causing compliance issues.

---

## Files to Edit

### 1. `packages/angular/cli/package.json`

Remove the `@modelcontextprotocol/sdk` entry from `dependencies`:

```diff
-    "@modelcontextprotocol/sdk": "1.27.1",
```

### 2. `packages/angular/cli/src/commands/command-config.ts`

Remove `'mcp'` from the `CommandNames` type union:

```diff
   | 'make-this-awesome'
-  | 'mcp'
   | 'new'
```

Remove the `'mcp'` entry from the `RootCommands` object:

```diff
   'make-this-awesome': {
     factory: () => import('./make-this-awesome/cli'),
   },
-  'mcp': {
-    factory: () => import('./mcp/cli'),
-  },
   'new': {
```

### 3. `packages/angular/cli/BUILD.bazel`

**Remove the `angular_best_practices` genrule** (copies `best-practices.md` into
the MCP resources directory):

```diff
-genrule(
-    name = "angular_best_practices",
-    srcs = [
-        "//:node_modules/@angular/core/dir",
-    ],
-    outs = ["src/commands/mcp/resources/best-practices.md"],
-    cmd = """
-        cp "$(location //:node_modules/@angular/core/dir)/resources/best-practices.md" $@
-    """,
-)
```

**Remove `:angular_best_practices` from `RUNTIME_ASSETS`:**

```diff
     "//packages/angular/cli:lib/code-examples.db",
-    ":angular_best_practices",
 ]
```

**Remove `:node_modules/@modelcontextprotocol/sdk` from the main `ts_project` deps:**

```diff
-        ":node_modules/@modelcontextprotocol/sdk",
```

**Remove `:node_modules/@modelcontextprotocol/sdk` from the test `ts_project` deps:**

```diff
-        ":node_modules/@modelcontextprotocol/sdk",
```

### 4. `packages/schematics/angular/workspace/files/__dot__gitignore.template`

Remove the line that un-ignores `.vscode/mcp.json`:

```diff
 !.vscode/extensions.json
-!.vscode/mcp.json
 .history/*
```

### 5. `packages/schematics/angular/workspace/index_spec.ts`

Remove `'/.vscode/mcp.json'` from both test assertions (the "should create all
files" and "should create correct files when using minimal" tests):

```diff
         '/.vscode/extensions.json',
         '/.vscode/launch.json',
-        '/.vscode/mcp.json',
         '/.vscode/tasks.json',
```

---

## Files/Directories to Delete

| Path                                                                          | What it is                                                                                                            |
| ----------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `packages/angular/cli/src/commands/mcp/`                                      | Entire MCP command implementation (~40+ source files, tools, resources, tests, testing utils)                         |
| `packages/schematics/angular/workspace/files/__dot__vscode/mcp.json.template` | VS Code MCP server config template generated in new workspaces                                                        |
| `tests/e2e/tests/mcp/`                                                        | E2E tests for MCP tools (4 files: `registers-tools.ts`, `best-practices.ts`, `ai-tutor.ts`, `find-examples-basic.ts`) |

Delete them all:

```bash
rm -rf packages/angular/cli/src/commands/mcp
rm -f  packages/schematics/angular/workspace/files/__dot__vscode/mcp.json.template
rm -rf tests/e2e/tests/mcp
```

---

## Regenerate the Lockfile

After making the above changes:

```bash
pnpm install --no-frozen-lockfile
```

This removes `@modelcontextprotocol/sdk` and its transitive dependencies
(`@hono/node-server`, `hono`, `jose`, `pkce-challenge`, `eventsource`,
`express-rate-limit`, `json-schema-typed`, `zod-to-json-schema`, etc.) from
`pnpm-lock.yaml`.

---

## Verification

Confirm no MCP references remain in source/config files:

```bash
grep -ri --include='*.ts' --include='*.js' --include='*.json' \
  --include='*.mjs' --include='*.template' \
  'mcp\|modelcontextprotocol' \
  packages/ tests/ modules/ \
  --exclude-dir=node_modules -l
```

This should return no results (exit code 1). References in `CHANGELOG.md` and
`pnpm-lock.yaml` (from other packages' transitive deps like `@angular/ng-dev`)
are expected and harmless.

---

## Summary of Removed Transitive Dependencies

Removing `@modelcontextprotocol/sdk` eliminates these packages from the CLI's
dependency tree:

- `@hono/node-server`
- `hono` (the CVE source)
- `jose`
- `pkce-challenge`
- `eventsource` / `eventsource-parser`
- `express-rate-limit` (the MCP SDK's own copy)
- `json-schema-typed`
- `zod-to-json-schema`
- `cors`, `raw-body`, `content-type` (as MCP-only deps)
