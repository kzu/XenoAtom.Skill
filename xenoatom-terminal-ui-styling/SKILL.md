---
name: XenoAtom.Terminal.UI — Styling & Markup
description: Themes, styles, brushes, gradients, color schemes, markup syntax, and ANSI/VT primitives in XenoAtom.Terminal.UI.
---

## When to use

Use this skill when the user needs help with:
- Applying and customizing themes (`Theme.Default`, `Theme.Terminal`, custom color schemes)
- Per-control styles (`ButtonStyle`, `TextBoxStyle`, `BorderStyle`, etc.)
- Dynamic/computed styles driven by state
- Linear gradient brushes on text controls
- Color creation (RGB, RGBA, basic-16, alpha blending)
- Markup syntax (`[bold red]...[/]`) for terminal output and `Markup` control
- ANSI/VT primitives via `XenoAtom.Ansi`
- Glyphs and `Rune` values for custom control rendering

## Instructions

### Themes

A `Theme` is a set of semantic style tokens used as the default environment for a visual subtree.

- **`Theme.Default`** — RGB dark theme (use for fullscreen apps)
- **`Theme.DefaultLight`** — RGB light theme
- **`Theme.Terminal`** — uses the terminal's own default colors (use for inline/live widgets)

Themes are resolved from the environment (`Visual.GetTheme()`). Override for a subtree:

```csharp
using XenoAtom.Terminal.UI.Controls;
using XenoAtom.Terminal.UI.Styling;

var themed = new Group(new TextArea("Themed region"))
    .Style(Theme.FromScheme(ColorScheme.CherryDark));
```

### Theme design tokens (semantic)

| Token | Purpose |
|-------|---------|
| `Surface`, `PopupSurface` | Background surfaces |
| `ControlFill`, `ControlFillHover`, `ControlFillPressed` | Button-like fills |
| `InputFill`, `InputFillFocused` | Text input surfaces |
| `Border`, `FocusBorder` | Stroke colors |
| `Accent`, `Selection` | Key interaction colors |

`PopupSurface` in dark RGB themes is intentionally close to the app background (not "lifted") so dialogs/popups don't look washed out.

### Per-control styles

Each control has a corresponding `*Style` record. Apply statically:

```csharp
new Button("OK").Style(new ButtonStyle { Tone = ControlTone.Primary });
```

Style variations with `with`:

```csharp
var danger = ButtonStyle.Default with { Tone = ControlTone.Error, ShowBorder = true };
new Button("Delete").Style(danger);
```

Styles are `record` types — always use `with` for modifications.

### Dynamic styles driven by state

Computed style from a factory (dependency-tracked):

```csharp
var isDanger = new State<bool>(false);

new Button("Deploy")
    .Style(() => isDanger.Value
        ? ButtonStyle.Default with { Tone = ControlTone.Error }
        : ButtonStyle.Default);
```

Style from a binding:

```csharp
var style = new State<ButtonStyle>(ButtonStyle.Default);
new Button("Apply").Style(style);
```

### Brushes and gradients

Brushes provide per-cell gradient colors. Available on select controls (`TextBlock`, `TextFiglet`, `TextBox`):

```csharp
var brush = Brush.LinearGradient(
    new GradientPoint(0f, 0f),    // left
    new GradientPoint(1f, 0f),    // right
    [
        new GradientStop(0f, Colors.DeepSkyBlue),
        new GradientStop(1f, Colors.White)
    ]);

new TextBlock("Gradient text")
    .Style(TextBlockStyle.Default with { ForegroundBrush = brush });
```

`Brush.Solid(color)` for a uniform per-cell color.

Gradient interpolation uses `Theme.GradientMixSpace` (`ColorMixSpace.Oklab` by default). Override with `mixSpaceOverride` on the brush.

### Color

```csharp
Color.Rgb(0x50, 0x9A, 0xF6)           // opaque RGB
Color.RgbA(0x50, 0x9A, 0xF6, 0x88)   // RGBA (alpha blended during rendering)
Color.Basic16(7)                        // terminal indexed color
Color.Default                           // terminal default
```

**Alpha blending:** Even though terminals render one color per cell, `Color.RgbA` values are blended during rendering — use low alpha (0x10–0x40) for hover/focus overlays.

### Color schemes

`ColorScheme` represents a 16-color palette. Built-in curated schemes (via Root Loops generator):

```csharp
// List all predefined schemes
var schemes = ColorScheme.GetPredefinedSchemes();

// Use a specific scheme
Theme.FromScheme(ColorScheme.CherryDark)
Theme.FromScheme(ColorScheme.OceanLight)

// Generate a custom scheme
var custom = ColorScheme.Generate(seed: 42, dark: true);
```

`Theme.Default` uses `ColorScheme.RootLoopsDark`. `Theme.DefaultLight` uses `ColorScheme.RootLoopsLight`.

### Markup syntax

XenoAtom uses a lightweight markup for styling terminal text via `XenoAtom.Ansi.AnsiMarkup`. Used in:
- `Terminal.WriteMarkup(...)` for terminal output
- The `Markup` control for inline styled text
- `MarkupTextParser` for custom control rendering

**Basic examples:**

```csharp
new Markup("[bold]Hello[/] [gray]world[/]!");

new Markup("[black on yellow]Warning[/] disk is almost full");

new Markup("[bold]Title: [red]Error[/][/] details");

new Markup("[bold yellow on blue]Highlighted[/]");
```

**Tag syntax:**
- Tags: `[token token ...]...[/]`
- Multiple tokens in one tag (space-separated): `[bold yellow on blue]`
- Close the most recent tag: `[/]`
- Escape literal brackets: `[[` → `[`, `]]` → `]`

**Style tokens:**

| Category | Tokens |
|----------|--------|
| Weight/decoration | `bold`, `italic`, `underline`, `strikethrough`, `blink`, `dim`, `overline` |
| Named foreground | `red`, `green`, `blue`, `yellow`, `cyan`, `magenta`, `white`, `gray`, `black` (and `bright-*` variants) |
| RGB foreground | `#RRGGBB` |
| Background | `on <color>` (e.g. `on blue`, `on #FF0000`) |
| Reset | `default`, `reset` |

**`Markup` control:**

```csharp
var markup = new Markup(
    "[bold green]Success![/] Built [cyan]42[/] files in [yellow]1.23s[/]"
);
```

**Terminal.WriteMarkup:**

```csharp
Terminal.WriteMarkup("[bold]Starting deployment...[/]\n");
Terminal.WriteMarkup($"  Status: [green]{status}[/]\n");
```

### ANSI/VT primitives (XenoAtom.Ansi)

For low-level terminal output without controls:

```csharp
using XenoAtom.Ansi;

// Style emission (SGR)
Terminal.Write(AnsiStyle.Bold);
Terminal.Write(AnsiStyle.ForegroundColor(Color.Rgb(0x50, 0x9A, 0xF6)));
Terminal.Write(AnsiStyle.Reset);

// Move cursor
Terminal.Write(AnsiSequences.CursorUp(3));
Terminal.Write(AnsiSequences.CursorColumn(1));
```

### Glyphs and Rune

Controls use `Rune` for box-drawing characters and glyphs, enabling re-theming:

```csharp
// Override border glyphs in a style
new Border("Custom").Style(BorderStyle.Rounded);

// Custom glyph in a control style
new TabControl(tabs)
    .Style(TabControlStyle.Default with
    {
        CloseButtonRune = new Rune('×'),
    });
```

**Nerd Font icons** — use the generated `NerdFont` helpers for official Nerd Fonts glyphs:

```csharp
using XenoAtom.Terminal.UI;

var icon = NerdFont.FolderOpen;  // returns Rune for 󰝰
new TextBlock(icon.ToString());
```

### Common `*Style` types

| Control | Style type | Key options |
|---------|-----------|-------------|
| `Button` | `ButtonStyle` | `Tone`, `ShowBorder`, `Padding` |
| `Border` | `BorderStyle` | `BorderStyle.Rounded`, custom glyphs/colors |
| `Group` | `GroupStyle` | `GroupStyle.Rounded` |
| `TextBox`/`TextArea` | `TextBoxStyle` | `Padding`, `ForegroundBrush`, `BackgroundBrush` |
| `TextBlock` | `TextBlockStyle` | `ForegroundBrush`, `BackgroundBrush`, `Trimming` |
| `ScrollViewer` | `ScrollViewerStyle` | scrollbar thickness/colors |
| `TabControl` | `TabControlStyle` | `Compact`, `Legacy` presets; glyph overrides |
| `TreeView` | `TreeViewStyle` | indentation, glyphs, selection colors |

### ControlTone

`ControlTone` applies semantic coloring from the current theme:

```csharp
ControlTone.Default   // normal
ControlTone.Primary   // accent
ControlTone.Success   // green-ish
ControlTone.Warning   // yellow-ish
ControlTone.Error     // red-ish
```

```csharp
new Button("Delete").Tone(ControlTone.Error);
new Button("Save").Tone(ControlTone.Primary);
```
