# Dynamic-Dispatch Coverage Playbook

**Audience:** a Claude agent continuing this work.
**Mission:** systematically close static-extraction coverage holes for **dynamic
dispatch** across **every language and framework codegraph supports**, and validate
each one the same way, so cross-symbol *flows* exist in the graph everywhere.

> This is the top-level playbook. The deep design for one mechanism (the callback
> synthesizer) is in [`callback-edge-synthesis.md`](./callback-edge-synthesis.md).
> Full investigation context + findings: auto-memory `project_codegraph_read_displacement`.

---

## 1. The goal (why this matters)

codegraph's value is being **the map** — answering structural/flow questions
(`trace`, `impact`, callers, "how does X reach Y") that grep/Read cannot. Agents
will use codegraph instead of Read **only when it is sufficient**. We proved
empirically (see memory) that the lever for sufficiency is **coverage**, not
prompting/hooks/new-tools: when a flow is missing from the graph, the agent reads
the files to reconstruct it; when the flow *is* in the graph, the agent can answer
completely without reading.

**Validated end-to-end on excalidraw:** after closing the update-flow hole, 2/3
headless agent runs answered the "how does an update reach the screen" question with
**Read 0 and a complete answer** — impossible before, because the key edge wasn't in
the graph. (Caveat: coverage *enables* the no-read path; agent confirm-by-reading
variance means it doesn't *force* it. Completeness improves unconditionally.)

The mission is to make that true for **all** languages/frameworks.

---

## 2. The problem class: dynamic dispatch

Static tree-sitter extraction captures explicit calls (`foo()`, `this.bar()`). It
**misses** any call whose target is computed/indirect. Four recurring shapes, with a
**difficulty gradient** (do the cheap ones first):

| # | Shape | Example | Fix mechanism | Cost |
|---|---|---|---|---|
| 1 | **Named attribute / descriptor** | django `self._iterable_class(self)` | framework resolver (`claimsReference` + `resolve()`) | **cheap** |
| 2 | **Field-backed observer** | `onUpdate(cb)` + `for(cb of cbs)cb()` | callback synthesizer (whole-graph pass) | medium |
| 3 | **String-keyed EventEmitter** | `on('e',fn)` / `emit('e')` | callback synthesizer (event-keyed) | medium |
| 4 | **Inline callback handler** | `on('e', function h(){})` / `() => {}` | extraction (named) + synthesizer link-through-body (anon) | named: cheap · anon: hard |

Key distinction driving the mechanism choice:
- **A named ref exists** to resolve (`_iterable_class` is an attribute name) → **resolver**.
- **No ref exists** (`cb()` is anonymous; needs registrar↔dispatcher correlation) → **synthesizer**.

---

## 3. Worked examples (the two mechanisms, end to end)

### 3a. Django ORM descriptor — the **resolver** pattern (Python)
- **Hole:** `QuerySet._fetch_all` calls `self._iterable_class(self)` (a runtime-chosen
  iterable, default `ModelIterable`), whose `__iter__` runs the SQL compiler. Static
  parsing can't resolve the attribute-as-callable → `_fetch_all`'s only callee was
  `_prefetch_related_objects`; `trace(_fetch_all, execute_sql)` returned no path.
- **Fix:** `djangoResolver` claims the unresolved `_iterable_class` ref through the
  name-exists pre-filter, then resolves it to `ModelIterable.__iter__`.
- **Files:** `src/resolution/types.ts` (`claimsReference?` on `FrameworkResolver`),
  `src/resolution/index.ts` (pre-filter in `resolveOne` consults `claimsReference`),
  `src/resolution/frameworks/python.ts` (`djangoResolver.resolve` + `claimsReference` +
  `resolveModelIterableIter`).
- **Result:** `trace(_fetch_all, execute_sql)` → `_fetch_all → __iter__ → execute_sql` (3 hops).

### 3b. Excalidraw observer + EventEmitter — the **synthesizer** (TS)
- **Hole:** `Scene.triggerUpdate` does `for (cb of this.callbacks) cb()`; `triggerRender`
  is registered via `scene.onUpdate(this.triggerRender)`. The `triggerUpdate →
  triggerRender` edge is dynamic → `trace` returned no path; the whole update flow broke.
- **Fix:** a whole-graph pass that detects registrar/dispatcher channels, correlates
  registration sites, and synthesizes `dispatcher → callback` edges. Plus extraction of
  **named** inline callbacks so handlers like express's `function onmount(){}` are nodes.
- **Files:** `src/resolution/callback-synthesizer.ts` (the pass — field observers +
  EventEmitter), `src/resolution/index.ts` (calls `synthesizeCallbackEdges()` at the end
  of `resolveAndPersistBatched`), `src/extraction/tree-sitter.ts` (`visitFunctionBody`
  extracts named nested functions).
- **Result:** `trace(mutateElement, triggerRender)` → 3 hops; express `use → onmount`.

---

## 4. The repeatable methodology (run this per language/framework)

### Step 1 — Pick the framework's canonical *flow* question
Every framework has a signature data/control flow. Pick the "how does X reach/become Y"
question and a real repo (add to `.claude/skills/agent-eval/corpus.json`). Examples:
- React state→DOM, Vue reactive→render, Svelte store→update
- Rails request→controller→view, Spring request→`@Controller`→service
- Express/Koa request→middleware→handler, FastAPI request→route→dependency
- Redux action→reducer→store, RxJS subscribe→operator→observer
- Any ORM: query builder → SQL execution (django pattern)

### Step 2 — Measure the hole (deterministic, no agent)
```bash
rm -rf <repo>/.codegraph && ( cd <repo> && codegraph init -i )
node scripts/agent-eval/probe-trace.mjs <repo> <from-symbol> <to-symbol>   # does the flow break? where?
node scripts/agent-eval/probe-node.mjs  <repo> <break-symbol>              # trail: is the next hop missing?
```
A "No direct call path … breaks at dynamic dispatch" + a sparse trail at the break
point **locates the hole** (this is exactly how `_iterable_class` and `triggerUpdate`
were found). Confirm it's dynamic by reading the break symbol's body.

### Step 3 — Classify → choose the mechanism (use the §2 table)
- `self.<attr>(...)` / descriptor / metaclass → **resolver** (§3a).
- `for(cb of store)cb()` / `store.forEach(cb=>cb())` → **field-observer synthesizer** (§3b).
- `on('e',fn)` + `emit('e')` → **EventEmitter synthesizer** (§3b).
- Inline handler not a node → **named:** extraction (already done generically in
  `tree-sitter.ts`); **anonymous:** synthesizer link-through-body (not yet built).

### Step 4 — Implement
- **Resolver:** add to `src/resolution/frameworks/<lang>.ts` — a `resolve()` branch +
  `claimsReference(name)` if the ref name isn't a declared symbol. Copy `djangoResolver`.
- **Synthesizer channel:** extend `src/resolution/callback-synthesizer.ts` — add the
  framework's registrar/dispatcher **name patterns** and **body patterns** (e.g. signals
  use `.connect()`/`.emit()`; Rx uses `.subscribe()`/`.next()`).
- Reindex (Step 2 command) and re-run `probe-trace` — the flow should now connect.

### Step 5 — Validate (the same way every time)
1. **Deterministic:** `probe-trace(from,to)` finds the path; `probe-node` shows the
   bridged hop. The previously-broken hop is closed.
2. **Precision:** count + spot-check synthesized/resolved edges — no explosion, correct targets:
   ```bash
   sqlite3 <repo>/.codegraph/codegraph.db \
     "select s.name||' → '||t.name||'  '||coalesce(e.metadata,'') from edges e \
      join nodes s on e.source=s.id join nodes t on e.target=t.id where e.provenance='heuristic';"
   ```
   (Resolver edges aren't `heuristic`; verify via the trace + callees instead.)
3. **Regression:** node count stable (`select count(*) from nodes;` before/after — a big
   jump means an extraction change over-fired); existing traces on a control repo intact.
4. **End-to-end agent eval:** run the flow question with codegraph and measure
   **reads / answer-completeness / cost** vs a pre-fix baseline:
   ```bash
   # headless (exact cost + clean tool sequence)
   bash scripts/agent-eval/run-agent.sh <repo> with "<flow question>"
   # or the full A/B + interactive Explore-subagent path:
   scripts/agent-eval/audit.sh local <name> <url> "<flow question>" all
   ```
   Then parse: `Read` count, codegraph-tool count, cost, and whether the answer now
   contains the glue symbols (the ones that previously required a read).

### Success criteria (per language/framework)
- `trace` finds the canonical flow end-to-end (no dynamic-dispatch break).
- Agent can answer the flow question with **Read 0** (achievable in ≥ some runs) and the
  glue symbols appear in the answer.
- **No node explosion** and no regression on a control repo.
- Synthesized edges are precise on a spot-check (no generic-name over-linking).

---

## 5. Validation toolkit (reference)

| Tool | Purpose |
|---|---|
| `scripts/agent-eval/probe-trace.mjs <repo> <from> <to>` | call-path between two symbols (the hole detector) |
| `scripts/agent-eval/probe-node.mjs <repo> <sym> [code]` | symbol + trail (callers/callees); `code` adds the body |
| `scripts/agent-eval/probe-context.mjs <repo> "<task>"` | context output incl. call-paths |
| `scripts/agent-eval/probe-explore.mjs <repo> "<query>"` | explore output |
| `scripts/agent-eval/{audit,run-agent,itrun}.sh` | agent A/B (headless + interactive); also the `/agent-eval` skill |
| `sqlite3 <repo>/.codegraph/codegraph.db` | direct edge/node inspection (provenance, metadata, counts) |

Probe scripts use the built `dist/` — run `npm run build` first. Reindex after any
extraction or resolution change (`rm -rf <repo>/.codegraph && codegraph init -i`) — the
synthesizer/resolvers run at index time. Test fixtures: keep a tiny per-pattern fixture
(see `/tmp/cb-fixture/bus.js`; **move into `__tests__/`** when shipping).

---

## 6. Coverage matrix (fill in as you go)

Status legend: ✅ done+validated · 🔬 hole identified · ⬜ not started.
`Mechanism`: R = resolver, S = synthesizer channel, X = extraction.

| Language | Framework(s) | Canonical flow to test | Mechanism | Status |
|---|---|---|---|---|
| TypeScript/JS | React / observer / EventEmitter / React Router | state→render; dispatch→callback; route→component | S + X | ✅ rendering+dispatch (excalidraw); **React Router JSX routing** `<Route path component={C}/>` (v5) + `element={<C/>}` (v6) → component (react-realworld **0→10, 10/10**). 🔬 object data-router `createBrowserRouter([{path,element}])` (modern v6) + a pre-existing Next.js false-positive (config files in a `pages/` app dir treated as routes) |
| TypeScript/JS | Vue / Nuxt | template events (@click→handler); component composition; reactive→render | S + X | ✅ events + composition (vitepress S / vben M / element-plus L); 🔬 reactive→render (vue-core Proxy runtime — frontier, deferred) |
| TypeScript/JS | Svelte / SvelteKit | template calls/composition; SvelteKit action→api; store→DOM | X | ✅ already strong (realworld S / skeleton M / shadcn L): template `{fn()}` calls, `<Pascal/>` composition, `import * as api` namespace, `load`→api all work out of the box. + exported-const object-of-functions extraction (SvelteKit `actions`). 🔬 `$lib`-namespace-from-action + store/reactive frontier |
| TypeScript/JS | Express / Koa | request → route → handler → service | R + X | ✅ named handlers + middleware + controller/service (resolver) + **inline arrow handlers → service body calls** (realworld S 19 / parse M / ghost L 65 edges). 🔬 custom routers (payload had 0 routes — not `app.get`-style) |
| TypeScript/JS | NestJS | request → @Controller → DI service → repo | R | ✅ already well-covered (realworld S / immich M-L / amplication L): @decorator routes (HTTP/GraphQL/microservice/WS) via resolver + DI `this.svc.method()` controller→service resolves correctly at scale (name + co-location). No dynamic-dispatch hole. 🔬 committed `dist/` build output gets indexed (realworld) — general build-dir-ignore follow-up |
| TypeScript/JS | RxJS / signals | subscribe → operator → observer | S | ⬜ |
| Python | Django ORM | QuerySet → SQL compiler | R | ✅ |
| Python | Django / DRF (views) | url → view → model | R + X | ✅ url→view (`path`/`url`/`as_view`) + **DRF `router.register`→ViewSet** (realworld S / wagtail M / saleor L); ORM QuerySet→SQL (prior work). 🔬 signals (`post_save`→receiver), DRF viewset CRUD actions (inherited), saleor GraphQL resolvers |
| Python | Flask / FastAPI | request → route → handler → dependency | R + X | ✅ **Flask: handler resolved across intervening decorators (`@login_required`) + stacked `@x.route` lines** (microblog S 6→27, redash L decorator routes 6/6); **FastAPI: empty-path router-root routes `@router.get("")` incl. multi-line** (realworld S 12→20 / Netflix dispatch L **290/290 100%**) + **bare-name builtin guard** — a handler named after a Python builtin method (`index`/`get`/`update`/`count`…) was filtered as a builtin and lost its route→handler edge. 🔬 Flask-RESTful class-based `add_resource(Resource, '/x')` (redash — separate mechanism, not the README decorator/blueprint shape); FastAPI `Depends()` dependency edges (resolver exists, light validation) |
| Go | Gin / chi / net-http | request → route → handler → service | X | ✅ **routes on ANY group var** (`v1.GET`, `PublicGroup.GET`) not just `r/router` (gin-vue-admin S→M 4→259 / realworld S / gitness L) — was missing all group-routed apps; named handlers resolve precisely. 🔬 inline `func(c){}` handlers (anonymous, body lost), gitness chi custom (26/321) |
| Rust | Axum / actix / Rocket | request → route → handler | R + X | ✅ **Axum chained methods + namespaced handlers** — `.route("/x", get(h1).post(h2))` emitted only the first method+handler, and `get(mod::handler)` captured the module not the fn (realworld-axum S **12→19, 19/19**); balanced-paren scan + per-method nodes + last-`::`-segment handler. **Rocket attribute macros 550/556 (99%)** (Rocket repo L) — already strong. crates.io named axum routes resolve (6/8; rest are closures/var handlers; its API is mostly the utoipa `routes!` macro = frontier). Cargo-workspace module resolution (prior work). **actix builder API** `web::resource("/x").route(web::get().to(h))` / `.to(h)` / App `.route("/x", web::get().to(h))` (actix-examples **51→128 routes, 35→112 resolved**) — was the dominant actix style and fully missed (the handler is in `.to(h)`, not `get(h)`). 🔬 actix `web::scope("/api")` prefix (not prepended to nested resource paths) + anonymous `.to` closure handlers |
| Java | Spring | request → @RestController → @Autowired service → repo | R + X | ✅ **bare `@GetMapping`/`@PostMapping` + class `@RequestMapping` prefix join → route→method** (realworld S / mall M / halo L) — was missing all path-less method mappings; DI controller→service resolves (name + dir). 🔬 Spring Data JPA derived queries (`findByEmail`) — metaprogramming frontier |
| Kotlin | (coroutines / DI) | flow/callback dispatch | ? | ⬜ |
| Swift | Vapor | request → route → controller | R + X | ✅ **was 0 routes on every real app** — the extractor required an `app/router/routes` receiver + a `"path"` literal, but real Vapor routes on grouped builders (`let todos = routes.grouped("todos"); todos.get(use: index)`) with NO path arg. Rewrote: any receiver, optional/non-string path segments, `.grouped`/`.group{}` prefix tracking, `use:` discriminator. vapor-template S **0→3 (3/3**, nested `/todos/:todoID`), SteamPress M **0→27 (27/27)**, SwiftPackageIndex-Server L **0→14 (14/14** handler resolution). 🔬 typed-route enums (SPI `SiteURL.x.pathComponents` — path label only, handler still resolves) + closure handlers `app.get("x"){ }` (anonymous) |
| C# | ASP.NET Core | request → [Http*] action → DI service → EF | X | ✅ **feature-folder detection** (realworld 0→19 — was undetected) + **bare `[HttpGet]` + class `[Route]` prefix** (eShopOnWeb 9→33 / jellyfin L) — co-located so no claimsReference needed. 🔬 EF Core LINQ/DbSet (metaprogramming frontier) |
| Ruby | Rails / Sinatra | request → routes.rb → Controller#action → model | R | ✅ **RESTful `resources`/`resource` routing → controller#action** (realworld S 16 / spree M / forem L), pluralization + only/except + claimsReference; explicit routes fixed to precise `controller#action` too. 🔬 ActiveRecord dynamic finders (`Article.find_by_slug`) — metaprogramming frontier |
| PHP | Laravel | request → route → controller → Eloquent | R | ✅ **precise `Route::get([Ctrl::class,'m'])` / `'Ctrl@m'` → Ctrl@method** (realworld S / firefly M / bookstack L) — was resolving the bare method name to the WRONG controller (every `index`→ArticleController); Route::resource→controller. 🔬 Eloquent dynamic finders/relationships (metaprogramming frontier) |
| PHP | Drupal | request → *.routing.yml → _controller/_form | R | ✅ **`claimsReference` for FQCN handlers** (`\Drupal\…\Class::method` passed the pre-filter only because the `::method` name was known; bare `_form` FQCNs `\…\FormClass` and single-colon `Class:method` controller-services were dropped before resolve()) + **single-colon controller match** + **detect via composer `type:drupal-*` / `name:drupal/*` + `*.info.yml` fallback** (a contrib module with empty `require` was undetected → 0 routes). admin_toolbar S **0→14 (14/14)** / webform M 208 (**144**) / core L 836 (536→**731, 87%**). Remainder is the **entity-annotation handler frontier** (`_entity_form: type.op` resolves via the entity's PHP `#[ContentEntityType]` handlers, not a direct class). 🔬 **OOP `#[Hook]` attributes** — Drupal 11 moved ~all procedural hooks to attribute methods (core: 418 `#[Hook]` files vs 3 procedural), so the resolver's docblock/`module_hook` detection is obsolete for modern core (0 hook edges) |
| C/C++ | (callback structs / vtables) | function-pointer dispatch | ? | ⬜ |
| Dart | Flutter | setState → build; build → child widgets | S + X | ✅ **setState→build synthesizer** (Dart analog of react-render: a State method whose body calls `setState(` → `build`) gated to `.dart` + **foundational Dart method-range fix** — Dart models a method body as a *sibling* of the signature, so method nodes were signature-only (`end==start`); now `endLine` spans the body (required for ALL body analysis: callees, context slices, the synthesizer's body scan). counter `initState→build`, books `build→BookDetail/BookForm`; widget composition already static (compass_app `build→ErrorIndicator/HomeButton`). Controls unchanged (excalidraw 9,290 / django 302 — the range fix only extends sibling-body grammars). 🔬 MVVM Command/ChangeNotifier dispatch (compass_app — no setState) + `Navigator.push(MaterialPageRoute(builder:))` nav routes |
| Lua / Luau | (Neovim / Roblox) | event/callback dispatch | S | ⬜ |
| Scala | (Akka / Play) | actor message → handler | ? | ⬜ |

(Verify the exact supported set against `src/extraction/languages/` and
`src/resolution/frameworks/` before starting — this table is a starting point.)

---

## 7. Known limits & gotchas (from the excalidraw/django work)

- **Coverage enables, doesn't force, the no-read path.** Agents still read to *confirm
  source* sometimes; cost stays ~flat (codegraph calls trade for reads). The reliable
  win is **completeness** + making Read-0 *possible*. Don't expect a guaranteed cost drop.
- **Vue (validated 2026-05-23, vitepress S / vben M / element-plus L).** SFC `<template>`
  is unparsed by the extractor, so template usage needs synthesis (`vueTemplateEdges`):
  `@click="fn"` → handler, kebab `<el-button>` → `ElButton`. PascalCase `<Child/>` is
  already covered by the JSX channel (the SFC component node spans the template). Result:
  agent reads drop in every size (vben login 1–3 vs 4–11), **strongest where handlers are
  local functions** (vben `handleLogin`/`handleSubmit`).
  **Composable-destructure handlers RESOLVED:** `@click="closeSidebar"` where
  `const { close: closeSidebar } = useSidebarControl()` now follows alias → composable →
  the returned `close` fn (when it's defined in the composable's file). vitepress sidebar
  flow dropped **6 → 0 reads** (best case). Precise-only — no fallback to the composable
  itself (the static `useX()` call edge already covers that), so it adds nothing where the
  returned fn can't be located (e.g. re-exported / external composable). Remaining limits:
  **prefix-convention kebab** — element-plus `el-button` → `button.vue` (component named
  `button`, not `ElButton`), so kebab stays unresolved there; and **reactive→render**
  (vue-core Proxy runtime) — the deep framework-internal frontier, deferred.
- **Svelte / SvelteKit (validated 2026-05-23, realworld S / skeleton M / shadcn L) — already well-covered.**
  Unlike Vue, the `.svelte` extractor already parses the template: `extractTemplateCalls` (`{fn()}`),
  `extractTemplateComponents` (`<Pascal/>` composition — skeleton 956 / shadcn 1610 reference edges),
  plus `import * as api` namespace + `load`→api resolution all work. Agent A/B (realworld login): with
  codegraph **1 read** vs without **4** — codegraph already wins out of the box. The one extraction gap
  was **object-of-functions** (`export const actions = { default: async () => {} }`; the walker
  deliberately skips object-literal functions to avoid inline-object noise). Fixed for EXPORTED consts
  (general — Redux/Express handler maps too); `extractFunction` `nameOverride` keeps inline-object arrows
  skipped. **Residual:** a `$lib`-alias namespace call (`api.post`) from an extracted action node doesn't
  resolve even though the same alias resolves for `load` — a deeper resolver interaction, deferred
  (local/relative calls from actions connect). **Lesson: measure before assuming a hole** — modern Svelte
  barely uses `on:click={fn}` (form actions / callback props instead), so the assumed event-handler hole
  wasn't the real one; Svelte needed far less than Vue.
- **Express / Koa (validated 2026-05-23, realworld S / parse M / ghost L) — high-value inline-handler fix.**
  The resolver already handled named handlers, middleware, and `XController.method`/`XService.method`.
  The real hole was **inline arrow route handlers** (`router.post('/x', async (req,res) => {...})` — the
  dominant modern pattern): the handler regex `[^)]+` broke on the arrow's `)`, so the route connected to
  NOTHING and the anonymous handler's body (the request→service flow) was lost. The entire inline-handler
  API was unreachable (realworld `POST /users/login` → 0 edges). Fixed (`frameworks/express.ts`): span the
  call with a string-aware balanced scan; for inline arrows, extract the body's calls (RESERVED-filtered to
  drop res/req/builtins) and attribute them to the route node → realworld **19** / ghost **65** precise
  route→service edges (POST /users/login→login, POST /articles→createArticle, …), no node explosion,
  framework-scoped (zero blast radius off Express). **Deterministic win is clear; the agent A/B is muddied
  by repo characteristics** — realworld (39 files) is below the size where codegraph beats reading, and
  Ghost's layered custom-API architecture makes both arms thrash. Residual: **custom routers** — payload's
  6.4k-file codebase had 0 routes (its router abstraction isn't `app.get`-style, so undetected). Lesson
  inverse of Svelte: Express's dominant pattern WAS the uncovered one, so it needed real work like Vue.
- **NestJS (validated 2026-05-23, realworld S / immich M-L / amplication L) — already well-covered.** The
  `nestjs` resolver handles @decorator routes (HTTP/GraphQL/microservice/WS). DI controller→service
  (`this.svc.method()`) resolves correctly **even at scale** — every immich controller→service edge hit the
  right same-module service (`addUsersToAlbum→addUsers`, `getMyApiKey→getMine`, `copyAsset→copy`) via
  name + co-location, no type_of edge needed. Agent A/B (immich album flow): codegraph **eliminated Grep
  (0 vs 3)** tracing route→controller→service. No dynamic-dispatch hole. One GENERAL hygiene gap surfaced
  (not NestJS-specific): the realworld example **commits its `dist/`** build output, which codegraph indexes
  (246 dup nodes) because the file walk only respects `.gitignore` with no default build-dir ignore. Real
  apps (immich/amplication) gitignore `dist/` (0 dup nodes), so it's narrow — a default ignore for
  `dist/build/out/.next/coverage` is a clean follow-up, deferred (core-indexer change, the user's call).
- **Rails (validated 2026-05-23, realworld S / spree M / forem L) — high-value RESTful-routing fix.** The
  `rails` resolver only saw explicit `get '/x' => 'c#a'` routes, so resource-routed apps (the dominant
  pattern) had ZERO route nodes (realworld + spree). Fixed (`frameworks/ruby.ts`): expand `resources :x` /
  `resource :x` into their RESTful actions (only/except filters + pluralization for the singular `resource`),
  reference a precise `controller#action`, and resolve that to the action method in `<ctrl>_controller.rb`
  (explicit routes fixed too — they referenced a bare ambiguous `action`). realworld **0→16**, forem
  **0→635** precise route→action edges. Agent A/B (forem comment-creation, large): codegraph **1–4 reads /
  0 grep / 47–53s** vs without **4–5 reads / 2–3 grep / 66–85s** — fewer reads, no grep, faster. **The
  `claimsReference` pre-filter was the gotcha:** `articles#index` names no declared symbol, so `resolveOne`
  dropped it before `resolve()` ran — needed the same claim hook as the django ORM work. Residuals: **Rails
  Engine routing** (spree still 0 — it mounts an engine, not `config/routes.rb` resources); ActiveRecord
  dynamic finders (`Article.find_by_slug` — metaprogramming frontier).
- **Spring (validated 2026-05-23, realworld S / mall M / halo L) — bare-mapping + class-prefix routing fix.**
  The resolver required a string path in the mapping regex, so BARE method mappings (`@PostMapping` with the
  path on the class `@RequestMapping`) — the dominant multi-method-controller pattern — were missed (halo
  had 28 routes for 2444 files; realworld's 2-action favorite controller linked only one). Fix
  (`frameworks/java.ts`): treat class `@RequestMapping` as a PREFIX (joined, not a bogus route); match
  verb-specific mappings BARE-or-with-path; also handle method-level `@RequestMapping(method=...)` (older
  style). realworld 13→19, mall →246 precise route→method (class prefix joined); DI controller→service
  resolves (`article→findBySlug`). Agent A/B (mall cart flow): with codegraph 0 reads/0 grep vs without 2/2.
  **A first cut regressed mall 292→1** by dropping `@RequestMapping`-on-method — *caught by the cross-repo
  route-count check*; the playbook's regression guard earns its keep. Residuals: halo's custom patterns
  (9/29 resolve); Spring Data JPA derived queries (metaprogramming frontier).
- **Django / DRF (validated 2026-05-23, realworld S / wagtail M / saleor L) — mostly covered + a DRF-router
  fix.** The ORM (`_iterable_class`→ModelIterable, the original investigation) and URL routing
  (`path`/`url`/`as_view`→view) were already done. The one hole: **DRF `router.register(r'articles',
  ArticleViewSet)`** (the core CRUD endpoints) wasn't extracted — only `path()`/`url()` were. Fix
  (`frameworks/python.ts`): match `router.register` (the STRING first arg separates it from
  `admin.register(Model, Admin)`, whose first arg is a model class) → route→ViewSet class. Narrow in this
  corpus (realworld has 1 router; wagtail uses `path()`, saleor is GraphQL) but real for DRF-router APIs.
  Agent A/B (wagtail Page flow, medium): codegraph **4–7 reads / 1–4 grep / 58–81s** vs without **7–9 reads
  / 6 grep / 82–86s** — fewer reads, fewer greps, faster. No regression (wagtail/saleor route counts
  unchanged — purely additive). Residuals: signals (`post_save`→receiver), DRF viewset CRUD actions
  (inherited from the base class, not in the user's ViewSet), saleor's GraphQL resolvers.
- **Laravel (validated 2026-05-23, realworld S / firefly M / bookstack L) — route precision fix.** The
  resolver discarded the controller from the handler: `Route::get([UserController::class,'index'])` /
  `'UserController@index'` emitted a BARE `index` ref, which name-matching mis-resolved to the WRONG
  controller (every `index`/`show` → whichever it found first; realworld GET user → ArticleController.index,
  should be UserController). Fix (`frameworks/laravel.ts`): emit precise `Controller@method` (array + string
  syntax, namespace-stripped) + `claimsReference` it past the pre-filter → existing Pattern-4
  `resolveControllerMethod`. realworld all routes correct; bookstack 267/332 precise (GET pages →
  PageApiController.list). Agent A/B (bookstack page-view, large): codegraph **2–3 reads / 1–2 grep /
  51–60s** vs without **4–6 / 3–5 / 60–74s**. No node explosion. Residuals: firefly resolves only 3/568
  (its fluent `->uses()` / `['uses'=>...]` handler format isn't parsed); Eloquent dynamic finders
  (metaprogramming frontier).
- **Gin / chi (validated 2026-05-23, realworld S / gin-vue-admin M / gitness L) — group-var routing fix.**
  The route regex matched only `(router|r|mux|app|e).METHOD(...)`, but real apps route on GROUP vars
  (`v1.GET`, `PublicGroup.GET`, `userRouter.POST`), so group-routed apps connected almost nothing
  (gin-vue-admin: **4 routes for 625 files**). Fix (`frameworks/go.ts`): broaden the receiver to ANY
  identifier — the verb + string-path + handler-arg gates keep it route-specific (`http.Get(url)` has no
  handler arg → excluded). gin-vue-admin **4→259** routes (257 resolve precisely: `POST createInfo →
  CreateInfo`); realworld stable (no regression); no garbage. **Agent A/B (create-user flow): codegraph
  0 reads / 0 grep / 26–30s vs without 3 / 3 / 52–53s — cleanest backend win yet (0/0, 2× faster).**
  Residuals: inline `func(c *gin.Context){}` handlers (anonymous, body lost — like Express before its fix);
  gitness's chi custom handlers (26/321).
- **ASP.NET Core (validated 2026-05-23, realworld S / eShopOnWeb M / jellyfin L) — detection + bare-attribute
  fix.** Two holes: (1) `detect()` only fired on a `/Controllers/` dir or root `Program.cs`/`.csproj` (which
  often isn't in the indexed source set), so feature-folder apps (realworld: `Features/*/FooController.cs`,
  subdir `Program.cs`) were NEVER detected → 0 routes despite a full controller set. Broaden: scan
  Controller/Program/Startup `.cs` for ASP.NET signatures. (2) the attribute regex required a string path →
  bare `[HttpGet]` (route on the class `[Route("[controller]")]`) missed (eShopOnWeb was 24 bare / 2
  string). Match bare-or-path + join the class `[Route]` prefix (like Spring). **No `claimsReference`
  needed** — ASP.NET attribute routes are co-located IN the controller with the action, so the bare method
  ref resolves same-file (unlike Rails/Laravel, whose routes live in a separate file). realworld 0→19,
  eShopOnWeb 9→33, jellyfin 362→399, all precise (`GET /articles → Get`, class prefix joined), no explosion.
  Agent A/B (eShop catalog listing): codegraph **1–2 reads / 0 grep / 63–75s** vs without **6–7 / 1–6 /
  77–79s**. Residual: EF Core LINQ/DbSet (metaprogramming frontier).
- **Flask / FastAPI (validated 2026-05-23, fastapi-realworld S / flask-microblog S / Netflix dispatch L /
  redash L) — decorator-extraction + builtin-name fixes.** Routes were extracted but the request→route→handler
  flow broke at two regex assumptions and one resolver filter. (1) **Flask required `def` immediately after
  `@x.route(...)`**, so any intervening decorator (`@login_required`, `@cache.cached`) or **stacked `@x.route`
  lines** (one view bound to several URLs) dropped the route — microblog extracted **6 of 27** real routes.
  Switched Flask to FastAPI's `findHandler` scan (match the decorator, then find the next `def`), skipping
  intervening decorators: **6→27**, all resolved. (2) **FastAPI's path regex `[^'"]+` rejected the empty path**
  `@router.get("")` (router/prefix-root routes, frequently multi-line) → realworld lost 8 endpoints (list/create
  article, comments, login/register). `[^'"]+`→`[^'"]*` + empty-path name guard: realworld **12→20**, Netflix
  dispatch **290/290 (100%)**. (3) **Bare-name builtin guard** (`src/resolution/index.ts`): a handler named
  after a Python builtin *method* (`index`, `get`, `update`, `count`…) was filtered by `isBuiltInOrExternal`
  and lost its route→handler edge — microblog's `index` view (its `/` + `/index` stacked routes) resolved to
  nothing. The dotted-method branch already had a `knownNames` guard; mirrored it onto the bare branch (a name
  a declared symbol owns is not a builtin call). +2 legit edges on realworld, **0 change on the django control**
  (302/373 identical — precision held). Flows trace end-to-end (`login → get_user_by_email` 2 hops;
  `create_user → from_dict`). Agent A/B (realworld login-auth flow, n=2/arm): codegraph **0–1 read / 0 grep /
  3–4 codegraph / 30–39s** (context→[search]→trace→node) vs without **3 read / 2 grep / 33–36s** — eliminates
  grep, cuts reads to 0–1 (small repo, so wall-clock ties; the tool-count drop is the win). Residuals: **Flask-RESTful** class-based
  `api.add_resource(Resource,'/x')` (redash's actual API shape — a separate class-method-as-verb mechanism, NOT
  the README's documented decorator/blueprint Flask) and a pre-existing **JS file-route false-positive** in
  redash's React frontend (32 bogus `.js` "routes" from a JS resolver — unrelated to Python). **Lesson: the
  builtin-name filter is a silent precision tax across Python** — any view/function named `get`/`index`/`update`
  loses edges; the fix is general (helps Django/DRF handlers too), not Flask-specific.
- **Drupal (validated 2026-05-23, admin_toolbar S / webform M / drupal-core L) — pre-filter + detection fixes.**
  The `*.routing.yml` extractor and the `_controller`/`_form` resolver already existed but two gaps kept most
  routes unlinked. (1) **The `claimsReference` pre-filter gotcha (again):** Drupal handler refs are FQCNs
  (`\Drupal\…\Class::method`), bare form classes (`\…\SettingsForm`), or single-colon controller-services
  (`\…\Controller:method`). Only the `::method` shape survived `resolveOne`'s pre-filter (its `member` is a
  known method name); the bare-FQCN forms and single-colon controllers named no declared symbol and were
  dropped before `resolve()` ran. Added `claimsReference` (FQCN / `Class:method` / `hook_*`) + a single-colon
  branch in the controller regex → core **536→731 of 836 routes (87%)**; all three previously-broken shapes now
  resolve (`/admin/content/comment`→CommentAdminOverview form, `/big_pipe/no-js`→setNoJsCookie controller).
  (2) **Detection missed standalone contrib modules:** `detect()` only checked composer `require` for a
  `drupal/*` dep, but a contrib module often has an EMPTY `require` and is identified only by
  `"name":"drupal/<m>"` + `"type":"drupal-module"` (admin_toolbar → 0 routes). Broadened to composer name/type
  + a `*.info.yml` fallback → admin_toolbar **0→14 (14/14)**. Canonical flow traverses (`getAnnouncements` ←
  `/admin/announcements_feed`); node count unchanged (resolution-only). Agent A/B (dblog route→controller,
  n=2/arm): codegraph **0 read / 1 grep / 20–22s** vs without **1 read / 2 grep + glob / 28–32s** — fewer
  tools and faster on the ~10k-file core. **Residuals (frontier):**
  entity-annotation handlers (`_entity_form: comment.default` → handler classes declared in the entity's
  `#[ContentEntityType]` annotation, not a direct ref — ~78 of core's ~105 remaining unresolved) and **OOP
  `#[Hook]` attributes** — Drupal 11 converted nearly all procedural hooks to `#[Hook('event')]` methods (core:
  418 attribute files vs 3 procedural `*.module` hooks), so the resolver's procedural-hook detection (docblock
  `@Implements` / `module_hook` naming) finds essentially nothing in modern core (0 hook edges). Both are real
  follow-ups, not regressions.
- **Rust / Axum + Rocket + actix (validated 2026-05-23, realworld-axum S / actix-examples + Rocket M / crates.io L) — Axum chained-method + namespaced-handler fix.**
  The attribute-macro path (`#[get("/x")] fn h`, actix/Rocket) and single Axum `.route("/x", get(h))` already
  worked, but the Axum extractor used a flat regex that captured only the FIRST `method(handler)` of a route
  and only a bare `\w+` handler. Two dominant Axum idioms broke it: (1) **method chains**
  `.route("/user", get(get_current_user).put(update_user))` — the `.put` arm produced NO route node, so half
  the API was missing (realworld-axum had only the GET of each chain); (2) **namespaced handlers**
  `get(listing::feed_articles)` — `\w+` captured `listing` (the module), so the route resolved to nothing.
  Rewrote with a balanced-paren scan of each `.route(...)` call, a per-method node, and last-`::`-segment
  handler names → realworld-axum **12→19 routes, 19/19 resolved** (every chained PUT/DELETE/POST now present;
  `feed_articles` resolves). **Rocket needed nothing** (550/556, 99% — attribute macros). crates.io confirms
  namespaced axum handlers resolve (router.rs 6/6) but defines most of its API via the `utoipa_axum` `routes!`
  macro (frontier) and has a SvelteKit frontend (42 of its 50 "routes" are `+page.svelte`, correctly
  attributed to SvelteKit). Agent A/B (update-user flow,
  n=2/arm): codegraph **0–2 read / 0 grep / 32–40s** vs without **3 read / 0–1 grep + glob / 33–41s** — modest
  (realworld-axum is in the small-repo tie zone) but consistent, with one fully-clean 0-read/0-grep run. Node
  count stable; the Axum fix is Axum-scoped (the attribute/actix/Rocket path is untouched).
- **Actix runtime routing (validated 2026-05-23, actix-examples) — the builder API was the dominant style and fully missed.**
  Actix's attribute macros (`#[get("/x")] fn h`) were covered, but real actix apps route via the builder API:
  `web::resource("/path").route(web::get().to(handler))`, `web::resource("/").to(handler)` (all methods), and
  App-level `.route("/path", web::get().to(handler))`. The handler lives in `.to(handler)`, not `get(handler)`,
  so the Axum `.route` scan extracted nothing for them — actix-examples had **80 `web::resource` calls** all
  unlinked. Added an actix block: scan each `web::resource("/path")` (bounding its method chain at the next
  resource to avoid bleed) for `web::METHOD().to(h)` pairs, fall back to a direct `.to(h)` (method `ANY`), plus
  the App-level `.route("/x", web::METHOD().to(h))` form. actix-examples **51→128 routes, 35→112 resolved
  (87.5%)** (`GET /user/{name}`→with_param, `POST /user`→add_user). No regression on Axum (realworld-axum still
  19/19) — the actix patterns (`web::resource`/`web::method().to()`) don't appear in Axum code. **Residuals
  (frontier):** `web::scope("/api")` prefixes aren't prepended to nested resource paths, and anonymous `.to(|req|
  …)` closure handlers have no named target (the ~16 still-unresolved).
- **Swift / Vapor (validated 2026-05-23, vapor-template S / SteamPress M / SwiftPackageIndex-Server L) — the resolver was effectively dead on real apps.**
  The Vapor extractor only matched `(app|router|routes).METHOD("path", use: handler)`, but modern Vapor routes
  on a grouped builder inside `RouteCollection.boot(routes:)`: `let todos = routes.grouped("todos");
  todos.get(use: index)` — any var receiver, NO path arg (the path is the group prefix). Every real app tested
  extracted **0 routes** (template, penny-bot, Feather, SteamPress, SPI). Rewrote the extractor: (1) any
  receiver `\w+` (not just app/router/routes); (2) optional path segments that may be non-string
  (`User.parameter`, `:id`, a path constant) — the `use:` keyword is the discriminator separating a route from
  `Environment.get("X")` / `req.parameters.get("X")`; (3) a group-prefix map from `let X = Y.grouped("a")` and
  `Y.group("a") { X in }` so a route on a grouped/nested var gets the full path (`todo.delete(use: delete)` →
  `DELETE /todos/:todoID`). Result: vapor-template **0→3 (3/3**, nested path exact), SteamPress **0→27
  (27/27**, incl. `BlogPost.parameter` routes), SPI **0→14 (14/14** handler resolution). Canonical flow
  traverses (`createPostHandler` ← `GET /createPost`, → `createPostView`). **Residuals (frontier):**
  typed-route enums (SPI registers via `app.get(SiteURL.x.pathComponents, use:)` — handler resolves but the
  path label is `/`, no string literal) and closure handlers (`app.get("hello") { req in }` — anonymous, no
  named target). penny-bot (Discord bot) and Feather (custom module router) have no standard Vapor routing at
  all — the Vapor ecosystem's routing styles vary widely. Agent A/B (create-post flow, n=2/arm): codegraph
  **0 read / 0 grep / 4 codegraph / 26–30s** (both runs fully clean) vs without **1–4 read / 0–2 grep +
  glob/bash, one run spawned a sub-agent / 34–48s**. Node count stable; fix is Vapor-scoped (SwiftUI/UIKit
  untouched).
- **React Router routing (validated 2026-05-23, react-realworld S) — the routing half of the React row.**
  React rendering (state→render, jsx-child) was already covered; route→component was NOT — `react.ts` extracted
  components/hooks and Next.js file routes but returned `references: []`, so `<Route>` declarations produced
  nothing. Added `<Route>` JSX extraction: scan a window after each `<Route\b` (so the nested `>` in
  `element={<Comp/>}` doesn't truncate it), pull `path="…"` + `component={C}` (v5) or `element={<C/>}` (v6) in
  any attribute order, emit a route node + component reference (resolves via the existing PascalCase
  `resolveComponent`). react-realworld **0→10, 10/10** (`/login`→Login, `/editor/:slug`→Editor,
  `/@:username`→Profile); `<Routes>` container excluded via the `\b` boundary. No regression on excalidraw
  (9,290 nodes, 46 react-render synth edges intact, 0 false routes). 🔬 the object **data-router** API
  `createBrowserRouter([{ path, element }])` (modern v6, used by bulletproof-react) is object-based not JSX — a
  separate frontier; plus a pre-existing Next.js false-positive (`*.config.mjs` in a `pages/` app dir treated
  as a route).
- **Dart / Flutter (validated 2026-05-23, flutter/samples: counter S / books S / compass_app M) — synthesizer + a foundational extractor fix.**
  Flutter's reactive hop is `setState(() {…})` re-running `build(context)` — framework-internal, no static edge,
  so "tap → handler → setState → rebuilt UI" dead-ends at setState (the Dart analog of React's setState→render).
  Added a `flutter-build` synthesizer channel (Phase 4b): for each Dart class with a `build` method, link every
  sibling method whose body calls `setState(` → `build` (gated to `.dart`). **But it was blocked by a
  foundational gap:** Dart models a method body as a *sibling* of the `method_signature` node, so every Dart
  method node had `endLine == startLine` (signature only) — `sliceLines(start,end)` saw only `void f() {`, never
  the body. Fixed in the shared `createNode`: when a function/method's resolved body sits beyond the node,
  extend `endLine` to it (guarded — child-body grammars are a no-op; controls excalidraw 9,290 / django 302
  unchanged). This fix is foundational, not Flutter-specific — every Dart callee/context/body scan was
  previously truncated. Result: counter `initState→build`, books `initState→build` + `build→BookDetail/BookForm`.
  **Widget composition needs no synthesis** — unlike JSX, Dart widgets are explicit constructor calls
  (`BookDetail(...)`), already static (compass_app `build→ErrorIndicator/HomeButton/_Card`). **Residuals
  (frontier):** MVVM state management (compass_app uses Command/ChangeNotifier + ListenableBuilder, 0 setState —
  a different dispatch shape) and `Navigator.push(MaterialPageRoute(builder: (_) => DetailPage()))` navigation
  (route-as-widget, uncovered).
- **Difficulty gradient is real:** named-ref dispatch (resolver) is cheap; anonymous
  callback dispatch (synthesizer) is medium; **anonymous-arrow handlers are the hard
  remaining gap** (no identity → need synthesizer link-through-body, not yet built).
- **Extraction changes are high blast radius.** The Phase-3 named-inline-callback
  extraction is in the *shared* `tree-sitter.ts` walker — re-check **node counts across
  several languages** after any extraction change (it held at +3 on excalidraw because
  anonymous arrows are skipped).
- **Synthesizer precision guards:** registrar-name uniqueness, named-only handlers, and
  an event **fan-out cap** (skip generic events like `error`/`change`). Receiver-type
  matching (via `type_of` edges) is the planned precision upgrade — deferred.
- **As-built shortcuts** (callback synthesizer): pairs registrar/dispatcher by *file*+field
  (class proxy), regex arg-recovery (named refs only), `provenance:'heuristic'` +
  `metadata.synthesizedBy` (the enum has no `'callback-synthesis'`). See the design doc.
- **Synthesizer runs only in `resolveAndPersistBatched`** (full index) — wire into
  `resolveAndPersist` for incremental sync before shipping.
- **Symbol ambiguity in `trace`:** common names (`render`, `execute_sql`) match many
  nodes; trace picks among them and may start from the wrong one. Trace from the specific
  method, not a class name.

---

## 8. Definition of done (the whole mission)

For each language × framework: the canonical flow `trace`s end-to-end, an agent can
answer the flow question with Read 0 in at least some runs with the glue present, no node
explosion, no regression — recorded in the matrix (§6) with the validating repo + numbers.
Then ship-prep: tests per mechanism, CHANGELOG, wire incremental, commit.
