---
layout: post
title: Tips, tricks, and good practices for Data-Driven Testing. Part 1.
description: "Data-driven testing can be the best thing after inventing the can opener. But when done improperly can be like cutting yourself with the can. Here are a few tips on how to do it properly. "
modified: 2019-09-03
tags: [dotnet, testing, NUnit, XUnit, DDT, craft, data-driven testing, tests]
series: "Data-Driven Testing"
image:
 feature: data/2019-09-03-data-driven-tests-tips-and-tricks-part1/logo.jpg
---

## Use of inconclusive


## Don't limit yourself to `` there is also `TestFixtureData`

It works in the same way, but can create test fixtures:

```csharp
[TestFixtureSource(typeof(FixtureDataSource), "GetFixtureData")]

public class MyTestFixture
{
    private string eq1;

    public MyTestFixture(string eq1, string eq2, string neq)
    {
        this.eq1 = eq1;
    }

    [Test]
    public void TestEquality()
    {
        Assert.AreEqual(eq1, eq2);
        if (eq1 != null && eq2 != null)
            Assert.AreEqual(eq1.GetHashCode(), eq2.GetHashCode());
    }

    [Test]
    public void Test()
    {
        Assert.AreEqual(eq1, eq2);
        if (eq1 != null && eq2 != null)
            Assert.AreEqual(eq1.GetHashCode(), eq2.GetHashCode());
    }
}


public class FixtureDataSource
{
    public static IEnumerable GetFixtureData
    {
        get
        {
            yield return new TestFixtureData("hello", "hello", "goodbye");
            yield return new TestFixtureData("zip", "zip");
            yield return new TestFixtureData(42, 42, 99);
        }
    }  
}
```