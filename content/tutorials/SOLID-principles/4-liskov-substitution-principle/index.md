---
title: "Liskov Substitution Principle"
date: 2019-09-18T00:00:00Z
weight: 4
draft: false
---

***
{{% colorbtn href="https://youtu.be/StT2I0mwjus?t=731" icon="fab fa-youtube" color="red"%}} Watch video {{% /colorbtn%}}
***

Liskov Substitution Principle(LSP) is about substitution of one object for another without affecting the program that uses those objects.

[Barbara Liskov](https://en.wikipedia.org/wiki/Barbara_Liskov) introduced this principle and she says

> What is wanted here is something like the following substitution property: If for each object o1 of type S there is an object o2 of type T such that for all programs P defined in terms of T, the behaviour of P is unchanged when o1 is substituted for o2 then S is a subtype of T.

Let's understand this with an example. 

Suppose we have a class `MusicPlayer` which allows users to play music

``` csharp
public class MusicPlayer
{
    public virtual void Play()
    {
        // play song
    }
}
```

We have `TrialMusicPlayer` which is inherited from `MusicPlayer` class

``` csharp
public class TrialMusicPlayer : MusicPlayer
{
    public override void Play()
    {
        if(_songsPlayedToday >= 5)
        {
            throw new Exception("Only 5 songs are allowed in a day.");
        }

        // play song
    }
}
```

`TrialMusicPlayer` class overrides `Play()` method to allow playing only 5 songs a day.

We have another class `PlayerWidget`.

``` csharp
public class PlayerWidget
{
    private readonly MusicPlayer _player;
    public PlayerWidget(MusicPlayer player)
    {
        _player = player;
    }

    public void PlayButtonPressed()
    {
        _player.Play();
    }
}
```
 This class represents one of the user interfaces that user can use to control the music player and it simply delegates user requests to the `MusicPlayer`.

{{% img "images/LSP.png" "" 250 %}}


The essence of Liskov Substitution Principle is that we should be able to use objects of derived type in place of objects of base type without altering the behaviour of the program that is using those objects, only then we can call the derived type as **subtype**.

In this example, `TrialMusicPlayer` is not qualified to be subtype of `MusicPlayer` because, we cannot substitute the object of `TrialMusicPlayer` for the object of `MusicPlayer` without changing (or breaking) `PlayerWidget`. This is because `PlayerWidget` is not built to handle the new exception thrown by `TrialMusicPlayer` when more than 5 songs are played.

There are many subtle ways in which we can break substitutability of objects. 

The following are the general rules to follow to avoid breaking substitutability

#### No new exception
No new exception from any method in the subtype. However, we can throw new exception which is subtype of the exception thrown by base class. This is because the exception handler of a type will handle the exceptions of derived exception type as well.

#### Covariance of return type and Contravariance of arguments in subtype methods

Let's first understand what **covariance** and **contravariance** mean. 

Covariance and Contravariance come into picture when we are dealing with complex type with one or more independent type components in them. Generics is one such type so we use custom generic type `ISpecialList<T>` for our example. 

Let's say we have a base class `Person`. We have another class `Employee` which is derived from `Person`. 

In general, if can assign object of `Employee` to type `Person`, but assigning object of `Person` to type `Employee` is not allowed.

```csharp
    Person p = new Employee(); // OK
    Employee e = new Person(); // Compile time error
```

This is not true by default for generic types like `ISpecialList<T>`. We cannot assign object of `ISpecialList<Employee>` to type `ISpecialList<Person>` or vice versa.

However, we can allow these assignments to work using _variance_.

If we allow assigning the object of `ISpecialList<Employee>` to type `ISpecialList<Person>` then we say `ISpecialList<T>` is **covariant**

If we allow assigning the object of `ISpecialList<Person>` to type `ISpecialList<Employee>` then we say `ISpecialList<T>` is **contravariant**

In order to avoid breaking substitutability, the method arguments must be contravariant and return type must be covariant in subtype.

{{% notice tip %}}
In C#, you can create covariant interface using `out` keyword and contravariant interface using `in` keyword. For example,
`ISpecialList<out T>` is covariant and `ISpecialList<in T>` is contravariant.
{{% /notice%}}

#### No strengthening of Preconditions
Preconditions are any validation that we perform before executing a method and it should not be strengthened in a subtype

#### No weakening Postconditions
Postconditions are any validation that we perform after the method is executed and it should not be weakened in a subtype

#### Preserve Invariants
Invariant refers to properties or states that are preserved when the method is executed. Any such invariants of supertype must be preserved in a subtype

#### History constraint
Subtype should not introduce a new method that allows mutating state that is immutable in supertype

### References

1. https://en.wikipedia.org/wiki/Liskov_substitution_principle

2. https://www.hpl.hp.com/techreports/90/HPL-90-121.pdf

3. [Covariance and Contravariance - Wikipedia](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science))