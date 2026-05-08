# Controls Reference

Complete reference for all built-in controls in XenoAtom.Terminal.UI — buttons, text input, lists, layout containers, data display, overlays, navigation chrome, and charts.

### Text Input Controls

#### TextBox — single-line editor

```csharp
var name = new State<string>("");
new TextBox().Text(name);
new TextBox("Initial text");
new TextBox().Placeholder("Enter value...").MinWidth(20);

// Password mode
new TextBox().IsPassword(true).ClipboardMode(TextBoxClipboardMode.Disabled);
```

Key properties: `Text`, `TextAlignment`, `IsPassword`, `PasswordRevealMode`, `ClipboardMode`
Shortcuts: `Ctrl+Z` undo, `Ctrl+R` redo, `Ctrl+C/X/V` clipboard
Default alignment: `Start` × `Start`

#### TextArea — multi-line editor

```csharp
var content = new State<string>("");
new TextArea().Text(content);
new TextArea("Line 1\nLine 2");

// With scroll (typical usage)
new ScrollViewer(new TextArea(longText)).MinHeight(8).MaxHeight(20);
```

Shortcuts: `Ctrl+F` find, `Ctrl+H` replace, `Ctrl+Z` undo, `Ctrl+R` redo
Default alignment: `Start` × `Start` (implements `IScrollable`)

#### CodeEditor — code editor with line numbers

```csharp
var code = new State<string>("// code");
new CodeEditor().Text(code).ShowLineNumbers(true);

// Indentation
new CodeEditor(source).IndentationStyle(CodeEditorIndentationStyle.Spaces).IndentationSize(2);

// Status bar binding
new Footer().Left(new TextBlock(() => $"Ln {editor.Line}, Col {editor.Column}"));

// Programmatic navigation
editor.GoToLine(42);
editor.GoToLine(42, 8);   // line + column (1-based)
editor.GoToPosition(128); // UTF-16 position (0-based)
editor.OpenFind("TODO");
editor.OpenReplace("var");

// Wrap in ScrollViewer
new ScrollViewer(new CodeEditor(source).MinHeight(20));
```

Shortcuts: `Ctrl+F/H` find/replace, `Ctrl+G` go to line, `Ctrl+Z/R` undo/redo, `Tab` indent
Default alignment: `Start` × `Start` (implements `IScrollable`)

#### MaskedInput — structured template editor

```csharp
new MaskedInput("9999-9999-9999-9999;_");  // credit card, _ placeholder
new MaskedInput("99/99/9999");              // date
new MaskedInput("#HHHHHH");                 // hex color

var card = new MaskedInput("9999-9999-9999-9999;_");
card.Value        // slot chars only (positional, space = empty)
card.CompactValue // empty slots removed
card.IsValid      // all required slots filled
```

Token chars: `9`=digit(req), `0`=digit(opt), `A`=alpha(req), `a`=alpha(opt), `N`=alphanum(req), `n`=alphanum(opt), `H`=hex(req), `h`=hex(opt), `X`=any(req), `x`=any(opt), `D`/`d`=`[1-9]`(req/opt)
Case directives: `>` uppercase, `<` lowercase
Default alignment: `Start` × `Start`

#### NumberBox — numeric input

```csharp
var count = new State<int>(0);
new NumberBox().Value(count).Min(0).Max(100).Step(1);
```

Default alignment: `Start` × `Start`

#### ColorPicker

```csharp
var color = new State<Color>(Color.RgbA(0x50, 0x9A, 0xF6, 0x88));
new ColorPicker().AllowAlpha(true).ShowPalette(true).Value(color);

// Custom palette swatches
new ColorPicker().Palette(Color.Rgb(0xF7, 0x5B, 0x72), Color.Rgb(0x67, 0xAF, 0x34));
```

UI includes: RGB sliders, hex input, optional palette from theme or custom.
Default alignment: `Stretch` × `Start`

---

### Buttons & Toggles

#### Button

```csharp
new Button("Click me").Click(() => DoSomething());
new Button("Save").Tone(ControlTone.Primary).Click(handler);
new Button("Delete").Tone(ControlTone.Error).Click(handler);
new Button("Disabled").IsEnabled(false);

// Content can be any Visual
new Button().Content(new HStack("Save", new Spinner()));

// Style
new Button("Danger").Style(ButtonStyle.Default with { Tone = ControlTone.Error });
```

Activation: `Enter`/`Space` when focused; mouse click
Default alignment: `Start` × `Start`

#### CheckBox

```csharp
var accepted = new State<bool>(false);
new CheckBox("Accept terms").IsChecked(accepted);
new CheckBox("Notify").ValueChanged((_, e) => Console.WriteLine($"Checked: {e.NewValue}"));
```

Activation: `Space`/`Enter` when focused; mouse click
Default alignment: `Start` × `Start`

#### RadioButton

```csharp
var choice = new State<int>(0);
new VStack(
    new RadioButton("Option A").GroupValue(0).SelectedValue(choice),
    new RadioButton("Option B").GroupValue(1).SelectedValue(choice),
    new RadioButton("Option C").GroupValue(2).SelectedValue(choice)
).Spacing(1);
```

Default alignment: `Start` × `Start`

#### Switch

```csharp
var enabled = new State<bool>(false);
new Switch().IsChecked(enabled);
new Switch("Dark mode").IsChecked(enabled);
```

Default alignment: `Start` × `Start`

---

### Lists & Selection

#### ListBox

```csharp
var items = new State<IReadOnlyList<string>>(["Alpha", "Beta", "Gamma"]);
var selected = new State<int>(0);

new ListBox<string>()
    .Items(items)
    .SelectedIndex(selected);

// Custom item template
using XenoAtom.Terminal.UI.Templating;
new ListBox<string>()
    .Items(["First", "Second"])
    .ItemTemplate(new DataTemplate<string>(
        Display: static (DataTemplateValue<string> v, in DataTemplateContext _) =>
            new HStack("→", new TextBlock(() => v.GetValue())).Spacing(1),
        Editor: null));
```

Keyboard: `Up`/`Down` select, `PageUp`/`PageDown` jump, `Home`/`End` first/last
Default alignment: `Stretch` × `Stretch` (implements `IScrollable`)

#### Select (dropdown)

```csharp
var select = new Select<string>()
    .Items(["Option A", "Option B", "Option C"]);

// With binding
var choice = new State<int>(0);
select.SelectedIndex(choice);
```

Opens popup list; `Enter` confirms, `Tab`/`Esc` closes.
Default alignment: `Start` × `Start`

#### TreeView

```csharp
var root = new TreeNode("Root") { IsExpanded = true };
root.Children.Add(new TreeNode("Child A"));
root.Children.Add(new TreeNode("Child B"));

var tree = new TreeView().Roots([root]);

// Icons and right-aligned visuals
var node = new TreeNode("Build pipeline")
{
    Icon = TreeNodeIcons.FolderGlyph,
    IconStyle = Style.None.WithForeground(Colors.Goldenrod),
}
.AddRightVisual("!", TreeNodeRightVisualVisibility.Always)
.AddRightVisual(new Button("×").Tone(ControlTone.Error).Click(dismiss),
    TreeNodeRightVisualVisibility.Hover);

// Selection
tree.SelectedNode    // preferred (not SelectedIndex — index shifts on expand/collapse)
tree.TrySelectNode(node);
```

Keyboard: arrows expand/collapse/select; mouse click selects, glyph click toggles.
Default alignment: `Stretch` × `Stretch`

---

### Layout Containers

#### VStack — vertical stack

```csharp
new VStack(
    new TextBlock("Title"),
    new TextBox("Input"),
    new Button("Submit")
).Spacing(1).Padding(2);
```

Default alignment: `Start` × `Start`

#### HStack — horizontal stack

```csharp
new HStack(new Button("Left"), new Button("Right")).Spacing(2);
```

Default alignment: `Start` × `Stretch`

#### WrapStack — wrapping flow

```csharp
new WrapStack(items.Select(i => new Button(i))).Spacing(1);
```

Items wrap to next row when they don't fit. Default alignment: `Start` × `Start`

#### Grid — grid layout

```csharp
using XenoAtom.Terminal.UI.Layout;

new Grid()
    .ColumnDefinitions([GridLength.Auto, GridLength.Star(1), GridLength.Fixed(10)])
    .RowDefinitions([GridLength.Auto, GridLength.Star(1)])
    .Add(new TextBlock("Name:"),  0, 0)
    .Add(new TextBox(),           1, 0)
    .Add(new TextBlock("Value:"), 0, 1)
    .Add(new NumberBox(),         1, 1);
```

Column/row sizing: `Auto`, `Fixed(n)`, `Star(n)` (proportional fill).
Default alignment: `Stretch` × `Stretch`

#### DockLayout — dock to edges

```csharp
new DockLayout()
    .Top(new Header("My App"))
    .Bottom(new CommandBar())
    .Left(new VStack(navItems).MinWidth(20))
    .Content(mainView);
```

Default alignment: `Stretch` × `Stretch`

#### Center

```csharp
new Center(new TextBlock("Centered content"));
```

Default alignment: `Stretch` × `Stretch`

#### Padder

```csharp
new Padder(new TextBlock("Padded"), new Thickness(2));
// or use .Padding() fluent method on any control that supports it
```

#### Border

```csharp
new Border(new TextArea("Content inside a border"));
new Border("Rounded").Style(BorderStyle.Rounded);
new Border(() => new TextBlock(DateTime.Now.ToString("T"))); // dynamic content
```

Consumes space around content. For labeled frames, use `Group`.
Default alignment: `Start` × `Start`

#### Group — labeled border frame

```csharp
new Group("Settings")
    .Content(new VStack(
        new CheckBox("Enabled"),
        new TextBox("Name")
    ));

new Group("Rounded").Style(GroupStyle.Rounded).Content("Content");
```

Has optional corner labels (top-left/top-right/bottom-left/bottom-right).
Default alignment: `Start` × `Start`

#### ScrollViewer

```csharp
new ScrollViewer(new VStack("Line 1", "Line 2", "Line 3"));

// Bounded viewport (most common usage)
new ScrollViewer(new TextArea(longText)).MinHeight(8).MaxHeight(20);
```

Delegates to content's `ScrollModel` when content implements `IScrollable` (TextArea, CodeEditor, ListBox, DataGridControl).
Mouse wheel scrolls; `Shift+wheel` scrolls horizontally.
Default alignment: `Stretch` × `Stretch`

#### Splitter

```csharp
new HStack(
    leftPane,
    new VSplitter(),    // vertical separator, drag to resize horizontally
    rightPane
);

new VStack(
    topPane,
    new HSplitter(),    // horizontal separator, drag to resize vertically
    bottomPane
);
```

---

### Data Display

#### TextBlock

```csharp
new TextBlock("Static text");
new TextBlock(() => $"Count: {count.Value}");  // computed, auto-updates
new TextBlock(query.Bind.Value);               // bound

// Text trimming
new TextBlock("Long text...").Style(TextBlockStyle.Default with
{
    Trimming = TextTrimming.EndEllipsis,
});
```

Default alignment: `Start` × `Start`

#### Markup — styled inline text

```csharp
new Markup("[bold green]Success![/] built [cyan]42[/] files in [yellow]1.23s[/]");
new Markup("[black on yellow]Warning[/] disk is almost full");
new Markup("[bold]Title: [red]Error[/][/] nested tags");
```

Default alignment: `Start` × `Start`

#### Table

```csharp
using XenoAtom.Terminal.UI.Controls;

new Table(
    headers: ["Name", "Status", "Progress"],
    rows: [
        ["Alpha", "Done", "100%"],
        ["Beta",  "Running", "45%"],
    ]);
```

Static table rendering. For interactive/sortable tables use `DataGridControl`.

#### DataGridControl — interactive virtualized table

```csharp
using XenoAtom.Terminal.UI.DataGrid;

public sealed partial class MyRow
{
    [Bindable] public partial int Id { get; set; }
    [Bindable] public partial string Name { get; set; } = string.Empty;
}

var doc = new DataGridListDocument<MyRow>()
    .AddColumn(MyRow.Accessor.Id)
    .AddColumn(MyRow.Accessor.Name);

using var view = new DataGridDocumentView(doc);
var grid = new DataGridControl { View = view, FrozenColumns = 1 };

// Optional: typed UI columns
grid.Columns.Add(new DataGridColumn<MyRow, int>
{
    Key = MyRow.Accessor.Id.Name,
    TypedValueAccessor = MyRow.Accessor.Id,
    Width = GridLength.Auto,
    CellAlignment = TextAlignment.Right,
    Sortable = true,
});

new ScrollViewer(grid);
```

Keyboard: `Ctrl+F` find, `F4` filter row, `F2`/`Enter` edit cell, arrow keys navigate, `Ctrl+A` select all, `Ctrl+C` copy (TSV).
Column sizing: `Auto`, `Fixed`, `Star`. Drag separator to resize; double-click to auto-size.
Sorting: click header to cycle, `Ctrl+click` for multi-sort.
Default alignment: `Stretch` × `Stretch`

#### ProgressBar

```csharp
var progress = new State<double>(0.0);  // 0.0 to 1.0
new ProgressBar().Value(progress);

new ProgressBar().Style(ProgressBarStyle.Bracketed);
new ProgressBar { IsIndeterminate = true }; // animated spinner-style
```

Default alignment: `Stretch` × `Start`

#### ProgressTaskGroup

```csharp
var work = new ProgressTask("Compile");
var download = new ProgressTask("Download") { Maximum = 1024 };

var group = new ProgressTaskGroup().Tasks([work, download]);

// In update loop:
work.Value = 0.5;
download.Value = 512;

// Custom columns
group.Columns([
    ProgressTaskColumns.Spinner(),
    ProgressTaskColumns.Label(),
    ProgressTaskColumns.Bar(),
    ProgressTaskColumns.Percentage(),
]);
```

Default alignment: `Stretch` × `Start`

#### Spinner

```csharp
new Spinner();
new Spinner().Style(SpinnerStyles.Dots);
new Spinner().Style(SpinnerStyles.Arc);
```

Default alignment: `Start` × `Start`

#### Sparkline

```csharp
new Sparkline().Values([1, 4, 2, 5, 3, 6]);
```

Always height 1. Values are downsampled to available width (spikes preserved).
Default alignment: `Start` × `Start`

#### Rule — horizontal separator

```csharp
new Rule();
new Rule("Section Title");
```

#### TextFiglet — large ASCII art text

```csharp
new TextFiglet("Hello");
new TextFiglet("Hi").Style(TextFigletStyle.Default with { ForegroundBrush = brush });
```

---

### Overlays

#### Dialog

```csharp
// Show from an event handler
var result = await Dialog.ShowAsync(
    "Confirm Delete",
    new TextBlock("Are you sure?"),
    buttons: [DialogButton.Yes, DialogButton.No]);

if (result == DialogButton.Yes)
    DeleteItem();
```

Dialogs save and restore focus. In fullscreen hosting, they block input from the rest of the app.

#### Popup

```csharp
var popup = new Popup
{
    Content = new VStack(new Button("Close").Click(() => popup.Close())).Padding(1),
    Placement = PopupPlacement.Below,
    PlacementTarget = button,
};
popup.Open();
```

Closed by clicking outside, pressing `Tab` or `Escape`.

#### Backdrop — modal overlay

```csharp
var backdrop = new Backdrop(
    new Group
    {
        Content = new VStack(
            new TextBlock("Modal content"),
            new Button("Close").Click(() => isOpen.Value = false)
        ).Spacing(1).Pad(new Thickness(1))
    });
```

Use with `ComputedVisual` to show/hide based on state.

#### Tooltip

```csharp
new Button("Hover me").Tooltip("This is a tooltip");
```

#### Toast

```csharp
// Show a transient notification
TerminalApp.Current.ShowToast("File saved", ToastSeverity.Success, TimeSpan.FromSeconds(3));
```

---

### Navigation Chrome

#### TabControl

```csharp
var logs = new TabPage("Logs", "Tail output") { ShowCloseButton = true };
logs.RequestClosing += (_, e) => { if (HasPendingSave()) e.Cancel = true; };

var tabs = new TabControl(
    new TabPage("Status", "Ready"),
    logs,
    new TabPage("Metrics", "42 req/s") { ShowCloseButton = true });

tabs.SelectionChanged((_, e) => { if (e.NewPage != null) LoadForTab(e.NewPage); });

// Programmatic control
tabs.TryCloseTab(logs);
tabs.MoveTab(logs, 0);

// Styles
tabs.Style(TabControlStyle.Compact);   // tight single-line
tabs.Style(TabControlStyle.Legacy);    // flat strip + boxed content
```

Tab overflow shows left/right scroll buttons. `Left`/`Right` arrows switch tabs when focused.
Default alignment: `Stretch` × `Stretch`

#### MenuBar

```csharp
new MenuBar(
    new Menu("File",
        new MenuItem("New",  () => NewFile()),
        new MenuItem("Open", () => OpenFile()),
        new MenuSeparator(),
        new MenuItem("Exit", () => app.RequestExit())),
    new Menu("Edit",
        new MenuItem("Find", () => Find())));
```

#### Header / Footer / StatusBar

```csharp
new Header("My App Title");
new Header().Left("Left").Right("Right");

new Footer().Left("Tab focus | Mouse").Right("Ctrl+Q quit");

new StatusBar().Left(new TextBlock(() => $"Ln {editor.Line}")).Right("UTF-8");
```

#### CommandBar

Shows context-aware key hints for the currently focused control:

```csharp
new DockLayout()
    .Content(new TextArea())
    .Bottom(new VStack(new CommandBar(), new Footer().Left("Ready")).Spacing(0));
```

#### CommandPalette

Provides typed command search (`Ctrl+Shift+P` style). Register globally or per-control via `AddCommand(...)`.
