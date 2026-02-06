---
name: walker-linux-config
description: Configure and customize Walker on Linux, including `~/.config/walker/config.toml`, provider behavior, keybinds, prefixes, sets, actions, themes, and launch/service setup with Elephant. Use when users ask to install Walker, tune launcher behavior, fix config issues, migrate settings, customize theme/layout XML/CSS, or troubleshoot slow startup and provider integration.
---

# Walker Linux Config

Configure Walker with a practical workflow grounded in upstream docs and defaults from the Walker source tree.

## Workflow

1. Identify the user's setup and goal.
2. Inspect current Walker and Elephant configuration.
3. Propose minimal, concrete config changes.
4. Apply edits in the correct location.
5. Validate behavior and provide rollback-friendly next steps.

## 1) Identify Setup and Goal

Collect enough context to avoid wrong assumptions:

- Install method: distro package or source build.
- Session stack: Wayland/X11, desktop/compositor, distro family.
- Whether Elephant is running and which providers are installed.
- Desired outcome: behavior, keybinds, providers, theme, menus, performance, troubleshooting.

If the user request is clear, proceed directly and infer reasonable defaults.

## 2) Inspect Config and Runtime State

Primary paths and facts:

- Main config path: `~/.config/walker/config.toml`.
- Theme path: `~/.config/walker/themes/<theme-name>/`.
- Default reference config: `resources/config.toml` in Walker source.
- Provider docs source: `elephant generatedoc` (reflects installed providers).

When debugging startup or integration issues, verify:

- `elephant` service status.
- Walker launch mode (`walker` vs `walker --gapplication-service`).
- Provider availability and prefixes.

## 3) Customize with Small, Targeted Edits

Prioritize edits that map directly to the user request.

### Behavior and UX

Tune keys like:

- `close_when_open`, `click_to_close`, `selection_wrap`, `resume_last_query`.
- `global_argument_delimiter`, `exact_search_prefix`.
- `hide_quick_activation`, `hide_action_hints`, `hide_return_action`.

### Providers, Prefixes, and Sets

Use:

- `[providers]` for `default`, `empty`, `max_results`, `ignore_preview`.
- `[[providers.prefixes]]` for explicit provider routing.
- `[providers.sets.<name>]` with `walker -s <name>` for scenario-specific workflows.
- `[providers.max_results_provider]` for per-provider caps.

### Actions and Keybinds

Use:

- `[keybinds]` for global navigation and activation keybinds.
- `[providers.actions]` for per-provider action bindings.

Important merge behavior from code:

- Provider actions merge by `action` name.
- `unset = true` removes a default action.
- Prefixes merge by provider; redefining a provider prefix overrides its default prefix.

### Theming

Theme files are composable; only create what is needed:

- `style.css`
- `layout.xml`
- `keybind.xml`
- `preview.xml`
- `item_<provider>.xml`

Operational detail:

- CSS changes do not require Walker restart.
- XML changes require Walker restart/reload.

## 4) Validate and Troubleshoot

After edits, verify fast and in order:

1. Config parses and Walker starts.
2. Prefixes route queries to intended providers.
3. Keybinds and provider actions behave as expected.
4. Theme changes appear in UI.

For performance complaints:

- Prefer service mode: `walker --gapplication-service`.
- Optionally use socket activation (`nc -U /run/user/<uid>/walker/walker.sock`).
- On NVIDIA, try `GSK_RENDERER=cairo`.

If Elephant integration fails:

- Ensure Elephant is running.
- Confirm required providers are installed/enabled.
- Regenerate provider docs with `elephant generatedoc` and align config names.

## References

- Read `references/walker-config-reference.md` for default keys, command patterns, and high-signal config snippets.
