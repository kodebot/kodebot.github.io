---
title: "Open-Closed Principle"
date: 2019-09-18T00:00:00Z
weight: 3
draft: false
---

***
{{% colorbtn href="https://youtu.be/StT2I0mwjus?t=368" icon="fab fa-youtube" color="red"%}} Watch video {{% /colorbtn%}}
***

Open-Closed Principle(OCP) says software must be open for extension but closed for modification. When we first hear this, it might sound like oxymoron but it is not when we understand this principle correctly. 

This principle was first introduced by [Bertrand Meyer](https://en.wikipedia.org/wiki/Bertrand_Meyer) and his definition of this principle is:

> Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification.

Later, [Robert C. Martin](https://en.wikipedia.org/wiki/Robert_C._Martin) provided the following improved definition which is suitable for modern software development

> You should be able to extend the behaviour of a system without having to modify that system.

Let's try to understand this in detail. How can a software be open for extension and closed for modification? After all, if we want to add a new functionality or change an existing functionality, we need to change the code. So what this principle is actually talking about then?

The principle says

1. identify changes that are frequent 
2. keep the parts of system affected by those frequent changes open for extension
3. keep the parts of system not affected by those changes closed for modification

Let's look at an example to make this clear.

We have an `Accountant` class here that calculates tax for given income

``` csharp
public class Accountant
{
    public decimal CalculateTax(decimal income){
        return income * 0.2m;
    }
}

```
We can say, the code in the example above confirms to Open-Closed principle as long as the `CalculateTax()` method doesn't need to be changed frequently. It doesn't need to be changed when the `Accountant` class supports only one country and the tax rules or laws for that country is not changed frequently.

On the other hand, if we want to support tax calculation for other countries then we can "modify" `CalculateTax()` method, like in the example below, to have different calculation per country

``` csharp
public class Accountant
{
    private readonly Country _country;
    
    public Accountant(Country country)
    {
        _country = country;
    }

    public decimal CalculateTax(decimal income)
    {
        switch(_country)
        {
            case Country.UK:
                return income * 0.2m;
            case Country.USA:
                return income * 0.25m
        }
    }
}

```

{{% img "images/OCP_violation_2.png" "" 100 %}}

Every time we add support for new country, the `CalculateTax()` method need to be updated.

Note, in this example, we have introduced a new constructor parameter of type `Country` enum. This represents a country that need to be used when calculating tax. The `CalculateTax()` method is modified to use different tax calculation based on the country that was passed in when `Accountant` class was constructed.

If we know that we are going to add support for more countries frequently, then `Accountant` class is definitely not confirming to Open-Closed Principle.

To confirm to Open-Closed Principle, we need to make the code in `CalculateTax()` method open for extension but keep the `Accountant` class and its contract closed for modification. 

We need to keep `Accountant` class and its contract closed because, we don't want the consumers of `Accountant` class affected whenever a new country is added. That is, if the contract (i.e. public signature) of `Accountant` class is changed then we need to update all the existing consumers or if the `CalculateTax()` method body is changed then we have high risk of breaking existing functionality and some or all of the existing consumers may be affected.

Neither of those scenarios are ideal.

One of the ways of making `Accountant` class confirming to Open-Closed Principle is by using abstraction. We can abstract the tax calculation by creating an interface. The interface will have just one method in this case and that will take the income as input and return tax as a result.

``` csharp
public interface ITaxCalculator
{
    decimal Calculate(decimal income);
}
```

Now we need to change the constructor of `Accountant` class to accept `ITaxCalculator` instead of `Country` enum as parameter.

The new version of `Accountant` class that confirms to Open-Closed Principle will look like the following

```csharp
public class Accountant
{
    private readonly ITaxCalculator _taxCalculator;

    public TaxCalculator(ITaxCalculator taxCalculator)
    {
        _taxCalculator = taxCalculator;
    }

    public decimal CalculateTax(decimal income)
    {
        return _taxCalculator.Calculate(income);
    }
}
```

{{% img "images/OCP.png" "" 250 %}}

We can create concrete versions of this interface for each country implementing tax calculation applicable for each of those countries without changing `Accountant` class. 

Another example for Open-Closed Principle is "plugin" model implementation. Plugins allow extending the functionality of host program with out changing its code.

When confirming to Open-Closed Principle, more often than not, we might need to make some modification but, those modification are usually isolated and trivial. For example, when creating a new class supporting tax calculation for a new country, the `Country` enum need to be updated to include the new country and any logic that we have to map `Country` enum value to tax calculation class need to be updated to include the new ones.

If we look at the essence of the example we have seen so far, it is all about swapping the tax calculation logic at runtime. We don't necessarily need to use interface to achieve this. We can use higher order function (i.e. function taking another function as a parameter), abstract class or anything that allows swapping code at runtime to get the same end result.

Designing a software that confirms to Open-Closed Principle for every possible change that may occur in the future is impossible and trying to do that will only create unnecessary complexity and increase the cost of initial development. 

[Robert C. Martin](https://en.wikipedia.org/wiki/Robert_C._Martin) says 

> Resisting premature abstraction is as important as abstraction itself

So, we need to identify what to design that confirms to Open-Closed Principle and what not to. Sometimes we can identify this by making an educated guess based on our experience or by talking to domain experts but sometimes we need to wait until the first change occurs. 

Sometimes, we may get it wrong and design something that rarely changes but confirms to Open-Closed Principle. so, as you guessed, the challenge is not in applying Open-Closed Principle but identifying what should be designed confirming this principle.

### References
1. https://hackernoon.com/why-the-open-closed-principle-is-the-one-you-need-to-know-but-dont-176f7e4416d

2. https://blog.cleancoder.com/uncle-bob/2014/05/12/TheOpenClosedPrinciple.html

3. Chapter 9 - Agile Principles, Patterns, and Practices in C# by Robert C. Martin and Micah Martin