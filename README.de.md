# XenForo 2.3 AI Agent Skills
*Diese Dokumentation ist auch verfügbar auf: [English](README.md)*

![XenForo](https://img.shields.io/badge/XenForo-2.3%2B-orange.svg) ![PHP](https://img.shields.io/badge/PHP-8.2%2B-blue.svg) ![License](https://img.shields.io/badge/License-MIT-green.svg)

Herstellerneutrale KI-Agenten-Skills als strikter „Discipline Layer“ für die Entwicklung von XenForo 2.3 Add-ons. Konzipiert als direkte Ergänzung zu den offiziellen XenForo-Skills (`php cmd.php xf-dev:install-ai-agents` / `xf-*`) ohne Namenskollisionen.

## Enthaltene Skills

* **[`xf23-strict-policy`](skills/xf23-strict-policy/SKILL.md)** – Universelle XenForo 2.3 Entwicklungs- und Architektur-Guardrails:
  * **Co-Install-Sicherheit (`XFCP_`)**: Erzwingt 1:1-Methodensignaturen in Class Extensions, damit KI-Agenten und Linter keine Return-Types (`: void`, `: bool`) auf untypisierten XenForo-Core-Methoden ergänzen.
  * **`Setup.php`-Symmetrie**: Stellt sicher, dass alle angelegten Tabellen und erweiterten Core-Spalten bei der Deinstallation rückstandsfrei entfernt werden.
  * **Frontend & Vanilla JS**: Strikter jQuery-Verzicht, Schutz vor der `XF.on(element, event, handler, options)`-Signaturfalle, `_files/js/`-Ablageregel und Dev-Import-Regeln.
  * **UI & Icons**: Verbot von rohen HTML-`<table>`-Layouts zugunsten nativer XenForo-Komponenten (`structItem`, `<xf:datalist>`, `contentRow`, `<dl class="pairs">`), Verbot von Inline-CSS/JS sowie Beschränkung auf FontAwesome 5.x Icons (Schutz vor halluzinierten FA6-Klassennamen).
  * **Security & Scaffolding**: PHP 8.2 Baseline / PHP 8.4 Zero-Deprecation-Regeln, striktes Verbot von PHP-Shell-Funktionen (`exec()` etc.), Verbot von Scratch-Entity-Skripten und Vorrang für `xf-make:*` sowie `~~Phrase~~`-CLI-Extraktion.
* **[`xf23-custom-fields`](skills/xf23-custom-fields/SKILL.md)** – Vollständiger Leitfaden zur Implementierung der 6-Schichten-Architektur für eigene XenForo 2.3 Custom Fields (`Setup.php`, `AbstractField`, `FieldValue`, `XF:CustomFieldsHolder`, `Repository`, `app_setup`-Container, Controller und Makro-Einbindung).

## Installation

Kopiere die Skill-Ordner aus `skills/` in das `.agents/skills/`-Verzeichnis deiner XenForo-Entwicklungsumgebung:

```bash
cp -r skills/xf23-* /pfad/zu/xenforo/.agents/skills/
```

Ergänze die Skills anschließend in der Skill-Tabelle deiner zentralen `AGENTS.md`:

```markdown
| Skill | When to use |
|---|---|
| `xf23-strict-policy` | Always — universal XenForo 2.3 engineering discipline, `XFCP_` signature safety, `Setup.php` symmetry, Vanilla JS & UI standards |
| `xf23-custom-fields` | When implementing or extending XenForo 2.3 Custom Fields on any content entity |
```
