# Chapter 4 - Testing

Too many pencils leave the line unchecked. After Assembling and Colorizing, *Pencilla Inc.* adds Testing as a third station.

Build Testing yourself with the same automatic pattern as Colorizing in [chapter 2](chapter-02-drivers.md). That transfer **is** this chapter's main exercise. Keep Colorizing open as a reference. Use the [reference solution](#reference-solution-only-if-stuck) only if you get stuck.


> [Table of contents](README.md) | [Previous](chapter-03-capabilities.md) | [Next](chapter-05-setup.md)

**On this page:** [Goals](#learning-goals) | [Starting point](#before-you-start) | [Practice](#practice) | [Check your reasoning](#check-your-reasoning) | [Troubleshooting](#troubleshooting)

## Learning goals

By the end of this chapter, you should be able to:

* Add Testing and implement the Ready / ProcessStart / ProcessResult handshake (cell + simulated driver)
* Reuse the Colorizing pattern with Testing type names
* Put Testing into the workplan and verify it in Processes

## Where you are in the journey

* Assembling / Colorizing: chapters 1-3
* **Testing**: this chapter (same automatic pattern as Colorizing)
* Packing: chapters 7-8

## Before you start

Finish chapters 1-3, so Assembling, Colorizing (with Green/Brown routing) and the shared workplan run.

You will create a TestingCell and SimulatedTestingDriver the same way you did with Colorizing: same signals (`Ready`, `ProcessStart`, `ProcessResult`), Testing activity and result types instead of Colorizing ones.

## What you will touch

| Kind | Items |
| ---- | ----- |
| CLI | `moryx add step Testing` |
| Projects | `PencilFactory.Resources.Testing` (package `Moryx.Drivers.Simulation` if needed) |
| Classes | `TestingCell`, `SimulatedTestingDriver` |
| UI | Resources (TestingCell + driver), Workplans, Orders, Processes |

## Add the Testing step

From the PencilFactory project root:

```bash
moryx add step Testing
```

That scaffolds the activity, cell project and related files.


> **Note:** The CLI often leaves placeholders (`Some`, `MyApplication`) or does not wire the new project into the solution. If the build fails, see [Troubleshooting: CLI leftovers](#troubleshooting-cli-leftovers-after-moryx-add-step-testing), fix those points, then continue.

### Check your progress

* `moryx add step Testing` completed from the PencilFactory root
* Solution builds (after fixing CLI leftovers if needed)

## TestingCell (mirror Colorizing)

> **Concept:** The machine handshake stays the same: Ready -> ProcessStart -> ProcessResult -> reset ProcessStart. What changes is the domain: `TestingActivity` and `TestingActivityResults`.

Open `ColorizingCell` next to `TestingCell`. Implement Testing so that:

1. **Constants** for `ProcessStart`, `ProcessResult` and `Ready` match Colorizing (same strings).
2. **`Driver`** is an `IInOutDriver` with subscribe/unsubscribe on `InputChanged` in the setter.
3. **`ProcessEngineAttached`** starts a Production session with `ReadyToWorkType.Push`.
4. **`StartActivity`** sets `Driver.Output[ProcessStart] = true` for a `TestingActivity`.
5. **`ProcessAborting`** clears `ProcessStart` and reports Failed with `TestingActivityResults`.
6. **`SequenceCompleted`** publishes ReadyToWork (Push).
7. **`OnInputChanged`**
   * On `Ready == true` (and not in an ActivityStart): ReadyToWork **Pull**
   * On `ProcessResult` during ActivityStart: set `ProcessStart = false`, read the result, publish Success or Failed via `TestingActivityResults`

Do not paste Colorizing unchanged. Rename every Colorizing type to the Testing equivalent.

### Overall flow (target behavior)

![Overall testing flow](./chapter-04/testing-overall-flow.png)

### Check your progress

* `TestingCell` compiles
* You can point to the Colorizing methods you mirrored

## SimulatedTestingDriver

Create `src/PencilFactory.Resources.Testing/SimulatedTestingDriver.cs`.

Mirror `SimulatedColorizingDriver`:

* Inherit `SimulatedInOutDriver`
* Same three signal names (`Ready`, `ProcessStart`, `ProcessResult`)
* `Ready(Activity)` sets Ready and raises `InputChanged`
* `OnOutputSet` reacts to `ProcessStart` (Idle vs Executing)
* `Result(SimulationResult)` writes `ProcessResult` from **`TestingActivityResults.Success`**, then raises `InputChanged`

### Step-by-step flow (target behavior)

Simulation applies the result after the execution time (no Success/Failed click):

![Step-by-step testing flow](./chapter-04/testing-step-by-step-flow.png)

In short: the simulator calls `Ready` first -> the cell Pulls work -> the Process Engine sends `TestingActivity` -> the cell sets `ProcessStart` -> after the fake run, `Result` / `ProcessResult` arrives -> the cell resets `ProcessStart` and reports `ActivityCompleted`. Same handshake as Colorizing.

### Check your progress

* `SimulatedTestingDriver` compiles and uses Testing result types, not Colorizing types

## If you are stuck (hints)

Use these only after you tried mirroring Colorizing yourself:

1. Signal strings must match between cell and driver (`"Ready"`, `"ProcessStart"`, `"ProcessResult"`).
2. After `ProcessResult`, reset `ProcessStart` to `false` or the next cycle may hang.
3. Subscribe and unsubscribe in the Driver setter (same bug class as chapter 2).
4. Fix CLI leftovers before chasing handshake bugs.
5. Re-read chapter 2 (`OnInputChanged`, `SimulatedColorizingDriver`) if a branch is unclear.
6. If you still cannot finish, open the [Reference solution](#reference-solution-only-if-stuck) so later chapters stay reachable.

## Resources in the UI

1. Create **SimulatedTestingDriver**
2. Create **TestingCell**
3. Set the TestingCell **Driver** reference to the SimulatedTestingDriver

![TestingCell with SimulatedTestingDriver](./chapter-04/testing-cell-and-driver.png)

### Check your progress

* TestingCell and SimulatedTestingDriver exist. Driver reference is set
* Testing needs no Worker Support click (simulation drives Ready / Result)

## Workplan and verify

Edit the existing [Workplan](https://github.com/PHOENIXCONTACT/MORYX-Framework/blob/main/docs/articles/abstractions/processing/workplans.md) and insert Testing between Colorizing and the end. Route Failed to the Failed connector, Success onward.

![Workplan with Testing step](./chapter-04/workplan-with-testing.png)

Start an order. Under **Processes** you see the running activities and selected cells. That is how you verify Testing.

![Process view with TestingActivity](./chapter-04/process-with-testing-activity.png)

### Check your progress

* Workplan: Assembling -> Colorizing -> Testing (Failed routed correctly)
* A full order shows `TestingActivity` in Processes with Success or Failed from the simulator

Finish Testing before chapters 5+, they assume Testing is already in the workplan.

## Reference solution (only if stuck)

<details>
<summary>Open the Testing reference after trying the task and hints</summary>

Try with Colorizing open first. Use this only if you are blocked and need a working Testing step for later chapters.

### TestingCell (core handshake)

```cs
private const string ProcessStart = "ProcessStart";
private const string ProcessResult = "ProcessResult";
private const string Ready = "Ready";

[ResourceReference(ResourceRelationType.Driver)]
public IInOutDriver Driver
{
    get => field;
    set
    {
        if (field?.Input != null)
            field.Input.InputChanged -= OnInputChanged;

        field = value;

        if (field?.Input != null)
            field.Input.InputChanged += OnInputChanged;
    }
}

protected override IEnumerable<Session> ProcessEngineAttached()
{
    yield return Session.StartSession(ActivityClassification.Production, ReadyToWorkType.Push);
}

public override void StartActivity(ActivityStart activityStart)
{
    _currentSession = activityStart;
    if (activityStart.Activity is TestingActivity)
    {
        Driver.Output[ProcessStart] = true;
    }
}

public override void ProcessAborting(Activity affectedActivity)
{
    if (_currentSession is ActivityStart activityStart)
    {
        VisualInstructor?.Clear(_currentInstruction);
        if (Driver != null)
        {
            Driver.Output[ProcessStart] = false;
        }

        activityStart.CreateResult((int)TestingActivityResults.Failed);
    }
}

public override void SequenceCompleted(SequenceCompleted completed)
{
    _currentSession = completed;

    var rtw = Session.StartSession(ActivityClassification.Production, ReadyToWorkType.Push);
    PublishReadyToWork(rtw);
    _currentSession = rtw;
}

private void OnInputChanged(object sender, InputChangedEventArgs args)
{
    if (args.Key == Ready && (bool)args.Value && _currentSession is not ActivityStart)
    {
        var rtw = Session.StartSession(ActivityClassification.Production, ReadyToWorkType.Pull);
        _currentSession = rtw;
        PublishReadyToWork(rtw);
    }
    else if (args.Key == ProcessResult && _currentSession is ActivityStart activitySession)
    {
        Driver.Output[ProcessStart] = false;
        var processResult = (bool)Driver.Input[ProcessResult];

        var result = activitySession.CreateResult(
            processResult
                ? (int)TestingActivityResults.Success
                : (int)TestingActivityResults.Failed);
        _currentSession = result;
        PublishActivityCompleted(result);
    }
}
```

### SimulatedTestingDriver

```cs
using Moryx.AbstractionLayer.Activities;
using Moryx.AbstractionLayer.Resources;
using Moryx.ControlSystem.Simulation;
using Moryx.Drivers.Simulation.InOutDriver;
using PencilFactory.Activities.TestingStep;

namespace PencilFactory.Resources.Testing;

[ResourceRegistration]
public class SimulatedTestingDriver : SimulatedInOutDriver
{
    private const string ProcessStart = "ProcessStart";
    private const string ProcessResult = "ProcessResult";
    private const string ReadyToWork = "Ready";

    public override void Ready(Activity activity)
    {
        SimulatedState = SimulationState.Requested;
        SimulatedInput.Values[ReadyToWork] = true;
        SimulatedInput.RaiseInputChanged(ReadyToWork, SimulatedInput.Values[ReadyToWork]);
    }

    protected override void OnOutputSet(object sender, string key)
    {
        if (key == ProcessStart)
        {
            SimulatedInput.Values[ReadyToWork] = false;
            SimulatedState = (bool)SimulatedOutput.Values[ProcessStart]
                ? SimulationState.Executing
                : SimulationState.Idle;
        }
    }

    public override void Result(SimulationResult result)
    {
        SimulatedInput.Values[ProcessResult] = result.Result == (int)TestingActivityResults.Success;
        SimulatedInput.RaiseInputChanged(ProcessResult, SimulatedInput.Values[ProcessResult]);
    }
}
```

</details>

## Checklist

* [ ] `moryx add step Testing` done. CLI leftovers fixed if needed
* [ ] `TestingCell` mirrors the Colorizing handshake with Testing types
* [ ] `SimulatedTestingDriver` mirrors Colorizing simulation with Testing results
* [ ] Driver linked on the TestingCell in Resources
* [ ] Testing in the workplan, order runs through Testing in Processes

## Summary

Testing reuses the Colorizing automatic pattern: same Ready / ProcessStart / ProcessResult handshake, Testing types and project names. You add the step to the workplan and verify it under Processes.

## Reflect

1. What stayed the same as Colorizing and what did you have to rename for Testing?
2. Why does the operator not click SUCCESS / FAILED at Testing (unlike Assembling)?
3. If you left a `ColorizingActivity` type name inside Testing code, what would go wrong?

## Practice

**Optional.** Later chapters continue with **one** TestingCell. Only do this if you want a short extra check on multiple resources.

**Why:** Same idea as two ColorizingCells, but Testing has no color split. Two tester instances can both match.

1. Predict: Can a second TestingCell (own SimulatedTestingDriver) run `TestingActivity` without new C#?
2. Add that second cell + driver in Resources, run an order, note in Processes which cell ran Testing.
3. **Remove** the second TestingCell and its driver again so chapter 5+ keep a single tester.

**Done when:** your prediction matched what you saw and only one TestingCell remains.

<details>
<summary>Hint</summary>

`TestingActivity` requires `TestingCapabilities` without pencil color, so any matching TestingCell is eligible. Delete the extra resources when you are finished.

</details>

## Check your reasoning

<details>
<summary>Compare your answers after attempting the questions and practice</summary>

1. Same: signal strings and the handshake methods. Renamed: cell, activity, results, simulated driver, namespaces/project.
2. Testing is driven by the simulated driver (Ready / ProcessResult), like Colorizing, not by Worker Support.
3. The wrong type never matches in `StartActivity` / `OnInputChanged`, so Testing may never start or never complete.

**Practice feedback:** Optional only. A second TestingCell can run Testing without new code. Remove it afterwards. Unlike Colorizing, the two testers need no different colors.

</details>

## Troubleshooting

Driver and simulation issues: see also [chapter 2 Troubleshooting](chapter-02-drivers.md#troubleshooting).

### Troubleshooting: CLI leftovers after `moryx add step Testing`

The CLI can leave placeholder names (`Some`, `MyApplication`) or skip wiring the new project into the solution. If build or IntelliSense fails after the command, fix the following.

**1. Add the project to the solution and to the App**

- Solution -> right-click -> **Add** -> **Existing Project** -> `PencilFactory.Resources.Testing.csproj`
- Right-click **PencilFactory.App** -> **Add** -> **Project Reference** -> check **PencilFactory.Resources.Testing**

**2. Fix the resource project reference**

In `PencilFactory.Resources.Testing.csproj`, replace a leftover template reference:

```xml
<!-- Wrong (CLI placeholder) -->
<ProjectReference Include="..\MyApplication\MyApplication.csproj" />

<!-- Correct -->
<ProjectReference Include="..\PencilFactory\PencilFactory.csproj" />
```

**3. Fix namespaces in `TestingCell.cs`**

```csharp
// Wrong (CLI placeholder)
using MyApplication.Activities.SomeStep;
using MyApplication.Capabilities;

// Correct
using PencilFactory.Activities.TestingStep;
using PencilFactory.Capabilities;

namespace PencilFactory.Resources.Testing;
```

**4. Replace leftover `Some` placeholders**

Search the Testing resource project (Ctrl+F) for `Some` and rename to `Testing`, for example:

- `SomeStateBase`: keep or rename, the class should target `TestingCell`
- `public class TestingCell : Cell, IAsyncStateContext`
- `Capabilities = new TestingCapabilities { Value = Value };`
- `activityStart.CreateResult((int)TestingActivityResults.Failed);`

Example for the generated state base file (`TestingStateBase.cs`):

```csharp
using Moryx.StateMachines;

namespace PencilFactory.Resources.Testing;

internal abstract class SomeStateBase(TestingCell context, StateBase.StateMap stateMap)
    : AsyncStateBase<TestingCell>(context, stateMap)
{
}
```

If the class is still named `SomeStateBase`, rename it for clarity (e.g. `TestingStateBase`) or leave it until chapter 12 if you do not use states on Testing yet. The important part is that usings and types refer to **Testing**, not `Some` / `MyApplication`.

Then rebuild the solution.

| Symptom | Likely cause |
| --- | --- |
| Testing never appears in workplan | Testing step/project not loaded, App reference missing |
| Build errors after `moryx add step` | CLI leftovers (`Some` / `MyApplication`). See above |
| Testing hangs like Colorizing did | Driver not linked, `ProcessStart` not reset, subscription missing |
| Order stops after Colorizing | Workplan Testing node not connected |

See also [Troubleshooting](troubleshooting.md) and [Help](README.md#help).

> [Table of contents](README.md) | [Previous](chapter-03-capabilities.md) | [Next](chapter-05-setup.md)
