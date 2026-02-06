# Walker Config Reference

## Key Sources

- Official docs: `https://benz.gitbook.io/walker`
- General config docs: `https://benz.gitbook.io/walker/customization/general-config`
- Theme docs: `https://benz.gitbook.io/walker/customization/theme`
- Providers docs: `https://benz.gitbook.io/walker/providers`
- Upstream default config: `resources/config.toml` in Walker repo
- Upstream config merge logic: `src/config.rs` in Walker repo

## Core Paths

- Config file: `~/.config/walker/config.toml`
- Theme directory: `~/.config/walker/themes/<theme-name>/`
- Optional extra theme location key: `additional_theme_location`

## Common Commands

```bash
walker --help
elephant --help
elephant generatedoc
walker --gapplication-service
walker
nc -U /run/user/$(id -u)/walker/walker.sock
```

## High-Value Config Areas

### Top-level behavior keys

- `force_keyboard_focus`
- `close_when_open`
- `click_to_close`
- `single_click_activation`
- `selection_wrap`
- `global_argument_delimiter`
- `exact_search_prefix`
- `theme`
- `resume_last_query`
- `actions_as_menu`

### Provider routing and result shaping

- `[providers].default`
- `[providers].empty`
- `[providers].ignore_preview`
- `[providers].max_results`
- `[providers.argument_delimiter]`
- `[providers.max_results_provider]`
- `[[providers.prefixes]]`
- `[providers.sets.<name>]`

### Keybinds and actions

- `[keybinds]` controls global navigation and activation.
- `[providers.actions]` controls action rows per provider.
- Action defaults can be replaced or removed (`unset = true`) by user config.

### Theme files

- `style.css`
- `layout.xml`
- `keybind.xml`
- `preview.xml`
- `item_<provider>.xml`

Behavior:

- CSS updates are hot-reloaded.
- XML updates require restart/reload.

## Practical Snippets

### Add provider prefixes

```toml
[[providers.prefixes]]
prefix = ">"
provider = "runner"

[[providers.prefixes]]
prefix = "@"
provider = "websearch"
```

### Define a provider set

```toml
[providers.sets.work]
default = ["desktopapplications", "runner", "websearch"]
empty = ["desktopapplications"]
```

Run with:

```bash
walker -s work
```

### Customize provider actions

```toml
[providers.actions]
runner = [
  { action = "run", default = true, bind = "Return" },
  { action = "runterminal", label = "run in terminal", bind = "shift Return" }
]
```

## Troubleshooting Checklist

1. Confirm Elephant is running.
2. Confirm required providers are installed.
3. Verify provider names and prefixes in config.
4. Launch Walker from terminal and inspect output.
5. Use service mode for performance-sensitive workflows.
6. On NVIDIA, test `GSK_RENDERER=cairo`.
