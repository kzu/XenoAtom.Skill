# Layout & Binding

Visual tree, layout protocol, State<T> binding, ComputedVisual, data templating, and culture-aware formatting in XenoAtom.Terminal.UI.

### Visual Tree

The fundamental building block is `Visual`:
- participates in layout (`Measure`/`Arrange`)
- participates in rendering
- receives routed input events

**One visual, one parent rule:** A `Visual` instance can only live in one place in the tree. Reusing the same instance in two containers throws `InvalidOperationException`. Create two visuals and bind them to shared state instead.

### State<T> — observable container

`State<T>` is the primary observable container for UI-driven state:

```csharp
var name = new State<string>("Alex");

var ui = new VStack(
    new TextBlock(() => $"Hello {name.Value}"),
    new TextBox().Text(name)
);
```

- `State<T>` is a class with a single `[Bindable] partial T Value { get; set; }`.
- Use `State<T>` for small, UI-only state. Use a custom bindable model for grouped values.

### Custom bindable models

```csharp
public sealed partial class AppModel
{
    [Bindable] public partial string Status { get; set; } = "Ready";
    [Bindable] public partial bool IsBusy { get; set; }
    [Bindable] public partial double Progress { get; set; }
}
```

Requires the type to be `partial`. The source generator produces the binding plumbing automatically.

### Tracking contexts

Bindable values are tracked when read during:
- dynamic update factories (`ComputedVisual`, `ContentSwitcher`)
- `PrepareChildren`
- layout (`Measure` / `Arrange`)
- `Render`

When a tracked value changes, only the affected visuals are invalidated — no manual `RequestRender` needed.

**Always read bindable state through properties, not private backing fields**, to ensure tracking works.

### Binding connections

Connect bindings using fluent methods or `.Bind`:

```csharp
var query = new State<string>("");

var box = new TextBox().Text(query);
var mirror = new TextBlock(query.Bind.Value);   // typed Binding<string>
var hidden = new VStack().IsVisible(query.Bind.Value.Select(v => v.Length > 0));
```

Lambda factories create computed bindings:

```csharp
new TextBlock(() => $"Items: {items.Value.Count}")
```

### ComputedVisual — dynamic subtrees

`ComputedVisual` rebuilds a subtree whenever its tracked dependencies change:

```csharp
var isOpen = new State<bool>(false);

var overlay = new ComputedVisual(() =>
{
    if (!isOpen.Value) return null;

    return new Backdrop(
        new Group
        {
            Content = new VStack(
                new TextBlock("Dialog"),
                new Button("Close").OnClick(() => isOpen.Value = false)
            ).Spacing(1).Pad(new Thickness(1))
        });
});

var root = new VStack(
    new Button("Open").OnClick(() => isOpen.Value = true),
    overlay
);
```

**Important:** `ComputedVisual` manages attaching/detaching. Never reuse a visual across different `ComputedVisual` instances.

### Suppressing tracking (advanced)

When you need to read/write state inside a tracking context without creating a dependency:

```csharp
using (BindingManager.Current.SuppressReadTracking())
{
    // reads here do not create dependencies
    var snapshot = telemetry.Value;
}

BindingManager.Current.RunAfterTracking(() =>
{
    // runs after the current tracking scope exits
    sideEffect.Value = true;
});
```

### ContentSwitcher — pre-built visual toggle

`ContentSwitcher` switches between a fixed set of pre-built visuals without rebuilding them:

```csharp
var page = new State<int>(0);

var switcher = new ContentSwitcher(page.Bind.Value)
{
    [0] = new TextBlock("Page 0"),
    [1] = new TextBlock("Page 1"),
    [2] = new TextBlock("Page 2"),
};
```

Use `ContentSwitcher` when the set of possible visuals is known up-front and you want to avoid teardown/rebuild overhead.

### Data Templating

`DataPresenter<T>` renders a collection using a template factory:

```csharp
var items = new State<IReadOnlyList<MyItem>>(Array.Empty<MyItem>());

var list = new DataPresenter<MyItem>(items.Bind.Value,
    item => new HStack(
        new TextBlock(item.Name),
        new TextBlock(() => item.Status.ToString())
    ).Spacing(2));
```

For observable collections, bind to an `ObservableCollection<T>` or any `INotifyCollectionChanged` source.

### Layout protocol

Terminal UI uses a two-pass cell-based layout:

1. **Measure** — compute `SizeHints` (min/natural/max + flex factors) under `LayoutConstraints`
2. **Arrange** — receive a finite `Rectangle` and position children inside it

```csharp
var constraints = LayoutConstraints.FromMaxSize(new Size(80, 25));
var constraints2 = LayoutConstraints.Unbounded;
```

**`SizeHints` helpers:**

```csharp
SizeHints.Fixed(new Size(w, h));
SizeHints.Flex(min, natural, max, growX: 1, growY: 0, shrinkX: 1, shrinkY: 0);
SizeHints.FlexX(min, natural);   // flex horizontally, fixed vertically
SizeHints.FlexY(min, natural);   // fixed horizontally, flex vertically
```

### What the framework handles automatically

- **Margin** — deflated from constraints before measure, inflated after; child `DesiredSize` already includes margin
- **`IsVisible = false`** — measures as `(0,0)`, excluded from layout, rendering, and hit testing
- **Alignment** — applied by the child when arranged into its slot; containers just provide the slot

### Custom control: padded single-child example

```csharp
public sealed partial class PaddedContent : ContentVisual
{
    [Bindable] public partial Thickness Padding { get; set; } = new(1);

    protected override SizeHints MeasureCore(in LayoutConstraints constraints)
    {
        var pad = Padding;
        var padH = pad.Horizontal;
        var padV = pad.Vertical;
        var maxW = constraints.IsWidthBounded ? Math.Max(0, constraints.MaxWidth - padH) : int.MaxValue;
        var maxH = constraints.IsHeightBounded ? Math.Max(0, constraints.MaxHeight - padV) : int.MaxValue;
        var inner = new LayoutConstraints(0, maxW, 0, maxH);

        var hints = Content?.MeasureHints ?? SizeHints.Fixed(Size.Zero);
        return hints.Inflate(padH, padV);
    }

    protected override void ArrangeCore(Rectangle rect)
    {
        var pad = Padding;
        Content?.Arrange(rect.Deflate(pad));
    }
}
```

**Layout rules:**
- Measure must be **pure** (no persistent state mutation)
- Always clamp subtractions: `Math.Max(0, max - padding)` to avoid negative values
- Reads of bindables during measure/arrange are tracked — layout re-runs automatically when they change

### Alignment defaults by control type

- Containers (`ScrollViewer`, `DataGridControl`): default `Align.Stretch`
- Leaf controls (`Button`, `TextBlock`): default `Align.Start` (size to content)

### Culture-aware formatting

XenoAtom.Terminal.UI resolves culture from the environment for value formatting. Controls like `NumberBox` use the ambient `CultureInfo`. Override per-subtree:

```csharp
new VStack(new NumberBox())
    .Culture(CultureInfo.GetCultureInfo("fr-FR"));
```

### Visibility vs enabled state

| Property | Layout | Render | Input |
|----------|--------|--------|-------|
| `IsVisible = false` | no (collapses) | no | no |
| `IsEnabled = false` | yes | yes (disabled style) | no |
| `IsHitTestVisible = false` | yes | yes | no (pointer only) |
