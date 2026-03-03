# Upgrade & Quality-of-Life Ideas

Ranked by impact-to-effort ratio. Grounded in the actual codebase.

---

## 1. Prefab Ancestry Resolver

**What**: When `prefab_create` generates an `.et` file, automatically look up the base game prefab via `asset_search`, read it with `game_read`, and pre-populate components/properties that already exist on the parent — so the generated prefab is a proper *delta* rather than a blank slate that overwrites inherited structure.

**Why**: Right now `prefab_create` generates prefabs from hardcoded component lists per type (character, vehicle, weapon, etc.) in `src/templates/prefab.ts:40-130`. These templates don't know what the parent prefab already provides. Users end up with prefabs that either duplicate parent components or miss required ones, causing silent failures in Workbench. The `create-mod` prompt even has to warn about MeshObject being invisible (line 103-116) — a problem that wouldn't exist if the tool just *read* the parent first.

**Where**: `src/templates/prefab.ts`, `src/tools/prefab-create.ts`, uses existing `asset_search` + `game_read` + `enfusion-text.ts` parser

**Effort**: M

**Category**: Missing Obvious

---

## 2. Validation-Driven Fix Suggestions

**What**: Extend `mod_validate` to return machine-actionable fix objects alongside each `ValidationIssue` — e.g., `{ fix: "move", from: "Scripts/MySrc.c", to: "Scripts/Game/MySrc.c" }` — so the LLM (or a future `mod_fix` tool) can apply fixes automatically instead of requiring the user to interpret text warnings.

**Why**: `mod_validate` (src/tools/mod-validate.ts) currently returns human-readable strings like `"Scripts/MySrc.c: Script is outside a valid module folder — it will be silently ignored"`. Claude has to parse these strings, figure out the fix, then call `project_write` or `project_read` to move files. Every validation check in `checkStructure`, `checkScripts`, `checkGproj`, `checkReferences`, and `checkNaming` has an obvious programmatic fix that could be expressed as a structured action.

**Where**: `src/tools/mod-validate.ts` (extend `ValidationIssue` interface at line 9), new `src/tools/mod-fix.ts`

**Effort**: M

**Category**: Power Feature

---

## 3. Cross-Index "Used By" Backlinks

**What**: Add reverse-lookup to `SearchEngine`: given a class, find all classes that *reference* it (as parent, parameter type, return type, or property type). Expose via a new `usedBy` field in `api_search` results and the `enfusion://class` resource.

**Why**: The search engine (`src/index/search-engine.ts`) indexes forward relationships — parents, children, methods, properties — but has no reverse index. When Claude is writing a modded class and needs to know "what calls this method?" or "what other components use this class as a property type?", it has to do speculative `api_search` queries. A reverse index would make composition discovery instant, especially for the 8,693 indexed classes.

**Where**: `src/index/search-engine.ts` (new `Map<string, string[]>` for usedBy, built in `load()` at line 48), `src/tools/api-search.ts` (format output), `src/resources/class-resource.ts`

**Effort**: M

**Category**: Power Feature

---

## 4. Smart Script Patching (Read-Modify-Write)

**What**: Add a `script_patch` tool that reads an existing `.c` file, parses it structurally (class name, methods, member variables), and supports targeted modifications — add method, modify method body, add member variable, add import — without rewriting the entire file.

**Why**: Currently modifying scripts requires `project_read` → Claude manually edits the full text → `project_write` the entire file. This is error-prone for large scripts and uses excessive context. The codebase already has the regex patterns for parsing class declarations and method signatures in `mod-validate.ts:132-133` and `templates/script.ts:329-354` — they just aren't composed into a structural editor.

**Where**: New `src/tools/script-patch.ts`, reuses parsing from `src/templates/script.ts` (extractMethodName, extractParamNames, stripOverride)

**Effort**: L

**Category**: Power Feature

---

## 5. Workbench Connection Health in Tool Descriptions

**What**: Make `wb_state` a lightweight pre-check that other `wb_*` tools can reference. Add a `connectionStatus` field to all `wb_*` tool responses that shows whether Workbench is connected, in play mode, or in edit mode — so Claude doesn't blindly call `wb_play` when already in play mode or call `wb_entity_create` when in play mode (which fails).

**Why**: The 20+ Workbench tools in `src/tools/wb-*.ts` all independently call `client.call()` which auto-launches Workbench on `CONNECTION_REFUSED`. But there's no state awareness between tool calls. `wb_play` while already in play mode, or `wb_entity_modify` while in play mode, causes errors that Claude has to recover from. The `wb_state` tool (src/tools/wb-state.ts) already returns the mode — it just isn't consulted automatically.

**Where**: `src/workbench/client.ts` (add cached state after each `rawCall`), all `src/tools/wb-*.ts` files

**Effort**: S

**Category**: UX Polish

---

## 6. Pattern Composition (Mix-and-Match)

**What**: Allow `mod_create` to accept an array of patterns instead of a single one, merging their scripts/prefabs/configs. A "game-mode + custom-faction + hud-widget" combo should produce one scaffold with all three systems wired together.

**Why**: `mod_create` (`src/tools/mod-create.ts:88`) accepts one `pattern` string. But real mods are compositions — a game mode usually needs a faction, a HUD, and a spawn system. The pattern library (`src/patterns/loader.ts`) already loads each pattern independently. The `mod_create` tool just needs to iterate over an array instead of a single pattern and handle name collisions in the `{PREFIX}` replacement.

**Where**: `src/tools/mod-create.ts` (change `pattern: z.string().optional()` to `patterns: z.array(z.string()).optional()`), `src/patterns/loader.ts`

**Effort**: S

**Category**: Clever Combo

---

## 7. Incremental Asset Index

**What**: Replace the session-scoped module-level cache for asset search (`let cachedIndex: AssetEntry[] | null = null` at `src/tools/asset-search.ts:29`) with a persistent on-disk index that uses file modification timestamps to incrementally update only changed files.

**Why**: The asset index rebuilds by walking the entire game directory + all `.pak` files on first search each session. For a full Arma Reforger install, this is thousands of files. The log line at `asset-search.ts:83` (`Asset index built: ${entries.length} files in ${elapsed}ms`) shows this is a blocking cold-start cost. A persisted index with mtime-based invalidation would make first searches instant on subsequent sessions.

**Where**: `src/tools/asset-search.ts` (replace `cachedIndex`/`cachedBasePath` with file-backed cache), possibly new `src/utils/cache.ts`

**Effort**: M

**Category**: Developer Experience

---

## 8. Dry-Run Mode for Mutation Tools

**What**: Add a `dryRun: boolean` parameter to `mod_create`, `script_create`, `prefab_create`, `config_create`, `layout_create`, and `project_write`. When true, return what *would* be created/modified without writing to disk.

**Why**: All creation tools (`script_create` at `src/tools/script-create.ts:89`, `mod_create` at `src/tools/mod-create.ts:142`, `prefab_create`, etc.) immediately write to disk via `writeFileSync`. There's no way for Claude to preview what it's about to generate and course-correct before committing. The `script_create` tool already has a partial pattern — when a file already exists (line 78), it returns the generated code without writing. Dry-run would generalize this.

**Where**: All tools in `src/tools/{mod,script,prefab,config,layout}-create.ts` and `src/tools/project-write.ts`

**Effort**: S

**Category**: UX Polish

---

## 9. Fuzzy Search with Typo Tolerance

**What**: Add Levenshtein distance scoring to the search engine so queries like "ScriptCompnent" or "GetPositon" still find results. Currently the search only does exact/prefix/substring matching.

**Why**: Every search method in `SearchEngine` (src/index/search-engine.ts:140-326) uses strict `===`, `startsWith`, and `includes` checks. The `create-mod` prompt (src/prompts/create-mod.ts:91-101) specifically warns "NEVER guess API method names" because a typo returns zero results. Fuzzy matching with a score penalty (e.g., edit distance 1 = score 40, edit distance 2 = score 20) would recover from common typos without returning garbage.

**Where**: `src/index/search-engine.ts` (add `levenshtein()` helper, integrate into `searchClasses`, `searchMethods`, `searchEnums`, `searchProperties` scoring)

**Effort**: S

**Category**: UX Polish

---

## 10. Workbench Console Log Capture

**What**: Add a tool that reads Workbench's script compilation output and runtime `Print()` log, so when `wb_play` fails to compile, Claude can read the actual error messages instead of guessing what went wrong.

**Why**: The `create-mod` prompt workflow (src/prompts/create-mod.ts:143) says "If compilation failed (errors in the Workbench console), fix with project_write." But Claude has no way to *see* those errors — it can only infer compilation failed from the lack of a success response. The Workbench NET API handler scripts (`mod/Scripts/WorkbenchGame/EnfusionMCP/`) already run inside Workbench's scripting environment and could potentially capture `Log.Info`/`Log.Error` output.

**Where**: New `mod/Scripts/WorkbenchGame/EnfusionMCP/EMCP_WB_GetLog.c`, new `src/tools/wb-log.ts`

**Effort**: M

**Category**: Missing Obvious

---

## 11. Class Hierarchy Visualization

**What**: Add a `tree` output mode to `api_search` that renders the inheritance chain as an ASCII tree, showing a class's ancestors (up to root) and immediate descendants with their key methods — a "class at a glance" view.

**Why**: `SearchEngine.getClassTree()` and `getInheritanceChain()` (src/index/search-engine.ts:368-423) already compute full ancestor/descendant chains. But `api_search` (src/tools/api-search.ts) formats results as flat markdown lists. When Claude is deciding which class to extend, it needs to see the full hierarchy at once — "SCR_BaseGameMode → SCR_BaseGameMode → GameMode → BaseGameMode → GenericEntity" — not just the immediate parent.

**Where**: `src/tools/api-search.ts` (new `format` parameter, new `formatClassTree()` function), uses existing `SearchEngine.getInheritanceChain()` and `getClassTree()`

**Effort**: S

**Category**: Developer Experience

---

## 12. MODPLAN as Structured Data

**What**: Replace the freeform markdown MODPLAN.md with a structured JSON/YAML format that tools can read and write programmatically. Add a `mod_plan` tool that can query plan status, mark phases complete, and generate the next phase's task list.

**Why**: Both prompts (`create-mod.ts:46-75` and `modify-mod.ts:34-36`) rely on MODPLAN.md as the handoff document between sessions. But it's freeform markdown that Claude has to parse with `project_read` and rewrite with `project_write`. A structured format with typed fields (phases, status, files, architecture notes) would make the handoff reliable instead of hoping Claude correctly parses the previous Claude's markdown.

**Where**: New `src/tools/mod-plan.ts`, updates to `src/prompts/create-mod.ts` and `src/prompts/modify-mod.ts`

**Effort**: M

**Category**: Clever Combo

---

## 13. Duplicate Code Consolidation: project_browse & game_browse

**What**: Extract the shared `listDirectory()`, `FILE_TYPE_MAP`, `formatSize()`, and `DirEntry` logic into a common utility. Both `project-browse.ts` and `game-browse.ts` have near-identical implementations of these (compare `src/tools/project-browse.ts:8-91` with `src/tools/game-browse.ts:9-81`).

**Why**: Two files, ~80 lines each, with copy-pasted directory listing logic. `game-browse.ts` adds `.emat` and `.sounds` to its `FILE_TYPE_MAP` but `project-browse.ts` doesn't — an inconsistency that means project browsing won't label material files. Any fix to one file has to be manually mirrored to the other.

**Where**: New `src/utils/dir-listing.ts`, refactor `src/tools/project-browse.ts` and `src/tools/game-browse.ts`

**Effort**: S

**Category**: Developer Experience

---

## 14. Component Discovery Tool

**What**: Add a `component_search` tool (or mode in `api_search`) that specifically searches `ScriptComponent` descendants, filterable by what they attach to (entities, vehicles, characters) and what events they handle.

**Why**: `SearchEngine.getComponents()` (src/index/search-engine.ts:479-487) already walks all ScriptComponent descendants. But it's only used internally — there's no tool that exposes it. The most common modding question is "what component do I attach to do X?" and currently Claude has to do multiple `api_search` queries and hope the right component surfaces. A dedicated component browser with event/capability filtering would make prefab composition much faster.

**Where**: `src/index/search-engine.ts` (expose `getComponents()` with filtering), new section in `src/tools/api-search.ts` or new `src/tools/component-search.ts`

**Effort**: S

**Category**: Missing Obvious

---

## 15. Config Validation (Beyond Parse Check)

**What**: Extend config validation in `mod_validate` to check that class names referenced in `.conf` files actually exist in the API index, and that required fields for known config types (faction configs, entity catalogs, etc.) are present.

**Why**: `checkConfigs()` in `src/tools/mod-validate.ts:173-191` only verifies that `.conf` files parse correctly via `parse(content)`. It doesn't check whether the class names and resource paths inside them are valid. A faction config referencing `"SCR_FactionManager"` (which doesn't exist — it's `SCR_Faction`) would pass validation but fail at runtime. The search engine's `hasClass()` method is already used for script reference checking in `checkReferences()` — it just needs to be applied to configs too.

**Where**: `src/tools/mod-validate.ts` (`checkConfigs` function at line 173), uses existing `SearchEngine.hasClass()`

**Effort**: S

**Category**: UX Polish
