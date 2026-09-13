# Chapter 5 - Setup

Keeping a ColorizingCell for every color turned out to be costly. *Pencilla Inc.* goes back to a single cell and expects MORYX to prepare it first whenever the paint does not match the order. You implement that as a ColorChange setup before Assembling, Colorizing and Testing.

> [Table of contents](README.md) | [Previous](chapter-04-testing.md) | [Next](chapter-06-cleanup.md)

**On this page:** [Goals](#learning-goals) | [Starting point](#before-you-start) | [Practice](#practice) | [Check your reasoning](#check-your-reasoning) | [Troubleshooting](#troubleshooting)

## Learning goals

By the end of this chapter, you should be able to:

* Add a ColorChange **setup** activity and let one ColorizingCell change its paint color
* Register a SetupTrigger so setup runs automatically when the cell color does not match the product
* Check that a Brown order does ColorChange first, then Assembling -> Colorizing -> Testing

## Where you are in the journey

* Assembling / Colorizing / Testing: chapters 1-4 (two ColorizingCells by color)
* **This chapter**: back to **one** ColorizingCell. Setup prepares it before production
* Cleanup: chapter 6. Packing: later

## Before you start

Finish chapter 4 so Assembling -> Colorizing -> Testing runs (including Testing).

Chapter 3 used one ColorizingCell per color. Here you keep only **one** cell (start as Green). Before coding: what happens to a Brown ColorizingActivity if only a Green cell exists? You will fix that with Setup, not with a second cell.

## What you will touch

| Kind | Items |
| ---- | ----- |
| CLI | `moryx add step ColorChange` (then delete unused resource files) |
| Projects | `PencilFactory` (`ColorChangeStep/*`, `ColorizingCapabilities`), `PencilFactory.ControlSystem` (SetupTriggers), `PencilFactory.Resources.Colorizing` |
| Classes | `ColorChangeActivity`, `ColorChangeParameters`, `ProvideColorTrigger`, updates on `ColorizingCell` |
| UI | Resources (one ColorizingCell + VisualInstructor + Driver), SetupProvider, Worker Support, Orders |

For new files, let Visual Studio add missing usings (`Ctrl + .`).

## Why setup?

Chapter 3 used one ColorizingCell per color. Now the other extreme: **one** cell (Color = Green). A Brown order finds no matching cell and stalls.

> **Concept:** **Setup** is a separate job **before** production. It changes the plant state
> (here: Color of the ColorizingCell) until capabilities match again. Setup jobs are created by the
> **SetupProvider** from triggers. They are **not** drawn as normal steps in the production workplan.
> Production still runs Assembling -> Colorizing -> Testing. Setup only prepares the resource graph.

How a Brown order runs when the cell is still Green (and what happens if the cell is already Brown, or ColorChange fails):

![Setup then production flow](./chapter-05/setup-flow.png)

In short: the trigger names the **target** color from the product. SetupProvider creates ColorChange only if the cell does not already provide it. Success updates `Cell.Color`. Then the normal production workplan runs.

More on [Activities](https://github.com/PHOENIXCONTACT/MORYX-Framework/blob/main/docs/articles/abstractions/processing/activities.md)
and [Cells](https://github.com/PHOENIXCONTACT/MORYX-Framework/blob/main/docs/articles/abstractions/control-system/cell-resource.md)
in the framework.

## Create the ColorChange step

In the project directory run:

```bash
moryx add step ColorChange
```

The CLI creates more than you need. Delete the following:

- `src/PencilFactory.Resources.ColorChange/` (entire folder)
- `src/Tests/PencilFactory.Tests/ColorChangeCellTest.cs`
- `src/PencilFactory/Capabilities/ColorChangeCapabilities.cs`
- `src/PencilFactory/Resources/IColorChangeResource.cs`

Keep:

- `src/PencilFactory/Activities/ColorChangeStep/*`

### Remove references

From these files, remove references to the deleted resource project:

- `PencilFactory.sln`
- `src/PencilFactory.App/PencilFactory.App.csproj`
- `src/Tests/PencilFactory.Tests/PencilFactory.Tests.csproj`

## Update ColorizingCapabilities for setup

Chapter 3 matched a concrete color only. Setup needs **any** ColorizingCell:
`ColorChangeActivity` uses `new ColorizingCapabilities()` without a color.
Make `Color` nullable and treat `null` as "any colorizing cell":

File: `src/PencilFactory/Capabilities/ColorizingCapabilities.cs`

```cs
public class ColorizingCapabilities : CapabilitiesBase
{
    /// <summary>
    /// Required color for production matching.
    /// Null means "any colorizing cell" (used by setup / color change).
    /// </summary>
    public PencilColor? Color { get; set; }

    protected override bool ProvidedBy(ICapabilities provided)
    {
        if (provided is not ColorizingCapabilities providedColorizing)
        {
            return false;
        }

        // Setup: no specific color required -> any colorizing cell matches
        if (Color == null)
        {
            return true;
        }

        return providedColorizing.Color == Color;
    }
}
```

Without this, Brown orders fail with: `No resource has required capabilities for 'ColorChangeActivity'`.

## ColorChangeActivity

Adjust the file: `src/PencilFactory/Activities/ColorChangeStep/ColorChangeActivity.cs`

Why these changes? Setup is not production: the activity needs `ActivityClassification.Setup`,
no process object and any ColorizingCell.

- Usings: `Moryx.ControlSystem.Activities`, `PencilFactory.Capabilities`
- Implement `IControlSystemActivity`
- `Classification => ActivityClassification.Setup`
- `RequiredCapabilities => new ColorizingCapabilities()`: you do not set `Color`, so it stays **`null`** ("any colorizing cell")

```cs
/// <summary>
/// Setup activity: change the paint color on a colorizing cell before production.
/// </summary>
[ActivityResults(typeof(ColorChangeActivityResults))]
public class ColorChangeActivity : Activity<ColorChangeParameters>, IControlSystemActivity
{
    public ActivityClassification Classification => ActivityClassification.Setup;

    public override ProcessRequirement ProcessRequirement => ProcessRequirement.NotRequired;

    /// <summary>
    /// Any colorizing cell can perform a color change (Color defaults to null = "any").
    /// </summary>
    public override ICapabilities RequiredCapabilities => new ColorizingCapabilities();

    protected override ActivityResult CreateResult(long resultNumber)
    {
        return ActivityResult.Create((ColorChangeActivityResults)resultNumber);
    }

    protected override ActivityResult CreateFailureResult()
    {
        return ActivityResult.Create(ColorChangeActivityResults.Failed);
    }
}

```

## ColorChangeActivityResults

File: `src/PencilFactory/Activities/ColorChangeStep/ColorChangeActivityResults.cs`

> **Note:** Setup steps need exactly 3 outputs so the SetupProvider can wire the
> workplan correctly: Success, TechnicalError, Failed.

```cs
public enum ColorChangeActivityResults
{
    [Display(Name = "Success")]
    Success = 0,

    [Display(Name = "Technical Error")]
    TechnicalError = 1,

    [Display(Name = "Failed")]
    Failed = 2
}
```

## ColorChangeParameters

Add `PencilColor Color` and copy it in `Populate`. The SetupTrigger sets `Color` on the task parameters from the product. `Populate` copies that onto the activity instance. The cell later reads `activity.Parameters.Color` when ColorChange succeeds.

```cs
public class ColorChangeParameters : VisualInstructionParameters
{
    public PencilColor Color { get; set; }

    protected override void Populate(Process process, Parameters instance)
    {
        base.Populate(process, instance);
        var parameters = (ColorChangeParameters)instance;
        parameters.Color = Color;
    }
}
```

## ColorChangeTask

File: `ColorChangeTask.cs`

Optional: a clear Description on the task makes it easier to read in the workplan editor.

```cs
[Display(Name = "ColorChange", Description = "Change the paint color on a colorizing cell (typically used as setup)")]
public class ColorChangeTask : TaskStep<ColorChangeActivity, ColorChangeParameters>
{
}
```

## Extend ColorizingCell for setup

The ColorizingCell must offer setup sessions and run ColorChange like a manual
instruction, similar to Assembling, but with `ActivityClassification.Setup`.


### Setup and production sessions

```cs
protected override IEnumerable<Session> ProcessEngineAttached()
{
    yield return Session.StartSession(ActivityClassification.Setup, ReadyToWorkType.Push);
    yield return Session.StartSession(ActivityClassification.Production, ReadyToWorkType.Push);
}
```

### ColorChange in StartActivity

```cs
case ColorChangeActivity:
    _currentInstruction = VisualInstructor.Execute(Name, activityStart, ColorChangeCompleted);
    break;
```

**Important: don't let the simulated driver finish ColorChange:**  
The ColorizingCell has a `SimulatedColorizingDriver` for production. The Process Simulator
also schedules a `Result` when *any* activity on that cell becomes `Running`, including
setup. If `OnInputChanged` treats every `ProcessResult` as "activity done", ColorChange
is auto-completed after ~1-3 seconds, production starts and Assembling appears while the
ColorChange instruction is still open.

Unlike chapter 3's driver-only ColorizingCells, this cell mixes **Instructor (setup)** and **Driver
(production)**. Guard both places:

```cs
// ColorizingCell.OnInputChanged: only production
else if (args.Key == ProcessResult
         && _currentSession is ActivityStart activitySession
         && activitySession.Activity is ColorizingActivity)
{
    // ... complete Colorizing as before ...
}
```

```cs
// SimulatedColorizingDriver.Result
if (result.Activity is not ColorizingActivity)
    return;
```

### Set Color after confirmation

Only on Success is the cell color changed. Failed / TechnicalError keep the old color.

```cs
private void ColorChangeCompleted(int instructionResult, ActivityStart activity)
{
    // Only Success changes the cell color. Failed / TechnicalError keep the old color
    if (instructionResult == (int)ColorChangeActivityResults.Success)
    {
        var colorChange = (ColorChangeActivity)activity.Activity;
        Color = colorChange.Parameters.Color;
    }

    _currentInstruction = 0;
    var result = activity.CreateResult(instructionResult);
    _currentSession = result;
    PublishActivityCompleted(result);
}
```

### SequenceCompleted: Setup vs Production

After setup, offer a setup session again. After production, offer production again,
otherwise the cell hangs.

```cs
public override void SequenceCompleted(SequenceCompleted completed)
{
    _currentSession = completed;

    if (completed.AcceptedClassification == ActivityClassification.Setup)
    {
        var setupRtw = Session.StartSession(ActivityClassification.Setup, ReadyToWorkType.Push);
        PublishReadyToWork(setupRtw);
        _currentSession = setupRtw;
        return;
    }

    var rtw = Session.StartSession(ActivityClassification.Production, ReadyToWorkType.Push);
    PublishReadyToWork(rtw);
    _currentSession = rtw;
}
```

## SetupTrigger

The trigger decides whether setup is needed before production and creates the
ColorChange task. Under the PencilFactory.ControlSystem project, create the folder
SetupTriggers and the following files.

### Config class

File: `src/PencilFactory.ControlSystem/SetupTriggers/ProvideColorConfig.cs`

```cs
using Moryx.ControlSystem.Setups;

namespace PencilFactory.ControlSystem.SetupTriggers;

public class ProvideColorConfig : SetupTriggerConfig
{
    public override string PluginName => nameof(ProvideColorTrigger);
}
```

### Trigger

File: `ProvideColorTrigger.cs`

Remember two methods:

1. `Evaluate`: Do we need setup? Which capabilities are missing?
2. `CreateSteps`: Which task do we create?

`Execution = BeforeProduction` means this trigger runs before production.
Cleanup with `AfterProduction` follows in the next chapter.

```cs
[ExpectedConfig(typeof(ProvideColorConfig))]
[Plugin(LifeCycle.Transient, typeof(ISetupTrigger), Name = nameof(ProvideColorTrigger))]
public class ProvideColorTrigger : SetupTriggerBase<ProvideColorConfig>
{
    public override SetupExecution Execution => SetupExecution.BeforeProduction;

    public override SetupEvaluation Evaluate(IProductRecipe recipe)
    {
        if (recipe.Product is not GraphitePencilType product)
            return false;

        return SetupEvaluation.Provide(
            new ColorizingCapabilities { Color = product.Color },
            SetupClassification.Manual);
    }

    public override IReadOnlyList<IWorkplanStep> CreateSteps(IProductRecipe recipe)
    {
        var product = (GraphitePencilType)recipe.Product;

        return
        [
            new ColorChangeTask
            {
                Parameters = new ColorChangeParameters
                {
                    Color = product.Color,
                    Instructions =
                    [
                        new VisualInstruction
                        {
                            Type = InstructionContentType.Text,
                            Content = $"Please change the colorizing cell to color '{product.Color}'."
                        }
                    ]
                }
            }
        ];
    }
}
```

### Activate the config (Command Center)

In the Command Center open the **SetupProvider** module, **Configuration** tab,
**SetupTriggers** section. Select type `ProvideColorConfig`, add it with Plus,
then Save and Restart. Don't forget to reincarnate the module.

![SetupProvider with ProvideColorConfig activated](./chapter-05/setup-provider-provide-color.png)

After you change configs in the Command Center, config files are created under
`src/PencilFactory.App/Config/`.

## Test setup

Chapters 2-3 used **one ColorizingCell per color**, each with a **Driver** only.
For setup you switch to **one** ColorizingCell that can be reconfigured.

1. Delete the extra ColorizingCells (keep one, e.g. `ColorizingCell_Green`).
2. Rename it to `ColorizingCell` if you like. Set **Color = Green**.
3. Keep **one** `SimulatedColorizingDriver` linked as **Driver** (remove unused drivers).
4. **New in this chapter:** also link **VisualInstructor** (the same one as Assembling from chapter 1) on this ColorizingCell.

Until chapter 4 the ColorizingCell had **no** VisualInstructor. Setup (ColorChange) is a
**manual** confirmation, so the cell now needs **both** references: Without the Instructor, a Brown order fails immediately (NullReferenceException) and no
worker instruction appears. Green can still work if the cell color already matches.

![Single ColorizingCell with Driver and Instructor linked](./chapter-05/single-colorizing-cell-driver-instructor.png)

That forces a changeover: one cell, default Green, Brown order. Create a Brown order and start production.

![ColorChange setup instruction to switch to Brown](./chapter-05/color-change-setup-instruction.png)

In worker assistance the instruction to switch to Brown appears.
After Success, under **Resources** the ColorizingCell **Color** is **Brown**.

![ColorizingCell Color property set to Brown after setup](./chapter-05/colorizing-cell-color-brown.png)

Then production runs as before (Assembling, Colorizing, Testing).

### Control test

1. ColorizingCell.Color already set to Brown
2. Start an order for Brown
3. No ColorChange in worker assistance. Production steps start directly.

### Check your progress

* One ColorizingCell remains (Color starts as Green). VisualInstructor **and** Driver linked
* Brown order: Worker Support shows ColorChange to Brown **before** Assembling / Colorizing / Testing
* After Success on ColorChange, Resources UI shows ColorizingCell.Color = Brown, then production runs
* Control test: cell already Brown -> no setup instruction for a Brown order

## Tip: Simulation success rate

When you produce larger orders, the simulated drivers often fail steps on purpose
(scrap). By default the success rate is often around **50%**, so roughly every
second simulated result can be Failed.

For smoother training runs, open the **Simulation** settings and raise the **Success rate** (e.g. from `50` to `90`). To do this, you need to open the configuration tab in the **MachineSimulator** module within the **CommandCenter**. Then most simulated Colorizing/Testing steps succeed. 

![Simulation success rate](./chapter-05/simulation-success-rate.png)

## Checklist

* [ ] Ran `moryx add step ColorChange` and deleted unused resource/capabilities files
* [ ] `ColorizingCapabilities`: nullable `Color`, `null` means any cell
* [ ] `ColorChangeActivity` as setup with `ColorizingCapabilities()` without Color
* [ ] Results Success / TechnicalError / Failed. Parameters with `Color`
* [ ] ColorizingCell: **VisualInstructor** and **Driver** both linked in the Resources UI
* [ ] ColorizingCell: setup and production sessions, ColorChange in StartActivity / SequenceCompleted
* [ ] `ProvideColorTrigger` created and activated in SetupProvider
* [ ] With one ColorizingCell (Green) tested a Brown order: setup first, then production
* [ ] (Optional) Simulation success rate increased (e.g. to 90)

## Summary

Setup prepares the cell **before** production. The trigger says which color is needed. SetupProvider starts ColorChange only if the cell does not already provide it. After a successful ColorChange, the cell's Color (and capabilities) match the product so Colorizing can run.

## Reflect

1. Why is ColorChange a setup job, not a normal step in the pencil workplan?
2. Why must ColorChange accept any ColorizingCell (`Color` null), not only the target paint color?
3. Why can `Evaluate` name the needed color and still produce no ColorChange job?

## Practice

With **one** ColorizingCell, first write down what you expect for each case, then run it and check:

1. Cell Green, product Brown. What happens before Assembling?
2. Cell already Brown, product Brown. What happens before Assembling?
3. A needed ColorChange, Worker chooses **Failed**. What is `Color` on the cell afterward?

When you are done, set Color back to **Green** so the next chapter starts from the same baseline.

<details>
<summary>Hint</summary>

Compare cell `Color` with the product color before each run. For Failed: only one branch in `ColorChangeCompleted` writes `Color`.

</details>

## Check your reasoning

<details>
<summary>Compare your answers after attempting the questions and practice</summary>

1. Setup changes the **machine/cell** before pencils are made. It is not one step per pencil in the production workplan.
2. If ColorChange required Brown already, the Green cell could never run the changeover. Null = any colorizing cell.
3. The trigger names the **target** capabilities. SetupProvider checks whether a cell already provides them and creates ColorChange only when needed. So Evaluate alone does not mean a setup job always appears.

**Practice feedback:** (1) ColorChange, then production. (2) No ColorChange, production starts. (3) Color stays Green (or whatever it was). Only Success updates it.

</details>

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Brown order stalls with no ColorChange | Trigger not activated in SetupProvider, app not restarted after config, Evaluate never returns true |
| ColorChange never appears in Worker Support | Setup session / VisualInstructor not linked, ColorChange not handled in cell StartActivity |
| Setup runs but Color stays Green | ColorChange does not update cell Color / Capabilities on Success |
| ColorChange appears although cell color already matches | Confirm Resources shows the same Color as the product. If yes: in `ColorChangeCompleted`, assign via the `Color` **property** (not only `_color`) so Capabilities update. Don't forget to save and restart after fixing |
| Production fails often after setup | Simulation success rate low. See tip above |

See also [Troubleshooting](troubleshooting.md) and [Help](README.md#help).

> [Table of contents](README.md) | [Previous](chapter-04-testing.md) | [Next](chapter-06-cleanup.md)
