# Troubleshooting

Cross-chapter quick reference for common MORYX ADP pitfalls. Prefer the chapter-specific **Troubleshooting** section while you are in that chapter. Use this page when the symptom spans several topics or you are unsure which chapter owns it.

> [Table of contents](README.md) | [Glossary](glossary.md) | [Help](README.md#help)

## Wrong folder for `moryx add`

**Problem:** `moryx add step` / `moryx add module` creates files in the wrong place or fails to find the solution.

**Check:** Your shell's current directory. It must be the **PencilFactory solution root** (the folder that contains the `.sln`), the same folder Visual Studio opened.

**Fix:** `cd` into that solution root, then run `moryx add ...` again. (`moryx new` is only for the first create. `moryx exec post-setup` talks to the running app over HTTP and does not depend on which folder you are in.)

## Missing usings / types do not compile

**Problem:** A type or attribute is red, or the build cannot find it.

**Check:** Missing `using`. Leftover CLI namespaces after `moryx add` (`MyApplication`, `Some`, ...). Missing project or package reference.

**Fix:** On the red name, use Visual Studio Quick Actions (`Ctrl + .`) and add the using it offers. If nothing useful appears, the project usually does not reference the assembly yet: add the ProjectReference or NuGet package the chapter just introduced, then rebuild.

## Databases not created / UI cannot connect

**Problem:** Products or Resources show connection errors. Modules stay down until databases exist.

**Check:** Command Center -> Databases. Did `moryx exec post-setup` run while the app was running?

**Fix:** Create any missing databases, then reincarnate the modules or restart the app ([Chapter 1](chapter-01-basics.md#encountering-database-issues-after-setup-section)). Use Erase + Create only for an intentional clean seed ([Chapter 9](chapter-09-initializer-importer.md)).

## NuGet restore slow or failing

**Problem:** First `F5` hangs on restore, or packages are missing at build.

**Check:** Network / NuGet feed. After you add a package in a chapter, both the project `PackageReference` and `Directory.Packages.props` must list it with matching versions.

**Fix:** Wait for the first restore (it can take minutes). Then rebuild. If restore still fails, clean the solution and rebuild.

## App URL / port

**Problem:** Browser or `moryx exec post-setup` cannot reach the app.

**Check:** The training default is `https://localhost:5000` (`PencilFactory.App` -> `Properties/launchSettings.json` -> `applicationUrl`). Confirm the app is running and that URL opens in the browser.

**Fix:** If you changed the port in `launchSettings.json`, open that URL instead and pass the same URL to post-setup with `--endpoint <URL>`. If the port looks correct but nothing answers, clean and rebuild, then start again.

## EntrySerialize vs DataMember

**Problem:** Property missing in the UI, or visible but lost after restart (or the reverse).

**Check:**

| Want | Need |
| ---- | ---- |
| Visible / editable in UI | `[EntrySerialize]` |
| Persisted in the database | `[DataMember]` |

**Fix:** Use both when the field should appear and survive restarts. On Colorizing `Color`, put `DataMember` on the backing field and `EntrySerialize` on the property so the setter updates capabilities. [Glossary](glossary.md#entryserialize), [Chapter 1](chapter-01-basics.md#quick-experiment-entryserialize-vs-datamember), [Chapter 3](chapter-03-capabilities.md).

## Orders blocked / production does not start

**Problem:** BEGIN does nothing useful. The process never reaches a cell.

**Check:** Default recipe on the product, workplan connections, cell has Instructor and/or Driver, capabilities match, cell publishes `ReadyToWork` / completes activities. Worker Support shows the correct display.

**Fix:** Walk Product -> Recipe -> Workplan -> Activity -> Cell ([Chapter 1](chapter-01-basics.md#orders-blocked--production-does-not-start)). Pack products need their own recipe/workplan ([Chapter 8](chapter-08-packing.md)). Orders from the ERP adapter still need release/start in the Orders UI ([Chapter 10](chapter-10-module-adapter.md)).

## ProcessEngine routing / no matching cell

**Problem:** Order stalls, no resource for an activity, or the wrong cell is chosen.

**Check:** `RequiredCapabilities` vs cell `Capabilities` / `ProvidedBy`, setup trigger when needed, constraints. CellSelector SortOrder, `MaxActiveJobs` / `MaxRunningOperations`.

**Fix:** Use the Processes view to see which cell was chosen. Then [Chapter 3](chapter-03-capabilities.md#troubleshooting), [Chapter 5](chapter-05-setup.md#troubleshooting), or [Chapter 11](chapter-11-cell-selectors.md#troubleshooting).

## Driver not subscribed / change has no effect until restart

**Problem:** Linking or changing a driver in the UI does nothing until a full restart. Ready / ProcessResult ignored.

**Check:** `Driver` setter subscribes/unsubscribes `InputChanged`. Input key names and values match what the simulator raises.

**Fix:** [Chapter 2](chapter-02-drivers.md#troubleshooting), [Chapter 4](chapter-04-testing.md#troubleshooting).

## Simulation stuck

**Problem:** Automatic step never finishes, or the next cycle never starts.

**Check:** Driver linked. `ProcessStart` reset to `false` after `ProcessResult`. Simulation / MachineSimulator present. Success rate not extremely low.

**Fix:** Reset `ProcessStart` after the result ([Chapter 2](chapter-02-drivers.md#troubleshooting)). Optionally raise the MachineSimulator success rate ([Chapter 5](chapter-05-setup.md#tip-simulation-success-rate)). Shorten execution times only when a chapter asks you to ([Chapter 11](chapter-11-cell-selectors.md)).

## Resource missing in Add dialog

**Problem:** Expected cell or driver type does not appear when adding a resource.

**Check:** Resource project is referenced by `PencilFactory.App` and the assembly loaded after rebuild.

**Fix:** Add the ProjectReference, rebuild, restart the app. [Chapter 1](chapter-01-basics.md#i-cannot-find-a-resource-in-the-add-dialog).

## Shared VisualInstructor under parallel orders

**Problem:** Wrong or overlapping Worker Support instructions when several manual stations run in parallel.

**Check:** Several manual cells still share one VisualInstructor while `MaxRunningOperations` / `MaxActiveJobs` > 1.

**Fix:** Give Assembling, Colorizing, Manual Testing and Packing each their own instructor before the parallel test in [Chapter 11](chapter-11-cell-selectors.md#troubleshooting).

## Module / adapter not running

**Problem:** ProductionBridge or the ERP mock adapter is missing or not Running in Command Center.

**Check:** App ProjectReferences. Facade exported, adapter imports it with `[RequiredModuleApi]`. Config saved and module restarted.

**Fix:** [Chapter 10](chapter-10-module-adapter.md#troubleshooting).

## Initializer / importer duplicates or empty Factory Monitor

**Problem:** Duplicate cells or products after re-running the seed. Factory Monitor stays blank.

**Check:** Databases cleared before re-seed. Factory Monitor packages referenced. Initializer invoked and ResourceManager reincarnated.

**Fix:** [Chapter 9](chapter-09-initializer-importer.md#troubleshooting).

## Colorizing `CellState` empty or never changes

**Problem:** In Resources, ColorizingCell shows no useful `CellState`, or it never moves between Idle, Setup and Production while you run ColorChange and Colorizing.

**Check:** Chapter 12 adds `CellState` so you can see the cell mode in the UI. Typical mistakes: `WithAsync` uses a stub base class instead of `ColorizingCellStateBase` or the cell never calls the state methods when setup/production starts or finishes.

**Fix:** Follow the [States section](chapter-12-advanced-topics.md#states-cell-state-machine) and [Chapter 12 Troubleshooting](chapter-12-advanced-topics.md#troubleshooting). If Production flashes by too fast to notice, that is often normal with a short simulator time: keep Resources open on the cell during Colorizing or temporarily raise Colorizing `ExecutionTime` in MachineSimulator.

---

> [Table of contents](README.md) | [Glossary](glossary.md)
