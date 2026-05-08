---
name: XenoAtom.Terminal.UI — Text Editing
description: TextBox, TextArea, CodeEditor, PromptEditor, MaskedInput, NumberBox, and the shared text editing subsystem (selection, undo/redo, find/replace, scrolling) in XenoAtom.Terminal.UI.
---

## When to use

Use this skill when the user needs help with:
- Single-line text input (`TextBox`, `MaskedInput`, `NumberBox`)
- Multi-line text editing (`TextArea`, `CodeEditor`)
- Prompt-style editor with command hints (`PromptEditor`)
- Text selection, copy/cut/paste, clipboard paste interception
- Undo/redo in text editors
- Find/Replace and Go To Line popups
- Scroll integration for text editors
- Implementing syntax highlighting in `CodeEditor`

## Instructions

### Controls sharing the text editing engine

| Control | Description |
|---------|-------------|
| `TextBox` | Single-line editor; password mode, overflow indicators |
| `TextArea` | Multi-line editor; soft wrapping, Find/Replace (`Ctrl+F`/`Ctrl+H`) |
| `CodeEditor` | Multi-line code editor; line numbers, pluggable margins, syntax highlighting, Go To Line (`Ctrl+G`) |
| `MaskedInput` | Structured template editor (credit cards, dates, IDs) |
| `NumberBox` | Numeric value with inline validation and binding |
| `PromptEditor` | Single/multi-line prompt with configurable accept/new-line command hints |

These secondary controls also use the same engine:
- `SearchReplacePopup` — reusable Find/Replace UI (hosted by TextArea, CodeEditor, LogControl)
- `LogControl` — selection/copy + Find (`Ctrl+F`)
- `DataGridControl` — inline cell editing

### TextBox

Single-line text input:

```csharp
using XenoAtom.Terminal.UI.Controls;

// Simple text box
var textBox = new TextBox("Initial text");

// Bound to State<string>
var name = new State<string>("");
var nameBox = new TextBox().Text(name);

// Password mode
var password = new TextBox().IsPasswordMode(true);

// With placeholder
var search = new TextBox().Placeholder("Search...");

// With constraints
var narrow = new TextBox()
    .MinWidth(20)
    .MaxWidth(40);
```

### TextArea

Multi-line editor:

```csharp
var area = new TextArea("Line 1\nLine 2\nLine 3");

// Inside a ScrollViewer for scrollable content
new ScrollViewer(new TextArea(longText))
    .MinHeight(8)
    .MaxHeight(20);

// Bound text
var content = new State<string>("");
new TextArea().Text(content);
```

Built-in keyboard shortcuts:
- `Ctrl+F` — Find
- `Ctrl+H` — Find/Replace
- `Ctrl+Z` / `Ctrl+R` — Undo / Redo

### CodeEditor

Code-oriented editor with line numbers and pluggable margins:

```csharp
using XenoAtom.Terminal.UI.Controls;

var editor = new CodeEditor(new CodeEditorConfig
{
    ShowLineNumbers = true,
    IndentWidth = 4,
    UseTabsForIndent = false,
})
{
    Text = "// Hello world\npublic class Foo { }",
};

new ScrollViewer(editor);
```

Built-in keyboard shortcuts:
- `Ctrl+F` — Find
- `Ctrl+H` — Find/Replace
- `Ctrl+G` — Go To Line
- `Ctrl+Z` / `Ctrl+R` — Undo / Redo
- `Tab` — indent (4 spaces by default)

**Editor location APIs:**
- `editor.Line` — bindable current line (1-based)
- `editor.Column` — bindable current column (1-based)
- Go To Line / Column / Position helpers (accessible via `Ctrl+G`)

### MaskedInput

Structured input with a visible template:

```csharp
// Credit card
new MaskedInput("9999-9999-9999-9999;_");

// Date MM/DD/YYYY
new MaskedInput("99/99/9999");

// Hex color
new MaskedInput("#HHHHHH");
```

**Template tokens:**

| Token | Allowed | Required |
|-------|---------|----------|
| `9` | `[0-9]` | Yes |
| `0` | `[0-9]` | No |
| `A` | `[A-Za-z]` | Yes |
| `a` | `[A-Za-z]` | No |
| `N` | `[A-Za-z0-9]` | Yes |
| `n` | `[A-Za-z0-9]` | No |
| `H` | `[A-Fa-f0-9]` | Yes |
| `h` | `[A-Fa-f0-9]` | No |
| `X` | any non-space | Yes |
| `x` | any non-space | No |

Non-token characters are literal separators. Escape token characters with `\`. Append `;c` to specify placeholder: `"9999;_"` shows `____`.

**Reading values:**

```csharp
var card = new MaskedInput("9999-9999-9999-9999;_");

new VStack(
    card,
    new TextBlock(() => $"Value: {card.Value}"),          // slot chars only
    new TextBlock(() => $"Compact: {card.CompactValue}"), // empty slots removed
    new TextBlock(() => $"Valid: {card.IsValid}")         // all required slots filled
).Spacing(1);
```

**Case directives in template:**
- `>` — convert subsequent input to uppercase
- `<` — convert subsequent input to lowercase

### NumberBox

Numeric input with binding:

```csharp
var count = new State<int>(0);

new NumberBox()
    .Value(count)
    .Min(0)
    .Max(100)
    .Step(1);
```

### PromptEditor

Prompt-style editor for REPL-like UX:

```csharp
using XenoAtom.Terminal.UI.Controls;

var prompt = new PromptEditor()
    .Placeholder("Enter command...")
    .OnSubmit(text => ExecuteCommand(text));

// Multi-line mode (Shift+Enter for new line, Enter to submit)
var multiPrompt = new PromptEditor { IsMultiLine = true };
```

### Clipboard paste interception

Intercept paste to handle non-text payloads (images, custom formats):

```csharp
using XenoAtom.Terminal.UI.Controls;

new PromptEditor()
    .ClipboardPasteHandler(context =>
    {
        if (context.TryGetData(TerminalClipboardFormats.Png, out var png))
        {
            SaveImage(png.Span);
            return "![pasted image](paste.png)";  // replacement text
        }

        return null; // fall back to normal text paste
        // return string.Empty; // suppress paste entirely
    });
```

The handler receives `TextEditorClipboardPasteContext` containing clipboard text, advertised formats, and raw data.

### Undo / Redo

All text editors support undo/redo:

- `Ctrl+Z` — undo
- `Ctrl+R` — redo

Undo/redo also integrates with programmatic edits — use `ITextDocument` edit APIs to keep undo history consistent.

### Find / Replace

`TextArea` and `CodeEditor` host a `SearchReplacePopup`:

```
Ctrl+F  — open Find
Ctrl+H  — open Find/Replace
```

The popup is anchored to the editor and uses `ISearchReplaceTarget`. The same popup can be used standalone via `SearchReplacePopup` in other controls (e.g., `LogControl`).

### Go To Line (CodeEditor)

```
Ctrl+G  — open Go To Line popup
Enter   — navigate to the one-based line number entered
Escape  — close and restore the caret
```

### Scroll integration

Text editors implement `IScrollable` so they work naturally with `ScrollViewer`:

```csharp
// TextArea / CodeEditor with scrollbars
new ScrollViewer(new TextArea(longContent))
    .MinHeight(10)
    .MaxHeight(30);
```

`ScrollViewer` delegates scroll model management to the editor's own `ScrollModel`, enabling consistent scrollbar behavior and keyboard/wheel navigation.

### Architecture (for custom integrations)

| Type | Role |
|------|------|
| `TextEditorBase` | Shared control base (focus, commands, cursor) |
| `TextEditorCore` | Editing behavior (nav, selection, clipboard, undo/redo, search) |
| `ITextDocument` | Document abstraction (storage + edits) |
| `TextDocument` | Simple in-memory document |
| `DynamicTextDocument` | Bridges a bindable `Text` property to the engine |
| `ScrollModel` | Viewport/extent model used by `IScrollable` |

For large documents, the multi-line editor caches per-line widths and wrap metadata, renders only visible rows, and uses sparse checkpoints for long wrapped lines. The caret uses the terminal cursor (not reverse-video), compatible with accessibility settings.

### Common style types

| Control | Style type | Notable properties |
|---------|-----------|---------------------|
| `TextBox` | `TextBoxStyle` | `Padding`, `ForegroundBrush`, `BackgroundBrush` |
| `TextArea` | `TextBoxStyle` (extended) | same as TextBox |
| `CodeEditor` | `CodeEditorStyle` | line number colors, current-line highlight |
| `MaskedInput` | `MaskedInputStyle` | placeholder/separator foregrounds, placeholder glyphs |
| `NumberBox` | `NumberBoxStyle` | inherits `TextBoxStyle` |
