# XenoAtom.Skill

A [Copilot CLI skill](https://agentskills.io) for [XenoAtom.Terminal.UI](https://github.com/XenoAtom/XenoAtom.Terminal.UI) — a .NET 10 retained-mode terminal UI framework.

## Installation

```bash
npx skills add kzu/XenoAtom.Skill
```

## Skill

**xenoatom-terminal-ui** — getting started, hosting modes (Write/Live/Run), async patterns, prompts, and ecosystem overview. Detailed reference files are loaded on demand:

| Reference | Description |
|-----------|-------------|
| `references/controls.md` | Complete controls reference: buttons, lists, layout containers, data display, overlays, navigation chrome, charts |
| `references/layout.md` | Visual tree, `State<T>`, binding, `ComputedVisual`, `ContentSwitcher`, data templating, layout protocol, custom controls |
| `references/styling.md` | Themes, per-control styles, dynamic styles, brushes/gradients, colors/RGBA, markup syntax, ANSI/VT, Nerd Font glyphs, `ControlTone` |
| `references/input.md` | Input pipeline, routed events (Direct/Preview/Bubble), custom event handling, focus, commands, `CommandBar`, context menus, `XenoAtom.CommandLine` integration |
| `references/text-editing.md` | `TextBox`, `TextArea`, `CodeEditor`, `PromptEditor`, `MaskedInput`, `NumberBox`, clipboard, undo/redo, Find/Replace, Go To Line |

## About XenoAtom.Terminal.UI

XenoAtom.Terminal.UI is a .NET 10 retained-mode terminal UI framework with:

- **Three hosting modes**: `Terminal.Write` (one-shot render), `Terminal.Live` (inline live), `Terminal.Run` (fullscreen)
- **Reactive state**: `State<T>` observable containers with automatic dependency tracking — no `RequestRender()` needed
- **Source generators**: `[Bindable]` and `[RoutedEvent]` attributes on `partial` types for zero-boilerplate binding
- **Rich control library**: 70+ controls from simple `TextBox` and `Button` to `DataGridControl`, `TreeView`, `CodeEditor`, and `ColorPicker`
- **Full theming**: markup syntax, custom brushes, gradients, RGBA colors, ANSI/VT passthrough, Nerd Font glyphs
- **Routed events**: Direct/Preview/Bubble strategies, identical to WPF routing but terminal-native

## License

See [LICENSE](LICENSE).
