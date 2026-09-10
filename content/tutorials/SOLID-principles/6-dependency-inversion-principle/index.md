---
title: "Dependency Inversion Principle"
date: 2019-09-18T00:00:00Z
weight: 6
draft: false
---

***
{{% colorbtn href="https://youtu.be/StT2I0mwjus?t=1243" icon="fab fa-youtube" color="red"%}} Watch video {{% /colorbtn%}}
***

Dependency Inversion Principle(DIP) is introduced by [Robert C. Martin](https://en.wikipedia.org/wiki/Robert_C._Martin) and his definition is

> A. High-level modules should not depend on low-level modules. Both should depend on abstractions.

> B. Abstractions should not depend upon details. Details should depend upon abstractions.

Let's look at an example of hypothetical mortgage application system. 

We have higher level `Mortgage` module with `Application` class and `BureauData` module at the next level with `BureauDataProvider` class and at lower level we have `Utility` module with `DataCleaner` class

{{% img "images/DIP_violation.png" "" 350 %}}

``` csharp
using BureauData;

namespace Mortgage
{
    public class Application
    {
        public Application(BureauDataProvider dataProvider)
        {

        }
    }
}
```

``` csharp
using Utility;

namespace BureauData
{
    public class BureauDataProvider
    {
        public BureauDataProvider(DataCleaner dataCleaner)
        {

        }
    }
}
```

``` csharp
namespace Utility
{
    public class DataCleaner
    {

    }
}
```

Here the higher level `Application` class directly depends on next level `BureauDataProvider` class and it directly depends on `DataCleaner` class in the lower level. `Application` class indirectly depends on `DataCleaner`. Now, changes to `DataCleaner` or `BureauDataProvider` class may directly affect `Application` class.

This is not a good situation. The higher level module should control the lower level module, not the other way around. Otherwise, changes in lower level classes would force changes on higher level classes and their consumers without adding any value.

In addition to this, when higher level classes directly depend on lower level classes, it is very difficult to reuse higher level classes in different context.

We can use Dependency Inversion Principle to address this issue. We first need to define the abstractions (interfaces) that higher level modules should work with then make the lower level modules implement them.

{{% img "images/DIP.png" "" 400 %}}

Here is the revised example

``` csharp
namespace Mortgage
{
    public interface IApplicationBureauDataProvider
    {

    }

    public class Application
    {
        public Application(IApplicationBureauDataProvider dataProvider)
        {

        }
    }
}
```

``` csharp
using Mortgage;

namespace BureauData
{
    public interface IDataCleaner
    {

    }

    public class BureauDataProvider : IApplicationBureauDataProvider
    {
        public BureauDataProvider(IDataCleaner dataCleaner)
        {

        }
    }
}
```

``` csharp
using BureauData;

namespace Utility: IDataCleaner
{
    public class DataCleaner
    {

    }
}
```

Now if we look at the `Application` class in `Mortgage` module, it depends on `IApplicationBureauDataProvider` interface which is defined in `Mortgage` module itself. So, `Application` class depends on the abstraction rather than the concrete implementation detail.

The `BureauDataProvider` class in `BureauData` module implements `IApplicationBureauDataProvider` from `Mortgage` module. So, the detail implementation class implements(depends on) the abstraction forced by higher level module.

Similarly, `BureauDataProvider` depends on `IDataCleaner` interface defined in `BureauData` module and `DataCleaner` class in `Utility` module is implementing `IDataCleaner` interface.


As we can see, the advantage of this design is that the changes to `BureauDataProvider` cannot affect `Application` because `BureauDataProvider` implements `IApplicationBureauDataProvider` and it cannot change the contract. Also, we can easily reuse `Application` class with another bureau data provider as long as the new bureau data provider implements `IApplicationBureauDataProvider`.


{{% notice note %}}
It is not necessary that the interfaces higher level module depend on should be packaged in the same module. It can be in different module as long it is owned by the higher level module (i.e. changed only when needed for higher level module).
{{% /notice %}}

{{% notice tip %}}
This principle is also referred to as **Hollywood Principle**. This is because _"Don't call us, we will call you"_ phrase is used in Hollywood auditions and this is the communication pattern we achieve with Dependency Inversion Principle.
{{% /notice %}}


### References
1. https://en.wikipedia.org/wiki/Dependency_inversion_principle

2. Chapter 11, Agile Principles, Patterns, and Practices in C# by Robert C. Martin; Micah Martin

3. https://en.wiktionary.org/wiki/Hollywood_principle