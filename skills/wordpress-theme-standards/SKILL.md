---
name: wordpress-theme-standards
description: "Trigger: building/editing a custom WordPress theme or child theme, WPCS, phpcs.xml, Theme Check, escape output, nonce. Write WP theme PHP that passes WordPress Coding Standards + Theme Check and stays upgrade-safe."
license: MIT
metadata:
  author: "pepeccz"
  version: "1.0"
---

# WordPress Theme Standards

## Activation Contract

Load when writing or reviewing custom WordPress **theme / child-theme** PHP (templates, `functions.php`, `inc/`, custom widgets) that must follow WordPress Coding Standards and survive plugin/parent-theme updates. Not for plugin-only work (use Plugin Check there) or non-WordPress PHP.

## Hard Rules

- Escape EVERY output at the point of output; never trust stored, meta, option, or request values. Choose the escaper per finding — never blanket-wrap, and never `esc_html()` a value that legitimately contains HTML (strips it) nor double-wrap already-escaped rich text.
- Before any state change: verify a nonce, check `current_user_can()`, then `wp_unslash()` + `sanitize_*()`. Never gate a plugin-rendered form you cannot pair with a `wp_nonce_field()`.
- Never leave a web-reachable, WP-bootstrapping script under webroot without a CLI/`ABSPATH` guard as its FIRST executable line (guard before bootstrap).
- No hardcoded post IDs, URLs, domains, or secrets. Resolve pages by slug (`get_page_by_path()` + a cached helper).
- Guard every dependency on a parent-theme or plugin internal (`function_exists`/`class_exists`/`has_action`); degrade, never fatal.
- `phpcs` (WordPress standard, `Generic.PHP.Syntax` EXCLUDED) MUST pass before commit. Never let `Generic.PHP.Syntax` run — it crashes on PHP 8.2+ and silently aborts scanning, hiding security findings.
- Do NOT restate WPCS rules in prose or code comments — `phpcs.xml` is the executable source of truth.

## Decision Gates

| Output context | Escaper |
|---|---|
| plain text node | `esc_html()` / `esc_html__()` / `esc_html_e()` |
| HTML attribute | `esc_attr()` / `esc_attr__()` |
| URL (href/src) | `esc_url()` (`esc_url_raw()` for storage) |
| known rich text (post content) | `wp_kses_post()` |
| inline JS / JSON-LD | `esc_js()` / `wp_json_encode()` + neutralise `</` → `<\/` |
| already-escaped upstream | `// phpcs:ignore <sniff> -- <specific reason>` |

## Execution Steps

1. Scaffold: copy `assets/phpcs.xml.starter` to the theme root; set the real text domain(s) and function prefixes.
2. Install tooling (`wp-coding-standards/wpcs`, `phpcompatibility/phpcompatibility-wp`, `squizlabs/php_codesniffer`). For Local by Flywheel, use the bundled-PHP recipe in `references/patterns.md`.
3. Write code applying the escaper / nonce / guard / no-hardcoded-ID hard rules from the first line — hardening is cheaper than remediation.
4. Run `phpcs --standard=phpcs.xml` before every commit; fix or justify each finding. Run PHPCompatibility for the target PHP range.
5. Run **Theme Check** in wp-admin (the theme analog of Plugin Check).
6. Verify on a running site: no inline PHP errors, no double-encoded entities (`&amp;lt;`), no broken/`?p=0` links.

## Output Contract

Theme PHP that: passes `phpcs` (WordPress standard, `Generic.PHP.Syntax` excluded) with zero real security findings; is PHPCompatibility-clean for the target range; passes Theme Check; and where every `phpcs:ignore` carries a concrete reason. `phpcs.xml` is committed at the theme root.

## References

- `assets/phpcs.xml.starter` — ready-to-use WPCS ruleset with the load-bearing exclusions.
- `references/patterns.md` — tooling recipe (incl. Local bundled PHP), upgrade-safe patterns, anti-patterns, escaping/nonce discipline.
