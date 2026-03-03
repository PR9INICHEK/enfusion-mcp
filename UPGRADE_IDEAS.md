# Enfusion MCP — Upgrade Ideas

Ranked by impact-to-effort ratio. Based on full codebase audit of tools, data quality, search engine, prompt engineering, and scraper pipeline.

**Key finding**: 85% of Arma class briefs are empty, 88% of methods lack descriptions, hierarchy.json is empty, and 0 classes have scraped enum values. The LLM is operating with very sparse context and high hallucination surface area.

---

## 1. Automatic Inheritance Context on Class Lookup

**What**: When `api_search` returns a class result, automatically include method signatures from the first 2-3 parent classes inline. `getInheritedMembers()` already exists in `SearchEngine` but is **never called from `api_search`** — it's only wired to the `enfusion://class/{name}` MCP resource, which most LLMs don't proactively read.

**Why**: The LLM sees `SCR_BaseGameMode` inherits from `BaseGameMode` but never sees `BaseGameMode`'s overridable methods unless it makes a second lookup. This is the #1 cause of incorrect override signatures — the LLM guesses what the parent exposes. The data is already indexed; it just isn't surfaced in the primary search tool.

**Where**: `src/tools/api-search.ts` (`formatClassResult`), `src/index/search-engine.ts` (`getInheritedMembers` already exists)

**Effort**: S

**Category**: Hallucination Prevention

---

## 2. Enum-Aware Search

**What**: Enfusion represents many enums as classes with static properties (189 classes have >3 properties and no methods — these are enum-like structs/data classes). The current `searchEnums` only checks `cls.enums[]` which is **empty for all 8,693 classes** (0 enums scraped). Add a heuristic that surfaces these property-only classes when `type: "enum"` is searched, and tag them as enum-like in results.

**Why**: When a modder asks "what are the damage types?" or "what EResourceType values exist?", `api_search(query: "EDamageType", type: "enum")` returns 0 results. The data exists as class properties but isn't categorized as enum-like. The LLM then invents enum values.

**Where**: `src/index/search-engine.ts` (`searchEnums`, `load`), `src/tools/api-search.ts` (`formatEnumResult`)

**Effort**: S

**Category**: Context Quality

---

## 3. Related Classes in Search Results

**What**: When a class lookup matches (e.g., `SCR_CharacterDamageManagerComponent`), also include a compact one-liner for each sibling class in the same API group — just name + brief. This gives the LLM discovery paths without requiring follow-up searches.

**Why**: Modders often ask "how do I do X" and the LLM finds one class, but the real answer involves a related class it doesn't know to search for (e.g., finding `SCR_DamageManagerComponent` but not `SCR_CharacterDamageManagerComponent`). Groups data exists (157 groups) but is only exposed as an MCP resource, never surfaced in tool search results.

**Where**: `src/tools/api-search.ts` (`formatClassResult`), `src/index/search-engine.ts` (groups already indexed)

**Effort**: S

**Category**: Context Quality

---

## 4. Wiki Full-Text Retrieval

**What**: Wiki search truncates results to 2,000 characters (`MAX_LENGTH = 2000` in `wiki-search.ts`). Add a `wiki_read` tool (or a `fullText` parameter on `wiki_search`) that returns the full page content by title, without truncation. Alternatively, raise the limit substantially when only 1 result matches.

**Why**: The average wiki page is 8,666 chars. Truncation at 2,000 cuts off the actual tutorial code examples — the most valuable part for the LLM. The modder asks "how does replication work?" and gets the intro paragraph but not the example code.

**Where**: `src/tools/wiki-search.ts` (add `fullText` param or separate `wiki_read` tool)

**Effort**: S

**Category**: Context Quality

---

## 5. Method Signature Validator Tool

**What**: Add a `script_check` tool that takes a class name + method signature and verifies it exists in the API index, returning the correct signature if there's a close match (Levenshtein/fuzzy). Gives the LLM a lightweight "did I get this right?" check without re-searching the entire class.

**Why**: 88% of methods lack descriptions. The LLM writes `override void OnPlayerSpawned(int playerId, IEntity entity)` but the real signature has `IEntity controlledEntity`. The prompt says "verify every method" but there's no ergonomic single-method verification tool — `api_search` returns full class dumps, making verification expensive in tokens.

**Where**: New file `src/tools/script-check.ts`, `src/index/search-engine.ts` (add fuzzy signature match)

**Effort**: M

**Category**: Hallucination Prevention

---

## 6. Example Code Snippets in Patterns

**What**: Expand pattern JSON files to include `codeExamples` — short, complete, working Enfusion code snippets (3-15 lines) for common operations within that pattern. Inject these into the create-mod prompt context when the pattern is selected.

**Why**: The current 10 patterns define class names and method stubs but give zero example logic. The LLM then writes the body of `override void OnPlayerConnected(int playerId)` by guessing at Enfusion API calls. Having a known-working 5-line snippet for "spawn a character at a spawn point" eliminates entire categories of hallucination.

**Where**: `data/patterns/*.json` (add `codeExamples` field), `src/patterns/loader.ts` (type update), `src/prompts/create-mod.ts` (inject into context)

**Effort**: M (writing correct snippets requires Enfusion domain knowledge)

**Category**: Hallucination Prevention

---

## 7. Base Game Prefab Introspection

**What**: Add an `asset_inspect` tool that reads a base game `.et` prefab from PAK, parses it with the existing Enfusion text parser, and returns a structured summary: entity type, components list, key property values, parent prefab reference. No raw dump — a formatted component manifest.

**Why**: When the LLM needs to inherit from or reference a base game prefab, it currently either guesses at the component structure or must use `game_read` which dumps raw Enfusion text that's hard to reason about. A structured summary means the LLM can see "this prefab has MeshObject, RigidBody, and SCR_InteractionHandlerComponent" and correctly compose its own prefab.

**Where**: New file `src/tools/asset-inspect.ts`, uses `src/pak/vfs.ts` + `src/formats/enfusion-text.ts`

**Effort**: M

**Category**: Modder Workflow

---

## 8. `script_create` Should Auto-Fetch Parent Methods

**What**: When `script_create` is called with a `parentClass`, automatically look up the parent class in the search engine and populate the method stubs with the actual overridable methods (virtual/protected) from the parent, instead of relying on hardcoded lists like `GAMEMODE_METHODS` and `COMPONENT_METHODS`.

**Why**: The hardcoded method lists are incomplete and will rot as the game updates. The API data already has the full method lists. A modder who says "create a component extending SCR_InventoryStorageManagerComponent" gets stubs for `EOnInit`/`OnPostInit` — generic ScriptComponent methods — instead of the actual overridable inventory methods. Currently `script_create` doesn't take a `SearchEngine` dependency at all.

**Where**: `src/tools/script-create.ts`, `src/templates/script.ts`, `src/server.ts` (inject SearchEngine dependency)

**Effort**: M

**Category**: Composability

---

## 9. Compilation Error Feedback Loop

**What**: After `wb_play` or `wb_reload` fails due to compilation errors, parse the Workbench response to extract file path, line number, and error message, then automatically `project_read` the failing file and present the error in context with surrounding code lines.

**Why**: Compilation errors from Workbench come back as opaque strings. The LLM can't see what line failed or what the actual error was, so it makes blind fixes. Auto-extracting the error location and reading the surrounding code would let it fix issues in one pass instead of 3-4 round trips. The prompt just says "fix with project_write" but gives no structured data to work from.

**Where**: `src/tools/wb-script-editor.ts` or new `src/tools/wb-compile-errors.ts`, `src/workbench/client.ts`

**Effort**: M

**Category**: Modder Workflow

---

## 10. Semantic Search via Trigram Index

**What**: Replace the pure substring/prefix scoring in `SearchEngine.searchClasses` (and methods/properties) with trigram matching on names. The current search requires the modder's exact terminology — `api_search("get health")` won't find `GetDamageStateThreshold` or `GetHealthScaled`.

**Why**: Modders describe what they want functionally ("get health", "move entity", "spawn AI"), but 88% of method descriptions are empty, so description-based matching fails. Substring matching against method *names* only works if you already know the Enfusion naming convention. Even basic trigram matching would catch `GetHealth` → `GetHealthScaled` or `Damage` → `DamageManagerComponent`.

**Where**: `src/index/search-engine.ts` (all search methods)

**Effort**: M

**Category**: Context Quality

---

## 11. Cross-Reference Validation on Write

**What**: When `project_write` writes a `.c` script, run a lightweight static check: extract class references and method calls via regex, verify them against the API index. Return warnings inline ("Warning: `HitZone.SetHealth()` not found in API — did you mean `SCR_CharacterDamageManagerComponent.FullHeal()`?").

**Why**: The prompt instructs "verify every method with api_search" but the LLM often doesn't. Doing it automatically on write catches hallucinated API calls before the modder even tries to compile. The existing `mod_validate` tool only checks parent class references (`checkReferences`), not method calls within scripts.

**Where**: `src/tools/project-write.ts`, `src/index/search-engine.ts`

**Effort**: L

**Category**: Hallucination Prevention

---

## 12. Common Pitfalls Context Injection

**What**: Maintain a structured list of known Enfusion gotchas (e.g., "scripts in Scripts/GameLib/ won't load for mods", "modded classes are global", "EntityEvent.FRAME requires SetEventMask", "`ref` keyword for reference types") and inject the relevant ones based on what the LLM is creating (detected from script_create type, api_search queries, etc.).

**Why**: The create-mod prompt already includes several hardcoded warnings (mesh requirement, API verification rule). But these are static. A data-driven pitfall system could match against context — if the LLM is creating an entity with `EOnFrame`, auto-inject the `SetEventMask` requirement. This catches the long tail of Enfusion-specific traps that aren't in the docs.

**Where**: New `data/pitfalls.json`, `src/tools/script-create.ts`, `src/prompts/create-mod.ts`

**Effort**: M

**Category**: Hallucination Prevention

---

## 13. Diff-Based Project Write

**What**: Add a `project_patch` tool that accepts a file path and a diff (old lines → new lines), rather than requiring the LLM to re-emit the entire file via `project_write`. This mirrors how code editors work and reduces token waste.

**Why**: For any script modification, the LLM currently must `project_read` (consuming tokens for the full file), then `project_write` the entire file again (emitting all tokens). For a 200-line script where it's changing 5 lines, this is 95% waste. A patch-based tool would let it emit only the changed lines, saving tokens and reducing error surface.

**Where**: New tool `src/tools/project-patch.ts` or extend `src/tools/project-write.ts`

**Effort**: M

**Category**: Modder Workflow

---

## 14. Component Compatibility Matrix

**What**: Build a mapping of which components commonly co-occur on entity types in the base game. Add a `type: "components"` search mode to `api_search` that, given an entity type (e.g., `GenericEntity`, `SCR_ChimeraCharacter`), returns which components are typically attached. Derive this by scanning base game `.et` files during asset indexing.

**Why**: Enfusion components have implicit compatibility rules (some require others, some conflict). The LLM frequently attaches incompatible components — e.g., putting `WeaponComponent` on a `GenericEntity` without the required `BaseWeaponManagerComponent`. A compatibility matrix derived from base game prefabs would prevent this.

**Where**: `src/tools/asset-search.ts` (extend indexing), new data structure in `src/index/`

**Effort**: L

**Category**: Hallucination Prevention

---

## 15. Multi-Mod Workspace Support

**What**: Currently `ENFUSION_PROJECT_PATH` points to a single addon. Support a workspace model where the path points to the `addons/` directory and tools accept a `modName` parameter to select which addon to operate on. `project_browse` would list all addons in the workspace, and creation tools would scope to the selected one.

**Why**: Many modders work on multiple mods simultaneously. The current single-path design means switching mods requires restarting the MCP server or passing `projectPath` on every call. `mod_create` already creates subdirectories under `projectPath`, but other tools don't navigate them well.

**Where**: `src/config.ts`, all tools that use `config.projectPath`

**Effort**: M

**Category**: Modder Workflow

---

## Data Quality Summary

| Metric | Value | Concern |
|--------|-------|---------|
| Arma classes indexed | 7,881 | Good coverage |
| Enfusion classes indexed | 812 | Good coverage |
| Empty class briefs | 85% | LLM has minimal context for class purpose |
| Empty method descriptions | 88% | LLM must guess method behavior from name alone |
| Scraped enum values | 0 | Enum search returns nothing; scraper misses Enfusion enum format |
| hierarchy.json entries | 0 | Empty file; inheritance relies solely on per-class parents[] |
| Wiki pages | 258 | Decent, but truncated to 2K chars in search results |
| Avg wiki page length | 8,666 chars | 75% of content is lost to truncation |
