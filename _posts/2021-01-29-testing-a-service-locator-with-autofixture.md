---
title: "Testing a service locator using AutoFixture"
---

Recently I was required to test an annoying piece of legacy code that used a [Service Locator][1] and I was becoming increasingly frustrated testing it.

The code looked something like this:

```csharp
    // program.cs
    ServiceLocator.Register<IFoo>(new Foo());
    ServiceLocator.Register<IBar>(new Bar());
    ServiceLocator.Register<IBaz>(new Baz());


    // baz.cs
    public Baz() 
    {
        this.foo = ServiceLocator.Find<IFoo>();
        this.bar = ServiceLocator.Find<IBar>();
    }
```

In order to improve testability the previous developer overloaded the constructor with dependencies.

```csharp
    // baz.cs

    // constructor #1
    public Baz() : this(ServiceLocator.Find<IFoo>(), ServiceLocator.Find<IBar>()) {}

    // constructor #2
    public Baz(IFoo foo, IBar bar) 
    {
        this.foo = foo;
        this.bar = bar;
    }
```

Recently I've been using AutoFixture a lot more. AutoFixture allows you to abstract a lot of the test setup and scaffolding that you would normally be required to do and it really makes testing a whole lot easier. The problem, however is that by default it uses the empty constructor when building a new test object. I would like it to use the second constructor, this way I don't need to rely on the service locator.

The first solution I tried was using AutoFixture's greedy strategy:

```csharp
    fixture.Customizations.Add(
        new MethodInvoker(
            new GreedyConstructorQuery()));
```

This caused the following error: 

> Ploeh.AutoFixture.ObjectCreationException : AutoFixture was unable to create an instance from System.SByte*, most likely because it has no public constructor, is an abstract or non-public type.

Later I found the correct solution on [stackoverflow][2] by AutoFixture's creator - seems that I wasn't the first person to try this.

```csharp
public class GreedyEngineParts : DefaultEngineParts
{
    public override IEnumerator<ISpecimenBuilder> GetEnumerator()
    {
        var iter = base.GetEnumerator();
        while (iter.MoveNext())
        {
            if (iter.Current is MethodInvoker)
                yield return new MethodInvoker(
                    new CompositeMethodQuery(
                        new GreedyConstructorQuery(),
                        new FactoryMethodQuery()));
            else
                yield return iter.Current;
        }
    }
}

Fixture fixture = new Fixture(new GreedyEngineParts());
```


[1]: https://en.wikipedia.org/wiki/Service_locator_pattern 
[2]: https://stackoverflow.com/a/12425942