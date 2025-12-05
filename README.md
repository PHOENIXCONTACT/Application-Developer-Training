# Application Developer Program

In this program, you will build a MORYX application from scratch. You will go
through the typical process of an application developer and learn about MORYX
concepts and terminology on the way.

You will accompany a pencil manufacturer on its road to the digital factory.
Despite using some specialized machines, they don't have any automated
processes right now.

## Who this tour is for

This tour addresses application developers, that have a basic understanding of
software development/programming. While *ideally* you are a **.NET/C#** developer,
used to work with **VisualStudio** and experienced in **Object Orientated
Programming**, all of that is **not mandatory** to master this.

## Prerequisite

Below is a list of patterns and basic concepts that MORYX is built upon, but you
don't need to know right upfront.

### General/OOP

* [Dependency injection (DI)](https://en.wikipedia.org/wiki/Dependency_injection#:~:text=In%20software%20engineering%2C%20dependency%20injection,leading%20to%20loosely%20coupled%20programs.)
* [SOLID Principles](https://www.c-sharpcorner.com/UploadFile/damubetha/solid-principles-in-C-Sharp)

### C\#/.NET

This is a list of more or less 'advanced' topics

* [C# Reflection](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/reflection-and-attributes/)

### Design Patterns

* [Facade](https://en.wikipedia.org/wiki/Facade_pattern#:~:text=The%20facade%20pattern%20(also%20spelled,complex%20underlying%20or%20structural%20code.))
* [Factory](https://en.wikipedia.org/wiki/Factory_method_pattern#C#)
* [Plugin](https://de.wikipedia.org/wiki/Plugin_(Entwurfsmuster))
* [Proxy](https://en.wikipedia.org/wiki/Proxy_pattern)
* [Repository](https://de.wikipedia.org/wiki/Repository_(Entwurfsmuster))
* [State](https://en.wikipedia.org/wiki/State_pattern)

## Requirements

Before you start, you need the following tools installed on your machine:

* [ ] [Git](https://git-scm.com/)
* [ ] [Visual Studio 2022](https://visualstudio.microsoft.com/downloads/) (Any version below wouldn't work, but you can install them in parallel)
* [ ] [.NET 10](https://dotnet.microsoft.com/en-us/download/dotnet/10.0)

## [Chapter 1 - Basics](chapter-1-basics.md)

In this chapter you will build an application to digitalize a manual pencil production line.

While it introduces the basic terminologies and shows how to create a MORYX
application from scratch it enables you to

* Model digital twins from existing 'things' within a factory and products
* Show instructions to workers
* Lay out a production workflow
* Run your first production process

MORYX Terminology and Concepts you will learn

* Resources, Cells
* ProductTypes, ProductInstances​
* VisualInstructions, WorkerSupport
* Activities, Tasks, Capabilities
* Parameters, ParameterBinding
* Workplans
* Orders

## [Chapter 2 - Drivers](chapter-2-drivers.md)

Since so many pencils were sold, the manufacturer decided that only manual cells aren't feasible anymore. So some manual cells are replaced by fully automated ones.

In this chapter you will learn how to

* Communicate with hardware
* Use different hardware, without having to adjust the source code
* Set up a simulated production

Terminology and Concepts you will learn

* Drivers
* Protocols
* Simulation

## [Chapter 3 - Basics II](chapter-3-basics-II.md)

The manufacturer decided to have a separate Colorizing Cell for each color in order not to have to change the paint anymore.

In this chapter you will learn how to

* Find the right Cell depending on a product property

MORYX Terminology and Concepts you will learn

* Capabilities
* ParameterBinding

## Help

If you need help, you can ask and find MORYX related questions on Stack Overflow using the tag [moryx](https://stackoverflow.com/questions/tagged/moryx). There is also a Gitter channel or you can open issues on GitHub.
