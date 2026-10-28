# XenForo 2.3 AI Agent Skills
*This documentation is also available in: [Deutsch](README.de.md)*

![XenForo](https://img.shields.io/badge/XenForo-2.3%2B-orange.svg) ![PHP](https://img.shields.io/badge/PHP-8.2%2B-blue.svg) ![License](https://img.shields.io/badge/License-MIT-green.svg)

Vendor-neutral AI agent skills providing a strict engineering and architecture discipline layer for XenForo 2.3 add-on development. Designed to complement XenForo's official `php cmd.php xf-dev:install-ai-agents` (`xf-*`) skills without naming collisions.

## Included Skills

* **[`xf23-strict-policy`](skills/xf23-strict-policy/SKILL.md)** – Universal XenForo 2.3 guardrails:
  * **Co-Install Safety (`XFCP_`)**: Enforces 1:1 method signatures on class extensions so AI agents and linters do not add return types (`: void`, `: bool`) to untyped XenForo core methods.
  * **`Setup.php` Symmetry**: Requires full uninstall reversal for all created tables and altered core columns.
  * **Frontend & Vanilla JS**: Strict jQuery ban, native `XF.on(element, event, handler, options)` signature enforcement, `_files/js/` asset storage policy, and Dev-Import rules.
  * **UI & Icons**: Bans raw HTML `<table>` layouts in favor of native XenForo components (`structItem`, `<xf:datalist>`, `contentRow`, `<dl class="pairs">`), bans inline CSS/JS, and enforces FontAwesome 5.x icon names over hallucinated FA6 names.
  * **Security & Scaffolding**: PHP 8.2 baseline / PHP 8.4 zero-deprecation rules, strict ban on PHP shell execution functions (`exec()`, etc.), ban on ad-hoc entity scratch scripts, and `xf-make:*` / `~~Phrase~~` CLI priority.
* **[`xf23-custom-fields`](skills/xf23-custom-fields/SKILL.md)** – Step-by-step reference for implementing XenForo 2.3's 6-layer Custom Fields architecture (`Setup.php`, `AbstractField`, `FieldValue`, `XF:CustomFieldsHolder`, `Repository`, `app_setup` container, controllers, and macro invocation).

## Installation

Copy the skill folders from `skills/` into your XenForo project's `.agents/skills/` directory:

```bash
cp -r skills/xf23-* /path/to/xenforo/.agents/skills/
```

Add the skills to the skill table in your root `AGENTS.md`:

```markdown
| Skill | When to use |
|---|---|
| `xf23-strict-policy` | Always — universal XenForo 2.3 engineering discipline, `XFCP_` signature safety, `Setup.php` symmetry, Vanilla JS & UI standards |
| `xf23-custom-fields` | When implementing or extending XenForo 2.3 Custom Fields on any content entity |
```
