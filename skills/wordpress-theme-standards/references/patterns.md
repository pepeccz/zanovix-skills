# WordPress Theme Standards — Patterns & Tooling

Detail behind the SKILL.md hard rules. Read when scaffolding tooling or when a rule needs the concrete pattern.

## Tooling recipe

Standard setup (any environment with system PHP + Composer):

```
composer require --dev wp-coding-standards/wpcs phpcompatibility/phpcompatibility-wp squizlabs/php_codesniffer
vendor/bin/phpcs -i          # confirm WordPress + PHPCompatibilityWP appear
vendor/bin/phpcs             # uses phpcs.xml at the theme root
vendor/bin/phpcbf            # auto-fix whitespace/formatting
vendor/bin/phpcs --standard=PHPCompatibilityWP --runtime-set testVersion 7.4- .
```

Local by Flywheel (no system PHP; binary is Debian-built): use the bundled PHP and point `LD_LIBRARY_PATH` at its shipped libs — but scope that env var INLINE to php calls only; exporting it breaks system `curl`/`git`.

```
LD_LIBRARY_PATH=<Local>/lightning-services/php-<ver>/bin/linux/shared-libs \
  <Local>/lightning-services/php-<ver>/bin/linux/bin/php vendor/bin/phpcs
```

Live DB scripts on Local also need `-d mysqli.default_socket=<run>/mysql/mysqld.sock` (and the `pdo_mysql` twin); re-derive the run path from `pgrep -af mysqld`.

## The Generic.PHP.Syntax trap (critical)

PHPCS 3.13.x `Generic.PHP.Syntax` calls `trim(null)` on PHP 8.2+, throws, and PHPCS reports `Internal.Exception` + "checking has been aborted" — silently skipping the rest of that file, so `WordPress.Security.*` sniffs never run and you get a false "0 findings". Always exclude the sniff in `phpcs.xml` (or pin `squizlabs/php_codesniffer:3.10.*`). Verify security sniffs actually fire, e.g. `phpcs --sniffs=WordPress.Security.EscapeOutput` should report real findings on unescaped output.

## Upgrade-safe patterns (do this)

- Custom Elementor widgets: extend only public APIs (`\Elementor\Widget_Base`, `elementor/widgets/register`, public `Controls_Manager` constants). Never reference private/internal classes.
- Register/enqueue behind `class_exists`/`function_exists` guards; degrade gracefully when a plugin (ACF, Polylang, WS Form, Elementor) is inactive.
- Cache-bust assets with `filemtime()`, not hardcoded version strings.
- Guard-before-bootstrap for any CLI/migration script: `require __DIR__.'/_guard.php';` (CLI/`WP_CLI` gate that 403s web requests) as the FIRST executable line, THEN load WordPress. Keep the direct-`php script.php` workflow working; do not force `wp eval-file` unless wp-cli is guaranteed present.
- Resolve pages by slug through ONE cached helper (`get_page_by_path()` + Polylang `pll_get_post()`), with an explicit id-keyed registry entry for any slug that is not unique per language. Log misses under `WP_DEBUG`.
- When removing a parent-theme callback, guard it: `if ( function_exists($cb) && has_action($hook,$cb) ) remove_action(...)`, and log the unexpected-missing case so a parent rename becomes observable instead of silently resurrecting parent chrome.
- Resolve plugin runtime handles dynamically (e.g. `get_option('elementor_active_kit')`), never hardcoded ids like `elementor-kit-191`.

## Anti-patterns (never ship)

- Web-executable, DB-mutating scripts under webroot with no guard.
- Hardcoded environment-specific post IDs / URLs / domains (break on DB clone/migration); inline API keys or secrets.
- `remove_action` on parent/plugin internals by name without a guard.
- Business logic living in the Code Snippets plugin (stored in the DB, outside git, unreviewable) — put it in `inc/` modules or a tracked mu-plugin.
- Blanket `// phpcs:ignoreFile` or an ignore without a `-- reason`. Note: `--` must be ASCII, not an em-dash (an em-dash silently disables the ignore).
- Duplicate array keys (e.g. in an i18n map) — the later value silently overwrites; `Universal.Arrays.DuplicateArrayKey` catches these.

## Escaping & nonce discipline

- Per-finding review, not blanket wrapping. Inspect the value's source before choosing: `esc_html` on plain text is output-identical; on HTML it strips; wrapping already-`wp_kses_post`'d rich text double-escapes and mangles it.
- i18n echoes: `esc_html_e()` / `echo esc_html__()`.
- State-changing handler order: check nonce → `current_user_can()` → `wp_unslash()` → `sanitize_*()` → act. Add `wp_nonce_field()` at the matching render site. A state-changing GET behind capability-only is still a CSRF gap (use `check_admin_referer()` + `wp_nonce_url()` if the trigger is ever hyperlinked).
- DB: `$wpdb->prepare()` for all interpolation; `$wpdb->esc_like()` on the term (not the whole pattern) before appending `%`.

## Note on Theme Check vs Plugin Check

Plugin Check targets plugins. For themes, the equivalents are **Theme Check** (wp-admin plugin) + WPCS via `phpcs.xml`. Both enforce the same WordPress Coding Standards.
