---
title: "Fake test dependency"
aliases: ["/tutorials/automated-unit-testing-basics/3-setup-techniques/3-fake-test-dependency/"]
date: 2019-02-22T10:25:49Z
weight: 3
draft: false
---

It is very common that the SUT depends on something else. The dependency could be anything like another function, object or external system and we may not be able to control their behavior all the time. A good unit test should not be affected by the changes in the dependencies of SUT. 

So, a common technique is to replace the dependencies with fake versions that we can control.

{{% img "images/fake-dependency.png" "" 350 %}}

For example, consider a function that takes age of the passenger and prints whether he/she is eligible for senior citizen discount when promotion is available.

The function which tells whether the promotion is available or not is defined in another module and we don't have control over its behavior

```
module promotions
    function isPromotionsAvailable():
        .
        .
        .
```

```
module discounts

function isDiscountAvailable(age):
    if promotions.isPromotionAvailable() && age > 60:
        print("discount available")
    else:
        print("discount not available")
```

However, in order to test all the branches and conditions of `isDiscountAvailable()` function, we need to be able to control the result of `isPromotionAvailable()` function. So, we first create fake version of `isPromotionAvailable()` function in `fakePromotions` module with an option for us to control the result

```
module fakePromotions
    offerPromotion = true

    function isPromotionAvailable():
        return offerPromotion
```
then replace the promotion with fakePromotion inside the test

```
function test_isDiscountAvailable():
    // arrange
    promotions  = fakePromotions  //replace promotions module with fakePromotions module
    promotions.offerPromotion = true
    age = 65

    // act
    result = discounts.isDiscountAvailable(age)

```

Depending on the language and framework we use, the way we replace the real dependency with fake one will be different.