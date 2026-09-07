# Application Developer Program

In this program, you will build a MORYX application from scratch. You will go through the typical process of an application developer and learn about MORYX concepts and terminology on the way.

You will accompany a pencil manufacturer on its road to the digital factory. Despite using some specialized machines, they don't have any automated processes right now.

## Who this tour is for

This tour addresses application developers that have a basic understanding of software development/programming. While *ideally* you are a **.NET/C#** developer, used to work with **Visual Studio** and experienced in **Object Oriented Programming**, all of that is **not mandatory** to master this.

## Prerequisite

Below is a list of patterns and basic concepts that MORYX is built upon, but you don't need to know them right upfront.

### General/OOP

* [Dependency injection (DI)](https://en.wikipedia.org/wiki/Dependency_injection)
* [SOLID Principles](https://www.c-sharpcorner.com/UploadFile/damubetha/solid-principles-in-C-Sharp)

### C#/.NET

This is a list of more or less 'advanced' topics

* [C# Reflection](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/reflection-and-attributes/)

### Design Patterns

* [Facade](https://en.wikipedia.org/wiki/Facade_pattern)
* [Factory](https://en.wikipedia.org/wiki/Factory_method_pattern#C#)
* [Plugin](https://de.wikipedia.org/wiki/Plugin_(Entwurfsmuster))
* [Proxy](https://en.wikipedia.org/wiki/Proxy_pattern)
* [Repository](https://de.wikipedia.org/wiki/Repository_(Entwurfsmuster))
* [State](https://en.wikipedia.org/wiki/State_pattern)

## Requirements

Before you start, you need the following tools installed on your machine:

* [ ] [Git](https://git-scm.com/)
* [ ] [Visual Studio 2026](https://visualstudio.microsoft.com/downloads/)
* [ ] [.NET 10](https://dotnet.microsoft.com/en-us/download/dotnet/10.0)

## Chapters

### [Chapter 1 - Basics](chapter-01-basics.md)

In this chapter you will create a MORYX application from scratch and digitalize a manual assembling station.

You will learn to:

* Model digital twins from existing 'things' within a factory and products
* Show instructions to workers
* Lay out a production workflow
* Run your first production process

Concepts: Resources, Cells, ProductTypes, ProductInstances, VisualInstructions, Activities, Tasks, Workplans, Orders

### [Chapter 2 - Drivers](chapter-02-drivers.md)

Since so many pencils were sold, the manufacturer decided that only manual cells aren't feasible anymore. So some manual cells are replaced by fully automated ones.

You will learn to:

* Encapsulate hardware communication in a Driver
* Use `IInOutDriver` with simulation

Concepts: Drivers, Protocols, Simulation

### [Chapter 3 - Capabilities](chapter-03-capabilities.md)

The manufacturer decided to have a separate Colorizing Cell for each color in order not to have to change the paint anymore.

You will learn to:

* Express what a cell can do with Capabilities
* Bind product data into activity parameters

Concepts: Capabilities, ParameterBinding

### [Chapter 4 - Testing](chapter-04-testing.md)

Quality complaints are rising, so the manufacturer adds Testing as a third station after Assembling and Colorizing. Like Colorizing, it should run automatically through a driver and simulation.

You will learn to:

* Add a new production step with the same driver pattern as Colorizing
* Wire a simulated Testing driver and extend the workplan

Concepts: Drivers, Simulation, automatic cells

### [Chapter 5 - Setup](chapter-05-setup.md)

Running a Colorizing Cell for each color became expensive. The manufacturer switches back to a single cell and wants it prepared before production whenever the paint does not match the order.

You will learn to:

* Trigger a ColorChange before production when capabilities do not match
* Use SetupTriggers and setup activities on the ColorizingCell

Concepts: Setup, SetupTrigger, ActivityClassification.Setup

### [Chapter 6 - Cleanup](chapter-06-cleanup.md)

After a brown order finishes, the cell should not stay brown forever. The manufacturer wants the line reset to a default color so the next order starts from a known state.

You will learn to:

* Run cleanup after the production job with an AfterProduction trigger
* Reuse the ColorChange activity for setup and cleanup

Concepts: Cleanup, SetupExecution.AfterProduction

### [Chapter 7 - PartLinks](chapter-07-partlinks.md)

Pencils alone are not enough for the shop floor. The manufacturer now sells retail packs: twenty pencils of one type plus a matching carton.

You will learn to:

* Model cartons and retail packs as product types
* Describe the bill of materials with PartLinks and Quantity

Concepts: ProductPartLink, Quantity, product types and instances

### [Chapter 8 - Packing](chapter-08-packing.md)

With the retail pack defined, workers need a packing station. The instruction should tell them how many pencils go into which carton, based on the pack's bill of materials (BOM).

You will learn to:

* Build a Packing cell with worker instructions from PartLinks
* Create a dedicated workplan and recipe for PencilPack orders

Concepts: Packing activity, separate workplan and recipe per product

### [Chapter 9 - Initializer and Importer](chapter-09-initializer-importer.md)

Clicking everything together in the UI worked for learning, but every new machine or training lab starts empty. The manufacturer wants the factory and master data created from code and a Factory Monitor to see the line.

You will learn to:

* Seed resources with a ResourceInitializer
* Import products, workplans, and recipes with a ProductImporter
* Show the line in the Factory Monitor

Concepts: ResourceInitializer, ProductImporter, Factory Monitor

### [Chapter 10 - Module, Facade, Adapter](chapter-10-module-adapter.md)

Orders no longer come only from the Orders UI. An ERP system knows article `PEN-GREEN`, while the factory knows material number `100001`. The manufacturer needs a bridge between those two worlds.

You will learn to:

* Build a server module with a facade for order rules
* Map ERP articles to material numbers with an adapter

Concepts: Server module, Facade, Adapter

### [Chapter 11 - CellSelectors](chapter-11-cell-selectors.md)

One testing station is no longer enough. The manufacturer adds a second automatic tester and a slower manual backup for brown pencils and wants load distributed intelligently.

You will learn to:

* Filter cells with Capabilities and Constraints
* Balance and optimize cell selection under load

Concepts: CellSelector, LoadBalancer, Constraints

### [Chapter 12 - Advanced Topics](chapter-12-advanced-topics.md)

The digital factory is running. Now the manufacturer asks for polish that shows up in almost every project: clearer order assignment, localized texts, operator notifications, richer activity results and visible cell states.

You will learn to:

* Plug in product and recipe assignments
* Localize texts and raise notifications
* Extend activity results and drive a cell state machine

Concepts: Assignments, Localization, Notifications, States

## Help

If you need help, you can ask and find MORYX related questions on Stack Overflow using the tag [moryx](https://stackoverflow.com/questions/tagged/moryx), or you can open issues on GitHub.
