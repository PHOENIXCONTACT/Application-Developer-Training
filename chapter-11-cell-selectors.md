# Chapter 11 - CellSelectors

One testing station cannot keep up when several orders run in parallel. *Pencilla Inc.* adds a second automatic tester and a slower manual backup for Brown Classic. You distribute work with Capabilities, Constraints and CellSelectors.

> [Table of contents](README.md) | [Previous](chapter-10-module-adapter.md) | [Next](chapter-12-advanced-topics.md)

**On this page:** [Goals](#learning-goals) | [Starting point](#before-you-start) | [Practice](#practice) | [Check your reasoning](#check-your-reasoning) | [Troubleshooting](#troubleshooting)

## Learning goals

By the end of this chapter, you should be able to:

* Distinguish automatic vs manual testing cells with Capabilities and Constraints
* Implement `TestingOptimizer` and `LoadBalancer` CellSelectors and configure SortOrder
* Extend the ResourceInitializer with T-2 / T-3 and verify routing under load

## Where you are in the journey

* Chapters 1-10: full line, seed, ERP intake
* **This chapter**: intelligent testing-cell selection under load
* Chapter 12: assignments, localization, notifications, states

## Before you start

The imported products and Default recipes should be in place and at least one automatic TestingCell should complete Testing orders.

You will add more testing cells and selectors. Keep the existing pencil workplan.

## What you will touch

| Kind | Items |
| ---- | ----- |
| Projects / files | `TestingCapabilities`, `ManualTestingCell`, `PencilProductConstraints`, `CellSelectors/*`, `PencilFactoryInitializer` |
| Classes | `TestingOptimizer`, `LoadBalancer`, `ManualTestingCell`, constraint helpers |
| UI | ProcessEngine ResourceSelectors, Orders MaxRunningOperations, MachineSimulator times, Processes view |

For new files, let Visual Studio add missing usings (`Ctrl + .`).

> **Concept:** Capabilities answer *can this cell do the activity?* Constraints and CellSelectors answer
> *which of the matching cells should run it under load or product rules?* Keep capability matching first;
> selectors only reorder or filter an already valid candidate set.

The selection stages connect as follows:

![How a TestingActivity finds a cell](./chapter-11/testing-cell-selection-flow.png)

More on [Cell selectors](https://github.com/PHOENIXCONTACT/MORYX-Framework/blob/main/docs/articles/abstractions/control-system/cell-selector.md)
in the framework.

## Extend TestingCapabilities

Path: `src/PencilFactory/Capabilities/TestingCapabilities.cs`

So that auto and manual cells are distinguishable, add the flag `ManualTesting`:

```csharp
/// <summary>
/// When required by an activity, only cells that provide manual testing match.
/// </summary>
public bool ManualTesting { get; set; }
```

In `ProvidedBy`, insert before `return true;`:

```csharp
if (ManualTesting && !providedTesting.ManualTesting)
    return false;
```

`TestingActivity` does not require Manual by default. Therefore auto and
manual cells both match the capability check. Differentiation comes only through
Constraints and Selectors.

## Mark TestingCell as automatic

Path: `src/PencilFactory.Resources.Testing/TestingCell.cs`

Action: Change one line in `OnInitializeAsync`, the automatic cell sets `ManualTesting = false`:

```csharp
// Before:
Capabilities = new TestingCapabilities { Value = Value };

// After:
Capabilities = new TestingCapabilities { Value = Value, ManualTesting = false };
```

Rest of the file unchanged.

## Create PencilProductConstraints

Path: `src/PencilFactory/Constraints/PencilProductConstraints.cs`

Action: New file (create folder `Constraints/`). The constraint checks the pencil color on the product of the activity:

```csharp
using Moryx.AbstractionLayer.Activities;
using Moryx.AbstractionLayer.Constraints;
using Moryx.AbstractionLayer.Recipes;
using PencilFactory.Products;

namespace PencilFactory.Constraints;

/// <summary>
/// Process constraints: which graphite pencil color may be dispatched to a cell.
/// </summary>
public static class PencilProductConstraints
{
    public static IConstraint ForPencilColor(PencilColor color) =>
        ExpressionConstraint.Equals<ActivityConstraintContext>(
            c => ((GraphitePencilType)((IProductRecipe)c.Process.Recipe).Product).Color,
            color);
}
```

## Create ManualTestingCell

Path: `src/PencilFactory.Resources.Testing/ManualTestingCell.cs`

Action: New file, modeled after `AssemblingCell` (VisualInstructor, no Driver). The manual cell provides `ManualTesting = true` and attaches the constraint to ReadyToWork:

```csharp
namespace PencilFactory.Resources.Testing;

/// <summary>
/// Manual testing workplace for Brown Classic graphite pencils
/// Uses a process constraint so only Brown Classic orders are dispatched here
/// </summary>
[ResourceRegistration]
public class ManualTestingCell : Cell, IAsyncStateContext
{
    private Session _currentSession;
    private long _currentInstruction;

    [ResourceReference(ResourceRelationType.Extension)]
    public IVisualInstructor VisualInstructor { get; set; }

    [DataMember, EntrySerialize]
    [Description("Configured value for the capabilities")]
    public int Value { get; set; }

    [DataMember, EntrySerialize]
    [Description("Pencil color this manual testing station accepts (process constraint)")]
    public PencilColor AcceptedColor { get; set; } = PencilColor.Brown;

    protected override async Task OnInitializeAsync(CancellationToken cancellationToken)
    {
        await base.OnInitializeAsync(cancellationToken);
        Capabilities = new TestingCapabilities { Value = Value, ManualTesting = true };
    }

    protected override IEnumerable<Session> ProcessEngineAttached()
    {
        yield return CreateProductionReadyToWork(ReadyToWorkType.Push);
    }

    protected override IEnumerable<Session> ProcessEngineDetached()
    {
        yield break;
    }

    public override void StartActivity(ActivityStart activityStart)
    {
        _currentSession = activityStart;
        switch (activityStart.Activity)
        {
            case TestingActivity:
                _currentInstruction = VisualInstructor.Execute(Name, activityStart, InstructionCompleted);
                break;
        }
    }

    private void InstructionCompleted(int instructionResult, ActivityStart activity)
    {
        _currentInstruction = 0;
        var result = activity.CreateResult(instructionResult);
        _currentSession = result;
        PublishActivityCompleted(result);
    }

    public override void ProcessAborting(Activity affectedActivity)
    {
        if (_currentSession is ActivityStart activityStart)
        {
            VisualInstructor?.Clear(_currentInstruction);
            _currentInstruction = 0;
            activityStart.CreateResult((int)TestingActivityResults.Failed);
        }
    }

    public override void SequenceCompleted(SequenceCompleted completed)
    {
        _currentSession = completed;

        var rtw = CreateProductionReadyToWork(ReadyToWorkType.Push);
        PublishReadyToWork(rtw);
        _currentSession = rtw;
    }

    private ReadyToWork CreateProductionReadyToWork(ReadyToWorkType type) =>
        Session.StartSession(
            ActivityClassification.Production,
            type,
            PencilProductConstraints.ForPencilColor(AcceptedColor));

    Task IAsyncStateContext.SetStateAsync(StateBase state, CancellationToken cancellationToken)
    {
        throw new System.NotImplementedException();
    }
}

```

## TestingParameters for Manual text

Path: `src/PencilFactory/Activities/TestingStep/TestingParameters.cs`

Action: Replace/extend method `Populate` (rest of the class stays).

If the class was empty so far, only add `Populate` so that T-3 gets a readable instruction text:

```csharp
protected override void Populate(Process process, Parameters instance)
{
    base.Populate(process, instance);

    var parameters = (TestingParameters)instance;

    if (parameters.Instructions is { Length: > 0 })
        return;

    var color = "(unknown)";
    if (process is ProductionProcess production && production.ProductInstance?.Type is GraphitePencilType pencil)
        color = pencil.Color.ToString();

    parameters.Instructions =
    [
        new VisualInstruction
        {
            Type = InstructionContentType.Text,
            Content =
                $"MANUAL TESTING: Test the graphite pencil ({color}). " +
                "Confirm SUCCESS when the pencil has been tested."
        }
    ];
}
```

Auto cells ignore the text (Driver). Without `Populate` you only see empty SUCCESS/Failed buttons at T-3.

### Check your progress

* `ManualTesting` distinguishes auto vs manual capabilities
* `ManualTestingCell` attaches Brown constraint on ReadyToWork, `Populate` yields readable Manual text

---

## Create CellSelectors

Folder: `src/PencilFactory.ControlSystem/CellSelectors/`

Action: Add 3 new files.

### TestingOptimizerConfig.cs (new)

The config controls from which load manual cells are included:

```csharp
namespace PencilFactory.ControlSystem.CellSelectors;

public class TestingOptimizerConfig : CellSelectorConfig
{
    public override string PluginName
    {
        get => nameof(TestingOptimizer);
        set { }
    }

    [DataMember, DefaultValue(3)]
    [Description("Open testing activities per automatic cell before manual cells are included")]
    public int ActivitiesPerAutoCellThreshold { get; set; } = 3;
}
```

### TestingOptimizer.cs (new)

Below the threshold the optimizer returns only auto cells. From the threshold, the full list including Manual:

```csharp

namespace PencilFactory.ControlSystem.CellSelectors;

[ExpectedConfig(typeof(TestingOptimizerConfig))]
[Plugin(LifeCycle.Transient, typeof(ICellSelector), Name = nameof(TestingOptimizer))]
public class TestingOptimizer : CellSelectorBase<TestingOptimizerConfig>
{
    public IActivityPool ActivityPool { get; set; }

    public override Task<IReadOnlyList<ICell>> SelectCellsAsync(
        Activity activity,
        IReadOnlyList<ICell> availableCells,
        CancellationToken cancellationToken)
    {
        if (activity is not TestingActivity || availableCells.Count <= 1)
            return Task.FromResult(availableCells);

        var manualCapability = new TestingCapabilities { ManualTesting = true };
        var automaticCells = availableCells
            .Where(c => !c.Capabilities.Provides(manualCapability))
            .ToList();

        if (automaticCells.Count == 0)
            return Task.FromResult(availableCells);

        var openTestingActivities = ActivityPool.GetByCondition(a => a is TestingActivity).Count;
        var threshold = Config.ActivitiesPerAutoCellThreshold * automaticCells.Count;

        return openTestingActivities >= threshold
            ? Task.FromResult(availableCells)
            : Task.FromResult((IReadOnlyList<ICell>)automaticCells);
    }
}
```

Example: With 2 auto cells and threshold 3, the manual cell (T-3) only comes in from
6 open Testing activities.

### LoadBalancer.cs (new)

The LoadBalancer does not change the set of candidates, only the order: least loaded cell first:

```csharp
namespace PencilFactory.ControlSystem.CellSelectors;

/// <summary>
/// Reorders candidate cells by load (least busy first). Same cells as input, different order.
/// </summary>
[Plugin(LifeCycle.Transient, typeof(ICellSelector), Name = nameof(LoadBalancer))]
public class LoadBalancer : CellSelectorBase
{
    // Persists across calls: open activity maps to the cell we ranked first for it
    private readonly Dictionary<IActivity, ICell> _primaryTarget = [];

    public override Task<IReadOnlyList<ICell>> SelectCellsAsync(
        Activity activity,
        IReadOnlyList<ICell> availableCells,
        CancellationToken cancellationToken)
    {
        if (availableCells.Count <= 1)
        {
            return Task.FromResult(availableCells);
        }

        // Rebuilt every call, the load is derived from _primaryTarget below
        var cellLoad = availableCells.ToDictionary(c => c, _ => 0);

        lock (_primaryTarget)
        {
            // Drop finished activities
            foreach (var key in _primaryTarget.Keys.ToList())
            {
                if (key.Result != null)
                {
                    _primaryTarget.Remove(key);
                }
            }

            // Bump load for cells still booked by running activities
            foreach (var cell in _primaryTarget.Values.Where(cellLoad.ContainsKey))
            {
                cellLoad[cell]++;
            }
        }

        var loadBalanced = cellLoad.OrderBy(pair => pair.Value).Select(pair => pair.Key).ToList();

        lock (_primaryTarget)
        {
            // Remember for the next call so parallel orders see this cell as busy
            _primaryTarget[activity] = loadBalanced[0];
        }

        return Task.FromResult((IReadOnlyList<ICell>)loadBalanced);
    }
}

```

## Extend the initializer

Path: `src/PencilFactory.App/PencilFactoryInitializer.cs`

### Adjust Display Description (optional)

```csharp
[Display(..., Description = "Creates cells (two automatic and one manual testing station), visual instructor and factory layout.")]
```

### Insert block after T-1 (second auto cell)

Replace the previous single Testing block (one driver, one cell, location `"T-1"`) with two automatic stations:

```csharp
// Automated testing cells (LoadBalancer distributes between T-1 and T-2)
var testingDriver1 = graph.Instantiate<SimulatedTestingDriver>();
testingDriver1.Name = "Testing Simulated Driver 1";

var testingCell1 = graph.Instantiate<TestingCell>();
testingCell1.Name = "Testing Cell 1";
testingCell1.Driver = testingDriver1;
testingCell1.Value = 0;
lineGroup.Children.Add(Place(graph, "T-1", "Automatic testing 1", "precision_manufacturing", 0.58, 0.32, testingCell1));

var testingDriver2 = graph.Instantiate<SimulatedTestingDriver>();
testingDriver2.Name = "Testing Simulated Driver 2";

var testingCell2 = graph.Instantiate<TestingCell>();
testingCell2.Name = "Testing Cell 2";
testingCell2.Driver = testingDriver2;
testingCell2.Value = 0;
lineGroup.Children.Add(Place(graph, "T-2", "Automatic testing 2", "precision_manufacturing", 0.74, 0.32, testingCell2));
```

### Insert Manual cell before Packing

```csharp
var manualTestingCell = graph.Instantiate<ManualTestingCell>();
manualTestingCell.Name = "Manual Testing Cell";
manualTestingCell.VisualInstructor = instructor;
manualTestingCell.Value = 0;
manualTestingCell.AcceptedColor = PencilColor.Brown;
lineGroup.Children.Add(Place(graph, "T-3", "Manual testing (Brown Classic)", "handyman", 0.66, 0.52, manualTestingCell));
```

### Factory children: add second driver

```csharp
factory.Children.Add(testingDriver1);
factory.Children.Add(testingDriver2);
```

### Prepare separate displays for parallel work

Before clearing/recreating resources for the load test, extend the chapter-9 initializer. After the existing factory and cells have been created and **before returning ResourceInitializerResult**, add:

```csharp
var assemblingInstructor = graph.Instantiate<VisualInstructor>();
assemblingInstructor.Name = "Assembling Instructor";
assemblingCell.VisualInstructor = assemblingInstructor;
factory.Children.Add(assemblingInstructor);

var colorizingInstructor = graph.Instantiate<VisualInstructor>();
colorizingInstructor.Name = "Colorizing Instructor";
colorizingCell.VisualInstructor = colorizingInstructor;
factory.Children.Add(colorizingInstructor);

var testingInstructor = graph.Instantiate<VisualInstructor>();
testingInstructor.Name = "Manual Testing Instructor";
manualTestingCell.VisualInstructor = testingInstructor;
factory.Children.Add(testingInstructor);

var packingInstructor = graph.Instantiate<VisualInstructor>();
packingInstructor.Name = "Packing Instructor";
packingCell.VisualInstructor = packingInstructor;
factory.Children.Add(packingInstructor);
```

The previous shared instructor can remain as an unused resource. In Worker Support select the display of the station you are operating. Use separate browser tabs when confirming different stations. The earlier screenshots show the sequential example's shared display.

### Afterwards

1. Clear Resources DB
2. Command Center, ResourceManager, Console: Initialize Resource
3. Check: A-1, C-1, T-1, T-2, T-3, P-1 + both drivers under Pencil Manufactory

### Check your progress

* `TestingOptimizer` and `LoadBalancer` compile, initializer creates T-1, T-2, T-3 after DB clear + Initialize

---

## ProcessEngine + Orders Management (Command Center)

### ProcessEngine

1. CONFIGURATION, ResourceSelectors: TestingOptimizer (Threshold 1, SortOrder 0)
2. ResourceSelectors: LoadBalancer (SortOrder 1)
3. JobSchedulerConfig, MaxActiveJobs: 4
4. SAVE + RESTART

![ProcessEngine ResourceSelectors configuration](./chapter-11/process-engine-selectors.png)

### Orders Management

1. CONFIGURATION, MaxRunningOperations: 4
2. SAVE + RESTART

![Order Management MaxRunningOperations set to 4](./chapter-11/order-management-max-running.png)

Selector chain:

| SortOrder | Plugin | Task |
| ---- | ----- | --- |
| 0 | TestingOptimizer | Manual only under load |
| 1 | LoadBalancer | Fair distribution T-1 / T-2 |

Order matters: LoadBalancer after Optimizer.

### Check your progress

* ProcessEngine: TestingOptimizer (SortOrder 0), LoadBalancer (SortOrder 1). MaxActiveJobs / MaxRunningOperations raised for parallel tests

## Testing

Note: Under the "Processes" view you can track which cell the ProcessEngine chooses for a TestingActivity.

### Why Manual is hard to hit in this line

Each pencil runs the workplan **in sequence**: Assembling -> Colorizing -> Testing.
With **one** Assembling cell and **one** Colorizing cell, pieces rarely pile up at
Testing at the same time. Auto cells T-1/T-2 usually finish before a third Testing
activity is waiting, so the Optimizer threshold is almost never reached and T-3
stays unused. That is expected with this layout, not a broken Manual cell.

### Practical trick: shorten Colorizing in MachineSimulator

The bottleneck before Testing is often Colorizing (default ~2400 ms). If Colorizing
is much faster, more pencils reach Testing while T-1/T-2 are still busy. Then the
Optimizer can include Manual.

In **Command Center** -> **MachineSimulator** -> **CONFIGURATION**:

1. **Specific execution times** -> add / set:
   - **Activity:** `PencilFactory.Activities.ColorizingStep.ColorizingActivity`
   - **ExecutionTime:** `500` ms
   - **CellId:** `0` (all cells)
2. **SAVE + RESTART**
3. Start several **Brown** orders (`100002`) in parallel and work Assembling promptly

![MachineSimulator ColorizingActivity ExecutionTime set to 500 ms](./chapter-11/simulator-colorizing-execution-500.png)

In practice this alone is often enough to see T-3 (Manual) a few times. You do **not**
need to change Testing execution time or Success rate for that.

### A) Green (100001): Constraints

One order is enough.

- Testing only T-1 / T-2 (automatic)
- T-3 gets nothing. Constraint: only Brown Classic.

### B) Brown (100002): Optimizer + Constraints

Start several Brown orders in parallel (ideally with Colorizing at 500 ms as described above).

- Normal / low load: like Green, only T-1 and T-2
- Under load (Threshold 1, enough **parallel** processes already at Testing): additionally Instruction at T-3 (Manual)
- Raise threshold to 3: T-3 less often

Here T-1, T-2 and T-3 can all be in play at once. That is the Optimizer, not a pure LoadBalancer test.

### C) LoadBalancer: T-1 / T-2

Several parallel orders with Green (`100001`) in parallel, not Brown Classic.

- Only auto cells (T-3 excluded via constraint)
- Observe several activity assignments: both T-1 and T-2 should be used under load. Do not infer a strict alternating schedule from this selector. Completion timing and available candidates affect each choice.

If a manual station appears idle, first select its dedicated display in Worker Support. Confirm the instructor references from the initializer before interpreting a missing instruction as a routing error.

### Check your progress

* Green: only T-1/T-2. Brown under load can reach T-3, repeated parallel Green activities use both automatic cells (exact alternation is not required)

## Checklist

* [ ] `TestingCapabilities.ManualTesting` and `TestingCell` adjusted
* [ ] `PencilProductConstraints` and `ManualTestingCell` created
* [ ] `TestingParameters.Populate` added for Manual text
* [ ] `TestingOptimizer` and `LoadBalancer` created and configured in ProcessEngine
* [ ] Initializer extended with T-2 and T-3, resources re-initialized
* [ ] MaxActiveJobs / MaxRunningOperations set
* [ ] Tests A-C (Green, Brown Classic, LoadBalancer) completed
* [ ] Colorizing `ExecutionTime` 500 ms in MachineSimulator so Manual appears under load

## Summary

Capabilities establish eligibility, constraints restrict a resource's accepted processes and selectors filter/rank candidates. The optimizer's threshold depends on open activities and available automatic cells. Testing under load requires observing several assignments, not a single screenshot.

## Reflect

1. Why must Green stay off T-3 even when the optimizer admits manual cells?
2. How do the optimizer and load balancer change the candidates differently?
3. Why can changing Colorizing execution time affect Testing-cell use?

## Practice

Compare **ActivitiesPerAutoCellThreshold = 1** and **3** with a similar Brown workload. Before each run, predict whether manual Testing (T-3) should become more or less likely. Note which cells actually get Testing assignments.

Use a Green order as control: it must not reach T-3 at either threshold. Restore threshold **1** afterward.

<details>
<summary>Hint</summary>

Threshold **1** lets the manual cell in sooner when Testing is busy. Threshold **3** waits longer before using T-3. Green never goes to T-3 because that cell only accepts Brown. The threshold does not change that.

</details>

## Check your reasoning

<details>
<summary>Compare your answers after attempting the questions and practice</summary>

1. The Brown constraint remains an eligibility condition for the manual resource. A selector cannot make a Green process satisfy it.
2. TestingOptimizer can drop manual candidates when load is low. LoadBalancer only ranks the cells that remain. It does not promise strict alternation.
3. Faster Colorizing can feed Testing before previous Testing activities finish, increasing concurrent demand. Assembling speed and other limits still affect that demand.

**Practice feedback:** Threshold 3 should make T-3 less likely than threshold 1 under similar load. Green never on T-3. Note what you saw. A few runs are not a performance proof.

</details>

## Troubleshooting

| Symptom | Likely cause |
| ---- | ----- |
| T-3 never used | Threshold not reached, Colorizing bottleneck, only Green orders (constraint) |
| Green reaches T-3 | Constraint missing / wrong `AcceptedColor` / ReadyToWork without constraint |
| Always same auto cell | LoadBalancer not configured or SortOrder before Optimizer incorrectly |
| Worker Support shows wrong / overlapping instructions when several orders run | Several manual cells still share **one** VisualInstructor. With MaxRunningOperations > 1 they can push instructions to the same display. Give Assembling, Colorizing, Manual Testing and Packing each their own instructor (as in the initializer change above) |
| Selectors have no effect | ProcessEngine not saved/restarted, wrong plugin names |
| No Testing cell at all | Capabilities/`ProvidedBy` broken, see [Troubleshooting](troubleshooting.md#processengine-routing-no-matching-cell) |

See also [Troubleshooting](troubleshooting.md) and [Help](README.md#help).

> [Table of contents](README.md) | [Previous](chapter-10-module-adapter.md) | [Next](chapter-12-advanced-topics.md)
