---
name: xenoatom-terminal-ui
description: Build .NET terminal UI apps with XenoAtom.Terminal.UI — a retained-mode TUI framework with fullscreen and inline hosting, data binding, controls, layout, styling, input handling, and text editing. Use for any XenoAtom.Terminal.UI question.
---

## When to use

Use this skill when the user is building .NET terminal applications using **XenoAtom.Terminal.UI**, including:
- Getting started with the library (installation, first app)
- Choosing between inline (`Terminal.Write`/`Terminal.Live`) and fullscreen (`Terminal.Run`) hosting modes
- Async patterns in terminal UI applications
- Prompt-style interactive inputs
- Understanding the ecosystem (XenoAtom.Terminal, XenoAtom.Ansi, XenoAtom.Logging, XenoAtom.CommandLine)
- Any general TUI question that doesn't fit a more specialized skill

## Instructions

### Prerequisites

- .NET 10 SDK (C# 14) — required
- NuGet package: `XenoAtom.Terminal.UI`

```bash
dotnet add package XenoAtom.Terminal.UI
```

Enable the source generator by marking types as `partial` (required for `[Bindable]` and `[RoutedEvent]`).

### Ecosystem

| Library | Role |
|---------|------|
| `XenoAtom.Terminal.UI` | UI widgets, layout, rendering, app model |
| `XenoAtom.Terminal` | Terminal I/O, hosting, input events |
| `XenoAtom.Ansi` | ANSI/VT primitives, markup, SGR |
| `XenoAtom.Logging` | Structured logging + `LogControl` sink |
| `XenoAtom.CommandLine` | CLI parser; `.Terminal` sub-package renders help via Terminal.UI |

Dependency chain: `Terminal.UI → Terminal → Ansi`

### Hosting modes

XenoAtom.Terminal.UI provides three hosting modes via C# 14 extension members on `Terminal`:

#### 1. `Terminal.Write` — one-shot rendering

Measures, arranges, renders, and writes the visual once to terminal output. Best for tables, summaries, and rich markup blocks.

```csharp
using XenoAtom.Terminal;
using XenoAtom.Terminal.UI;
using XenoAtom.Terminal.UI.Controls;

Terminal.Write(new VStack(
    new TextBlock("Build complete"),
    new ProgressBar { Value = 1.0 }
).Padding(1));
```

#### 2. `Terminal.Live` — inline live region

Runs a UI loop inline (below normal output). The region updates on each tick. Returns `TerminalLoopResult.Continue`, `Stop`, or `StopAndKeepVisual`.

```csharp
using XenoAtom.Terminal;
using XenoAtom.Terminal.UI;
using XenoAtom.Terminal.UI.Controls;

var progress = new State<double>(0);

Terminal.Live(
    new ProgressBar().Value(progress),
    onUpdate: () =>
    {
        progress.Value += 0.05;
        return progress.Value >= 1.0
            ? TerminalLoopResult.StopAndKeepVisual
            : TerminalLoopResult.Continue;
    });
```

Mouse input is disabled by default in `Terminal.Live`. To enable it:

```csharp
Terminal.Live(
    visual,
    onUpdate: () => TerminalLoopResult.Continue,
    options: new TerminalLiveOptions { EnableMouse = true });
```

#### 3. `Terminal.Run` — fullscreen application

Owns the entire viewport (alternate screen). Input is routed to focused controls. Exit with `Ctrl+Q` by default (configurable via `TerminalRunOptions.ExitGesture`).

```csharp
using XenoAtom.Terminal;
using XenoAtom.Terminal.UI;
using XenoAtom.Terminal.UI.Controls;

Terminal.Run(
    new VStack(
        new TextBlock("Press Ctrl+Q to exit"),
        new Button("Hello").Click(() => { /* ... */ })
    ).Padding(1),
    onUpdate: () => TerminalLoopResult.Continue);
```

#### Loop pacing

Default `LoopMode = Auto`: event/deadline-driven at ~66 Hz when active, idle when nothing changes. Use `LoopMode = Polling` with `UpdateWaitDuration` only for legacy periodic behavior.

### Async hosting

Use `Terminal.LiveAsync` / `Terminal.RunAsync` for async update callbacks. The UI stays single-threaded; the callback is a cooperative coroutine.

```csharp
var progress = new State<double>(0);

await Terminal.LiveAsync(
    new ProgressBar().Value(progress),
    async _ =>
    {
        await Task.Delay(50);
        progress.Value = Math.Min(1.0, progress.Value + 0.05);
        return progress.Value >= 1.0
            ? TerminalLoopResult.StopAndKeepVisual
            : TerminalLoopResult.Continue;
    });
```

**Async rules:**
- Do NOT use `ConfigureAwait(false)` inside `onUpdate` if you need to write UI state after the await — the continuation may run on a thread-pool thread.
- Routed event handlers (`Click(...)`, `OnKeyDown(...)`) are **synchronous**. Do not use `async void`.
- For background work, use `Dispatcher.InvokeAsync(...)` to post back to the UI thread:

```csharp
_ = Task.Run(async () =>
{
    var data = await FetchAsync().ConfigureAwait(false);
    await Dispatcher.Current.InvokeAsync(() => result.Value = data);
});
```

**Trigger async work from synchronous events (recommended pattern):**

```csharp
var shouldLoad = new State<bool>(false);
var text = new State<string>("Idle");

var button = new Button("Load").Click(() => shouldLoad.Value = true);

await Terminal.RunAsync(
    new VStack(button, new TextBlock().Text(() => text.Value)),
    async _ =>
    {
        if (!shouldLoad.Value) return TerminalLoopResult.Continue;
        shouldLoad.Value = false;
        text.Value = "Loading...";
        text.Value = await LoadDataAsync();
        return TerminalLoopResult.Continue;
    });
```

### Prompts

XenoAtom.Terminal.UI includes `Prompt` helpers built on top of `Terminal.Live` for interactive inline input.

```csharp
using XenoAtom.Terminal.UI;

var name = await Terminal.Prompt("Enter your name: ");
var confirmed = await Terminal.Confirm("Are you sure?");
var selection = await Terminal.Select("Choose:", new[] { "Option A", "Option B", "Option C" });
```

Prompts block until the user responds and then return the value.

### Fluent API patterns

All controls use fluent extension methods for configuration:

```csharp
new Button("Submit")
    .MinWidth(10)
    .IsEnabled(false)
    .Margin(new Thickness(0, 1, 0, 0))
    .AutoFocus(true)
    .Click(() => Console.WriteLine("Submitted"));
```

Common `Visual` fluent methods:
- `.HorizontalAlignment(Align.Stretch)` / `.VerticalAlignment(Align.Stretch)` / `.Stretch()`
- `.MinWidth(n)`, `.MaxWidth(n)`, `.MinHeight(n)`, `.MaxHeight(n)`
- `.Margin(Thickness)`, `.Padding(n)` (on controls that have padding)
- `.IsVisible(bool)`, `.IsEnabled(bool)`, `.AutoFocus(bool)`

### Reference files

For deeper feature areas, load the relevant reference on demand:
- [Controls reference](references/controls.md) — Button, TextBox, DataGrid, TabControl, and all built-in controls
- [Input & Commands](references/input.md) — keyboard/mouse events, focus, routed events, commands
- [Layout & Binding](references/layout.md) — visual tree, State<T>, ComputedVisual, data templating
- [Styling & Markup](references/styling.md) — themes, styles, brushes, gradients, markup syntax, ANSI
- [Text Editing](references/text-editing.md) — TextBox/TextArea/CodeEditor/PromptEditor, undo/redo, find/replace
