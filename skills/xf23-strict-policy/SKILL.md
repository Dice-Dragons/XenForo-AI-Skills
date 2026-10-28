---
name: xf23-strict-policy
description: Universal strict engineering, architecture, security, UI, JavaScript, and XFCP compatibility guardrails for XenForo 2.3 add-on development. Load alongside xf-shared-policy and xf-addon-dev.
---

# XenForo 2.3 Strict Engineering & Discipline Policy

This skill defines the universal "discipline layer" for XenForo 2.3 add-on development, preventing common AI code-generation pitfalls ("vibe-coding traps"), silent frontend failures, and multi-add-on production crashes.

---

## PHP Version Policy (PHP 8.2+ Baseline, PHP 8.4 Ready)
- **Minimum Requirement**: PHP 8.2.0 is the baseline (`"php": ["8.2.0", "PHP 8.2.0+"]` in `addon.json`).
- **Syntax Compatibility**: Code MUST execute cleanly on PHP 8.2 without parse/syntax errors (do NOT use PHP 8.3/8.4-only syntax like Property Hooks or Asymmetric Visibility).
- **PHP 8.4 Optimization & Zero Deprecations**:
  - Explicitly declare nullable parameters (e.g. `?string $foo = null`, NEVER `string $foo = null`).
  - No dynamic properties on classes.
  - Strict type checking and null-safe arguments for internal PHP functions (e.g. avoid passing `null` to `trim()`, `substr()`, `explode()`).
  - Full typing on properties, parameters, and return types on own classes (subject to the strict `XFCP_` exception below).

---

## Security, Boundaries & Environment Guardrails

1. **Core & External Protection**: NEVER modify core XenForo files in `src/XF/`, `js/xf/`, `src/vendor/`, or third-party add-ons outside your target add-on namespace unless explicitly directed by the user.
2. **Immutability of `src/config.php`**: `src/config.php` is strictly immutable for AI agents. Modifying, editing, or overwriting `src/config.php` under any circumstances is strictly forbidden.
3. **Upstream Skill Immutability**: NEVER edit or customize official XenForo skills prefixed with `xf-*` (`.agents/skills/xf-*/`).
4. **WebServer & MySQL Offline Handling**: If a database connection error (MySQL connection refused / Gone Away) or web server unreachable error occurs, do NOT attempt workarounds, alternative configs, or bypass scripts. Inform the developer immediately that WebServer or MySQL appears to be stopped and wait until services have been started.
5. **Strict Ban on `exec()` & Shell Execution in PHP**: The use of `exec()`, `shell_exec()`, `system()`, `passthru()`, `popen()`, `proc_open()`, and the backtick execution operator (`` ` ``) in PHP code is **100% STRICTLY FORBIDDEN** under all circumstances — even in local development environments, CLI commands, migrations, or dev tools. All OS operations, git commands, and process executions belong strictly in the terminal environment, never inside PHP application code.

---

## Co-Install Safety & Lifecycle Integrity

1. **Strict `XFCP_` Class Extension Signature Preservation (Co-Install Safety)**:
   - When overriding methods in a XenForo Class Extension (`extends XFCP_...`), the method signature MUST match the parent method in `from_class` **1:1**.
   - **NEVER add a return type** (such as `: void`, `: bool`, `: Structure`, `: AbstractReply`) to an overridden `XFCP_` method if the parent method in `from_class` has **no** return type declared. Even though PHP allows return type covariance in isolated inheritance, adding a return type in XenForo's dynamic `XFCP_` proxy chain triggers a compile-time `Fatal Error` (`Declaration of ... must be compatible with ...`) as soon as a second add-on extends the same method using XenForo's native untyped signature.
   - **NEVER add or narrow parameter type hints** where the parent method parameter in `from_class` is untyped, and always preserve pass-by-reference (`&`) flags and optional defaults.
2. **`Setup.php` Install / Uninstall Schema Symmetry**:
   - Every database table created (`createTable`) and every column added to a core or foreign table (`alterTable` -> `addColumn`) in `installStep*` MUST be cleanly reverted (`dropTable` / `dropColumns`) in `uninstallStep*`.

---

## XenForo 2.3 Architecture & Core Pattern Rules

1. **Thin Controller & Service-Pattern**:
   * Controllers MUST stay thin and focus on orchestrating actions, verifying permissions, filtering inputs (`$this->filter(...)`), and returning standard view responses (`$this->view()`, `$this->redirect()`, `$this->error()`, `$this->notFound()`).
   * Complex business logic, multi-step write operations, and data processing MUST be encapsulated in **Service** classes (`src/addons/<Vendor>/<AddonName>/Service/`).

2. **Entity-Repository Boundary & Database Rules (`getStructure` & Native SQL)**:
   * Entities MUST define their schema, column types, and relations in `public static function getStructure(Structure $structure)`.
   * **Finder Priority**: Standard database queries MUST be built using chainable Finders (`\XF::finder(...)`) or Entity relations.
   * **Native SQL Standards (When Raw SQL is Permitted)**:
     - **Allowed Scenarios**: Native SQL is permitted ONLY for:
       - Mass operations (`UPDATE`, `DELETE`, `INSERT / Bulk-Insert`) where loading entities would cause memory/performance bottlenecks.
       - Complex aggregations, reports, subqueries, `GROUP BY ... HAVING`, or `UNION` queries unsupported by Finders.
       - Install/Upgrade/Uninstall steps in `Setup.php` and background processing in `Job`.
     - **Strict Location Rule**: Native SQL queries MUST reside exclusively inside **Repository** classes (`\XF\Mvc\Entity\Repository`), **Job** classes (`\XF\Job\AbstractJob`), or **Setup** classes (`\XF\AddOn\AbstractSetup`). Native SQL is **STRICTLY FORBIDDEN in Controllers, Views, or Templates**.
     - **SQL-Injection Prevention (Prepared Statements)**:
       - **NEVER concatenate dynamic variables directly into SQL queries** (`"WHERE user_id = " . $userId` is strictly forbidden).
       - ALWAYS use prepared statements with parameter placeholders (`?`) and bind arrays:
         ```php
         $db = $this->db(); // In Repository / Setup / Job, or \XF::db()
         $count = $db->fetchOne('SELECT COUNT(*) FROM xf_table WHERE user_id = ? AND status = ?', [$userId, $status]);
         $row   = $db->fetchRow('SELECT * FROM xf_table WHERE table_id = ?', [$id]);
         $rows  = $db->fetchAll('SELECT * FROM xf_table WHERE date > ? ORDER BY date DESC', [$cutOff]);
         $pairs = $db->fetchPairs('SELECT table_id, title FROM xf_table WHERE active = ?', [1]);
         $ids   = $db->fetchCol('SELECT table_id FROM xf_table WHERE active = ?', [1]);
         ```
       - **Array Lists (`IN (...)`)**: Use `$db->quote($ids)` for lists, never unescaped `implode()`:
         ```php
         $rows = $db->fetchAll('SELECT * FROM xf_table WHERE user_id IN (' . $db->quote($userIds) . ')');
         ```
     - **Adapter Helper Methods for Writes**: Prefer native adapter write helpers:
       ```php
       $db->update('xf_table', ['status' => 'inactive'], 'user_id = ?', [$userId]);
       $db->delete('xf_table', 'cut_off_date < ?', [$cutOff]);
       $db->insert('xf_table', ['user_id' => $userId, 'title' => $title]);
       $db->insertOnDuplicate('xf_table', ['user_id' => $userId, 'count' => 1], ['count' => new \XF\Db\Sql\Expression('count + 1')]);
       ```
   * **Repositories (`\XF\Mvc\Entity\Repository`)**: Reserved for database tables and entities (e.g. `xf_icon_usage` in core `IconRepository`).
   * **Helper Classes (`src/addons/<Vendor>/<AddonName>/Helper/`)**: Static helper classes for file system indexing, registry/cache management, or utility logic without DB tables.

3. **HTML Boundary & Template Tag Standards**:
   * **HTML MUST BE CREATED ONLY IN TEMPLATES**: NEVER construct or concatenate HTML strings inside PHP classes, controllers, services, or option handlers.
   * **STRICT BAN ON RAW HTML `<table>` & Mandate for XenForo UI Components**:
     - Raw HTML tables (`<table>`, `<tr>`, `<td>`) are **STRICTLY FORBIDDEN** in XenForo templates for general data display, lists, and layout. Custom HTML tables break mobile responsiveness, ignore XenForo style palettes/variations, and violate core UI standards.
     - Agents MUST use XenForo's native responsive layout structures instead:
       - **Public Item Lists**: ALWAYS use **`structItemContainer` / `structItem`** (`.structItem-cell--icon`, `.structItem-cell--main`, `.structItem-cell--meta`, `.structItem-cell--latest`) for thread-like, resource-like, or content listings.
       - **Tabular & Management Data**: ALWAYS use **`<xf:datalist>` / `<xf:datarow>` / `<xf:maincell>`** (in AdminCP and Public) for data rows, management grids, and status tables.
       - **Compact Figure/Media Items**: Use **`contentRow`** (`.contentRow-figure`, `.contentRow-main`, `.contentRow-header`, `.contentRow-extra`) for search items, member lists, and widgets.
       - **Key-Value Attributes & Stats**: ALWAYS use definition pairs **`<dl class="pairs pairs--columns">`** or `pairs--justified` instead of two-column tables.
   * **Always Use Native XenForo Template Tags**:
     - **Forms**: ALWAYS use native XenForo form tags (`<xf:form>`, `<xf:textboxrow>`, `<xf:selectrow>`, `<xf:checkboxrow>`, `<xf:submitrow>`).
     - **Icons & Buttons**: ALWAYS use `<xf:fa icon="..." />` and `<xf:button href="..." icon="...">`.
   * PHP code prepares clean data structures/arrays and renders XenForo templates (`$templater->renderTemplate('admin:template_name', $viewParams)`).

4. **FontAwesome Icon Validation & Constants**:
   * Always reuse native constants from `\XF\Repository\IconRepository` for icon parsing and validation instead of custom regexes or hardcoded arrays:
     * `IconRepository::ICON_VARIANTS` (`['fab' => 'brands', 'fad' => 'duotone', 'fal' => 'light', 'far' => 'regular', 'fas' => 'solid']`)
     * `IconRepository::ICON_CLASS_BLOCKLIST_REGEX`
     * `IconRepository::ICON_CLASS_REGEX`

---

## Frontend JavaScript & LESS Best Practices (XenForo 2.3)

1. **STRICT BAN ON INLINE CSS**:
   - Inline `style="..."` attributes and `<style>` tags are **STRICTLY FORBIDDEN** in XenForo templates and HTML output.
   - ALL styling MUST reside exclusively in `.less` templates (e.g. `_output/templates/public/<prefix>_<name>.less` or `admin/<prefix>_<name>.less`).
   - Stylesheets MUST be included in templates via `<xf:css src="<prefix>_<name>.less" />`.
   - Dynamic visual states MUST be achieved using CSS classes (e.g. `class="{{ $isActive ? 'is-active' : '' }}"`) or CSS custom properties defined in LESS stylesheets, never raw inline styles.

2. **STRICT BAN ON INLINE JAVASCRIPT & Native XenForo Component Pattern**:
   - Inline `<script>` tags, inline event attributes (`onclick`, `onchange`, `onsubmit`, `onload`, etc.), and inline JS snippets in templates are **STRICTLY FORBIDDEN**.
   - ALL JavaScript MUST be implemented and registered the official XenForo way:
     - Structured as modular external handlers using `XF.Element.newHandler('<prefix>-<name>', { ... })` or click handlers `XF.Click.newHandler('<prefix>-<name>', { ... })`.
     - Attached to DOM elements via HTML data attributes: `data-xf-init="<prefix>-<name>"` or `data-xf-click="<prefix>-<name>"`.
     - Included in templates via `<xf:js src="<vendor>/<filename>.js" min="1" addon="<Vendor>/<AddOnName>" />`.
     - Stored exclusively in `src/addons/<Vendor>/<AddOnName>/_files/js/<vendor>/<filename>.js`. NEVER in root `js/`!
     - Registered in `build.json` under `"additional_files"` and `"minify"`.

3. **STRICT JQUERY BAN & Promise Standard**:
   - jQuery is **STRICTLY FORBIDDEN** in XenForo 2.3 add-ons. Use ONLY Vanilla JS (ES2020) and native `XF.*` APIs (`XF.Element.newHandler`, `XF.on`, `XF.ajax`, `XF.setupHtmlInsert`, `XF.Animate`).
   - `XF.ajax()` in XF 2.3 returns native ES6 Promises; ALWAYS use `.then((response) => ...)` and `.catch((error) => ...)`. Legacy jQuery Deferred methods (`.done()`, `.always()`) are strictly forbidden.

4. **`XF.on` Event Signature & Delegation Rules**:
   - `XF.on(element, namespacedEvent, handler, options)` expects the event listener **Function** as parameter 3 (NEVER pass a selector string as parameter 3 like jQuery's `$(el).on('click', 'selector', fn)`).
   - Passing a string as parameter 3 causes `addEventListener` to crash with `Uncaught TypeError: parameter 2 is not of type 'Object'`, which aborts XenForo's entire handler loop and breaks standard UI controls (like collapsibles / toggles).
   - For event delegation inside a container/form, check the target inside the handler function using `e.target && e.target.matches('selector')` or `e.target.closest('selector')`.

5. **Style Variations & Dark Mode Compatibility**:
   - All LESS/CSS files MUST use native XenForo CSS variables (`var(--xf-...)` / `@xf-textColor`, `@xf-contentBg`, `@xf-borderColor`, `@xf-chipBg`, etc.). NEVER use hardcoded hex colors (`#fff`, `#000`, etc.) for background or text colors, ensuring full compatibility with Light and Dark style variations.

6. **Strict FontAwesome 5.x Standard & Ban on FA6+ Icons**:
   - In templates, options, style properties, and BBCode, use **EXCLUSIVELY the FontAwesome 5.x icons included in XenForo**.
   - **Strict Ban on FA6+ Icons & Names**: FontAwesome 6 or higher icons and FA6-specific icon names (e.g. `fa-gear` -> use `fa-cog`, `fa-circle-question` -> use `fa-question-circle`, `fa-shield-halved` -> use `fa-shield-alt`) are **STRICTLY FORBIDDEN**. XenForo does not ship or render FA6 icon names.

7. **JavaScript Storage, Asset Resolution & Dev-Import Rules**:
   - All add-ons MUST store and edit JavaScript files exclusively inside `src/addons/<Vendor>/<AddOnName>/_files/js/...`.
   - **Strict Ban on Searching or Editing Root `js/` Directory**: AI agents MUST NEVER search for, edit, or create add-on files in the root `/js/` directory. Root `/js/` is strictly reserved for core XenForo and production release artifacts.
   - **No `.min.js` in `_files/`**: Store ONLY the raw, unminified `.js` source files in `_files/js/`. Minified `.min.js` files are generated by XenForo during `build-release` into the root `js/` directory.
   - **No Dev-Import for Pure JS & Templates**: In Development Mode, pure JavaScript changes in `_files/js/` and template changes in `_output/templates/` do NOT require running `php cmd.php xf-dev:import`. XenForo loads JS dynamically and compiles templates directly from the filesystem on page reload. Do NOT run `xf-dev:import` after routine template or JS edits.
   - **When `xf-dev:import` IS required**: Only for database-backed metadata (Template Modifications in `_output/template_modifications/`, Phrases, Options, Permissions, Style Properties, Class Extensions) or during release preparation.
   - **XenForo 2.3 Asset Resolution**: When the `addon="<Vendor>/<AddOnName>"` attribute is set in `<xf:js>`, XenForo automatically resolves and serves the JavaScript file directly from the add-on's `_files/` directory in development mode and uses `min` to resolve `.min.js` in production.
   - **`build.json` Registration**: All add-on JavaScript files MUST be listed under `"additional_files"` and `"minify"` in `build.json` so `php cmd.php xf-addon:build-release` copies files from `_files/js/` to `js/` and generates minified release assets.

8. **XenForo 2.3 JS & LESS Helper Conventions**:
   - **URL Building**: `XF.buildLink` is NOT present in JS. Use `XF.canonicalizeUrl((typeof XF.getApp === 'function' && XF.getApp() === 'admin' ? 'admin.php' : 'index.php') + '?route-prefix/action')`.
   - **Standalone SVG & Color Inheritance**: For autocomplete previews of un-sprited icons, use `XF.Icon.getStandaloneIconUrl(variant, name)` with CSS mask: `mask: url('...') center/contain no-repeat; background-color: currentColor;` to inherit text/theme color.
   - **Dropdown Transitions**: Use `XF.Animate.fadeDown(element)` and `XF.Animate.fadeUp(element)`. Disable live spin animations (`fa-spin`) in search result lists to preserve UI performance.
   - **CSS/LESS Selectors in XenForo 2.3**: In XenForo 2.3 SVG rendering, `<xf:fa>` outputs `<svg class="fa--xf ...">` elements, NOT `<i>` elements. Always update LESS selectors to target `svg, i, .fa--xf`.

---

## Anti-Scratch Script Policy & Scaffolding Standards (`xf-make:*` & `_output`)

1. **Strict Prohibition of Scratch Entity Scripts**:
   - **NEVER** write or execute ad-hoc PHP scratch scripts (`scratch_*.php`, CLI scripts bootstrapping `\XF::start(...)`) that instantiate, mutate, save, or delete XenForo entities (`\XF::em()->create(...)`, `$entity->save()`, `\XF::db()->query(...)`, `\XF::db()->insert(...)`).
   - **Why this is forbidden**: Scratch scripts bypass XenForo's development mode metadata system, desynchronize dev-output files (`_output/`) from the database, leave untracked database state, and break Git reproducibility.
   - **Where Entity Operations Belong**: Entity creation and data changes belong exclusively in:
     - `Setup.php` (install, upgrade, uninstall migrations)
     - Standard application flows (Repositories, Services, Jobs, Controllers)
   - Scratch scripts are permitted ONLY for read-only inspection/debugging when no CLI alternative exists.

2. **Scaffolding Priority (`php cmd.php xf-make:*`)**:
   Always prioritize official XenForo CLI generators and native `_output/` filesystem artifacts over manual class creation or database manipulation:

   | Task | Recommended CLI Command / Filesystem Path |
   |---|---|
   | **Phrases** | `php cmd.php xf-make:phrase <Vendor>/<Addon> phrase_title --text="..."` |
   | **Entities** | `php cmd.php xf-make:entity <Vendor>/<Addon> EntityName` |
   | **Public Controller** | `php cmd.php xf-make:controller <Vendor>/<Addon> Pub ControllerName` |
   | **Admin Controller** | `php cmd.php xf-make:controller <Vendor>/<Addon> Admin ControllerName` |
   | **API Controller** | `php cmd.php xf-make:controller <Vendor>/<Addon> Api ControllerName` |
   | **Repository** | `php cmd.php xf-make:repository <Vendor>/<Addon> RepositoryName` |
   | **Service** | `php cmd.php xf-make:service <Vendor>/<Addon> "ServiceName\Action"` |
   | **Job** | `php cmd.php xf-make:job <Vendor>/<Addon> JobName` |
   | **Class Extension** | `php cmd.php xf-make:class-extension <Vendor>/<Addon> TargetClass ExtensionClass` |
   | **Templates** | Authored directly in `_output/templates/<type>/<name>.<ext>` |
   | **Template Mod.** | Authored in `_output/template_modifications/<type>/<key>.json` |

---

## Entity Feature Discovery Rule (Mandatory for Entity Creation)

When creating or extending content entities (`XF\Mvc\Entity\Entity` / `Structure`), **ALWAYS ask the user** (if not specified in the initial prompt) which features the entity should support, or **propose the most sensible features** for that entity type:

1. **`Reactionable` / `Likable`**: `XF:Reactionable` behavior & reaction handlers for content appreciation.
2. **`FeaturedContent`**: XenForo 2.3 `XF:FeaturedContent` handler for global homepage slider/feature promotion.
3. **`Bookmarkable`**: `XF:Bookmarkable` behavior & handler for user bookmarks.
4. **`Taggable`**: `XF:Taggable` behavior & handler for content tag management.
5. **`AttachmentContainer`**: `XF:AttachmentContainer` for attaching files/images.
6. **`Searchable`**: `XF:Searchable` search handler for global forum search indexing.
7. **`SoftDeletable` / `State`**: Soft delete, approval queue, or moderation state behavior.
8. **`NewsFeedContainer` / `ActivityLog`**: Publishing activity items to the global news feed.
9. **`Webhooks`**: Registering webhook events in `_output/webhook_events/` for external integrations (Discord, Zapier, Slack).

---

## Template & Phrase Guidelines (XenForo 2.3.10+)

1. **Mandatory Phrase Creation Workflow (CLI & Template Tilde Markers)**:
   - **Mandatory Creation via CLI Only**: During active development, phrases MUST be created **WITHOUT EXCEPTION** through the official CLI tools using one of two methods:
     1. **Direct CLI Generation**:
        ```bash
        php cmd.php xf-make:phrase <Vendor>/<AddonName> <phrase_title> --text="English master text"
        ```
     2. **Template Tilde Markers (`~~Phrase~~`) with Subsequent CLI Extraction**:
        Enclose translatable English text directly in templates or template modifications with double tildes:
        ```html
        <xf:title>~~Edit note <b>{$note.title}</b>~~</xf:title>
        ```
        and subsequently extract and generate the master phrases via CLI (always run `--dry-run` first to preview extracted phrases):
        ```bash
        php cmd.php xf-dev:template-phraser --addon=<Vendor>/<AddonName> --phrase-prefix=<prefix>_
        php cmd.php xf-dev:template-modification-phraser --addon=<Vendor>/<AddonName> --phrase-prefix=<prefix>_
        ```
   - **Prohibition on Manual Filesystem & Database Creation**: Direct manual creation of `.txt` files in `_output/phrases/` and writing directly into the database (`xf_phrase`) via scratch PHP scripts or raw DB queries is strictly prohibited.
   - **Master Language STRICTLY English (`language_id = 0`)**: All master phrases (`language_id = 0` / `_output/phrases/*.txt`) MUST EXCLUSIVELY be authored in **English**.

2. **XenForo Native Component & Templater Standards**:
   * Always rely on native XenForo components and `$templater` methods instead of creating custom manual HTML strings.
   * **FontAwesome Icons**: Use `$templater->fontAwesome('fa-icon-name', ['class' => '...'])` in PHP or `<xf:fa icon="..." />` in templates. This routes icons through XenForo 2.3's native `IconRenderer` (SVG rendering).
   * **Buttons**: Use `$templater->button(\XF::phrase('...'), ['href' => '...', 'icon' => '...'])` in PHP or `<xf:button>` in templates.

3. **Template Authoring Best Practices**:
   * **Explicit `~~` Tilde Delimiters**: Mark phrasable text blocks explicitly using the `~~` delimiter pattern in templates (e.g. `~~Edit note <b>{$note.title}</b>~~`).
   * **Clean Text Node Wrapping**: Enclose user-facing strings directly inside explicit HTML tags or attribute values (e.g. `<label>Text</label>`, `title="Text"`), avoiding arbitrary inline tag mixing.
   * **Multi-line & HTML Formatting**: Enclose multi-line or HTML-formatted text blocks cleanly within tag boundaries so the scanner extracts the phrase intact without breaking markup.
   * **Variable Interpolation**: Express dynamic variables within text strings using clean token patterns (e.g. `{name}` or `{$var}`) so the phraser generates single, parameterized phrases instead of fragmenting text around variables.
   * **Reuse Existing Phrases**: Review `--dry-run` suggestions to reuse existing matching phrases before creating duplicate phrase entries.
