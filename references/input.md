# Input & Commands

Keyboard/mouse input, focus, routed events, commands, key hints (CommandBar), and CLI integration for XenoAtom.Terminal.UI.

### Input pipeline

`TerminalApp` converts raw terminal events into routed events on the visual tree:

**Keyboard flow:**
1. `TerminalApp` receives `TerminalKeyEvent`
2. Command shortcuts are checked first (focus-to-root traversal)
3. If not handled, `Visual.KeyDownEvent` is raised on the focused element
4. Text input: `Visual.TextInputEvent` with `TextInputEventArgs`
5. Paste: `Visual.PasteEvent` with `PasteEventArgs`

**Mouse flow:**
1. Hit test from the input root to find target visual (ignoring `IsHitTestVisible=false` visuals)
2. If target is not visible/enabled, walk up to nearest visible+enabled ancestor
3. Pointer is captured automatically on left mouse down (released on mouse up)
4. Raises `PointerPressedEvent`, `PointerMovedEvent`, `PointerReleasedEvent`, `PointerWheelEvent`

The input root changes when a modal overlay (dialog/popup) is active.

### Routed events

Events propagate through the visual tree with three routing strategies:

| Strategy | Direction |
|----------|-----------|
| `Direct` | Only the target |
| `Preview` | Root → target (parents first) |
| `Bubble` | Target → root (parents last) |

Most pointer events use `Preview | Bubble`. Keyboard events typically use `Direct` or `Bubble`.

**Event args properties:**
- `Handled` — marks event as consumed; skips subsequent regular handlers
- `OriginalSource` — where the event started
- `Source` — current visual during routing
- `RoutingPhase` — `Direct`, `Preview`, or `Bubble`

### Handling input in custom controls

Override the virtual methods on `Visual`:

```csharp
public sealed partial class MyControl : ContentVisual
{
    protected override void OnKeyDown(KeyEventArgs e)
    {
        if (e.Key == TerminalKey.Enter)
        {
            DoAction();
            e.Handled = true;
        }
    }

    protected override void OnPointerPressed(PointerEventArgs e)
    {
        if (e.RoutingPhase != RoutingPhase.Bubble) return;
        if (e.Button == TerminalMouseButton.Left)
        {
            Focus();
            e.Handled = true;
        }
    }

    protected override void OnPointerMoved(PointerEventArgs e) { }
    protected override void OnPointerReleased(PointerEventArgs e) { }
    protected override void OnPointerWheel(PointerEventArgs e) { }
    protected override void OnTextInput(TextInputEventArgs e) { }
    protected override void OnPaste(PasteEventArgs e) { }
}
```

To observe an event even after descendants mark it handled:

```csharp
AddHandler(Visual.PointerWheelEvent, OnWheelObserved, handledEventsToo: true);
```

### Pointer coordinates

`PointerEventArgs` provides:
- `X` / `Y` — terminal coordinates
- `UiX` / `UiY` — UI root coordinates (important in inline/live hosting where UI is offset)
- `LocalX` / `LocalY` — coordinates relative to the event target's `Bounds`

### Creating custom routed events

Mark a dispatch method with `[RoutedEvent]` on a `partial` type:

```csharp
public sealed partial class FancyButton : ContentVisual
{
    [RoutedEvent(RoutingStrategy.Bubble)]
    protected virtual void OnClick(ClickEventArgs e) { }

    protected override void OnPointerReleased(PointerEventArgs e)
    {
        if (e.Button == TerminalMouseButton.Left)
        {
            RaiseEvent(ClickEvent, new ClickEventArgs());
            e.Handled = true;
        }
    }
}
```

The source generator produces:
- `FancyButton.ClickEvent` static `RoutedEvent` identifier
- `FancyButton.ClickRouted` instance event (add/remove via `AddHandler`/`RemoveHandler`)
- `button.Click(handler)` fluent registration helper

### Focus

- Controls opt in with `Focusable = true`
- Tab navigation moves focus between focusable visuals
- Mouse down focuses the nearest focusable ancestor of the hit target
- `Popup` and `Dialog` save and restore focus automatically

Check focus state in rendering/input:

```csharp
if (HasFocus)
{
    // render focus cue
}
if (HasFocusWithin)
{
    // any descendant has focus
}
```

### Disabled visuals and routing

Disabled visuals (`IsEnabled = false`) do not receive input, but routing **continues through** them so enabled ancestors can still react. This allows `ScrollViewer` to scroll when the pointer is over a disabled child.

### Context menus (fullscreen only)

Right-click without `Handled = true` triggers context menu lookup:
1. Nearest `ContextMenuFactory` in the hovered chain
2. Fallback: commands with `CommandPresentation.ContextMenu`

### Commands

Commands make keyboard shortcuts discoverable across `CommandBar`, `CommandPalette`, and menus.

#### Command structure

```csharp
new Command
{
    Id = "App.Save",                              // stable identifier
    LabelMarkup = "[primary]Save[/]",             // ANSI markup supported
    Name = "save",                                // for typed command surfaces
    Gesture = new KeyGesture(TerminalChar.CtrlS, TerminalModifiers.Ctrl),
    Execute = _ => SaveDocument(),
    CanExecute = _ => HasUnsavedChanges(),
    IsVisible = _ => true,
}
```

#### Register commands on a control

```csharp
using XenoAtom.Terminal.UI.Input;

var editor = new TextArea("Hello");
editor.AddCommand(new Command
{
    Id = "Editor.Save",
    LabelMarkup = "[primary]Save[/]",
    Gesture = new KeyGesture(TerminalChar.CtrlS, TerminalModifiers.Ctrl),
    Execute = _ => SaveDocument(),
});
```

**Notes:**
- For Ctrl shortcuts, use `TerminalChar.CtrlX + TerminalModifiers.Ctrl`
- `Gesture` and `Sequence` are mutually exclusive
- `ConsumesGestureWhenUnavailable = false` — only reserve key while command is active

#### Multi-stroke key sequences

```csharp
new Command
{
    Id = "App.CommandPalette",
    LabelMarkup = "Command palette",
    Sequence = new KeySequence(
        new KeyGesture(TerminalChar.CtrlK, TerminalModifiers.Ctrl),
        new KeyGesture(TerminalChar.CtrlP, TerminalModifiers.Ctrl)),
    Execute = _ => OpenPalette(),
}
```

#### Global commands (TerminalApp)

The default quit command is `TerminalApp.DefaultQuitCommandId` (gesture: `Ctrl+Q`). Replace it to change the exit shortcut.

#### CommandBar — visible key hints

```csharp
using XenoAtom.Terminal.UI.Controls;

var root = new DockLayout()
    .Content(new TextArea())
    .Bottom(new VStack(
        new CommandBar(),
        new Footer().Left("Tab focus | Mouse").Right("Ctrl+Q quit")
    ).Spacing(0));
```

`CommandBar` shows commands for the current focus context, clipping when the line is full.

#### CommandPresentation flags

```csharp
CommandPresentation.CommandBar      // shown in CommandBar
CommandPresentation.CommandPalette  // shown in CommandPalette
CommandPresentation.Menu            // shown in MenuBar
CommandPresentation.ContextMenu     // shown in context menu
```

### Selection ownership (copy/paste UX)

`TerminalApp` tracks a single active selection owner:
- Starting a selection in a different control clears the previous selection
- `Ctrl+C` copies the active selection even if that control isn't focused
- Controls implement `ISelectionOwner` to participate
- Opt out with `IsSelectable = false`

### XenoAtom.CommandLine integration

For CLI apps that embed Terminal.UI help:

```bash
dotnet add package XenoAtom.CommandLine
dotnet add package XenoAtom.CommandLine.Terminal
```

```csharp
using XenoAtom.CommandLine;
using XenoAtom.CommandLine.Terminal;

var app = new CommandApp("my-tool", config: new CommandConfig
{
    OutputFactory = _ => new TerminalVisualCommandOutput(new TerminalVisualOutputOptions
    {
        SectionGroupMinWidth = 70,
        ErrorGroupMinWidth = 70,
    })
})
{
    new CommandUsage(),
    "Options:",
    { "n|name=", "The {NAME}", _ => { } },
    new HelpOption(),
    (ctx, _) => ValueTask.FromResult(0)
};

return await app.RunAsync(args);
```

Embed help as a Visual in a fullscreen app:

```csharp
var helpVisual = app.ToHelpVisual();
Terminal.Write(helpVisual);
```

### Recommended patterns for custom controls

- Store interaction state as `[Bindable]` properties (`IsPressed`, `IsOpen`, `SelectedIndex`) — invalidation is automatic
- Set `Handled = true` when fully consuming a gesture
- Prefer **routed events** for user-observable interactions (clicked, selection changed, activated)
- Prefer **commands** for keyboard shortcuts that should appear in `CommandBar` / `CommandPalette`
