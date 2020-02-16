---
layout: post
title:  "Case insensitive comparisons between an enum and string in C#"
date:   2020-02-16 19:30:00 +0000
categories: csharp
---

A quick exploration into the fastest way to do a case insensitive comparison between a `string` and an `enum` in C#

## TL:DR

```csharp
MyEnum.ToString().Equals(MyString, StringComparison.OrdinalIgnoreCase);
```

## Background

One rainy day in the office a small bug cropped up - calls to one of our services would fail if one of the properties was *lowercase* instead of *Glorious Proper Case*.

Long story short, a `string` was being compared with an `enum` and was not handling case insensitivity. 

The initial code looked like the following:

```csharp
// Bad - sensitive
var result = MyList.Single(x => x.MyEnum.ToString() == MyString);
```

And the following was put in place:

```csharp
// Good - insensitive 
var result = MyList.Single(x => x.MyEnum.ToString().Equals(MyString, StringComparison.OrdinalIgnoreCase));
```

Fantastic, **problem solved!**

Right?

## The Connundrum

During the code review for the wonderful, one line change, a question was raised as to whether it was more efficient to parse the string as the type of the `enum` and then perform the comparison.

This also raised the question of whether the `.ToString()` output of the `enum` could just be converted to lowercase (`.ToLower()`) and compared with the lowercase version of the input string.

And let's not forget, there is also the option of using `string.Compare()` instead of `equals` when comparing the two.

So, of the four comparison methods, which do you think would  be the fastest?

```csharp
// 1.
MyEnum.ToString().Equals(MyString, StringComparison.OrdinalIgnoreCase);

// 2.
MyEnum.ToString().ToLower().Equals(MyString.ToLower());

// 3. (the third parameter in Enum.Parse is ignoreCase)
MyEnum.Equals(Enum.Parse(typeof(TestEnum), MyString, true));

// 4. 
string.Compare(MyEnum.ToString(), MyString, StringComparison.OrdinalIgnoreCase);
```

## Benchmarking

Using the fantastic [BenchmarkDotNet](https://benchmarkdotnet.org) it was easy to throw together a quick benchmark to see which of the four were the fastest:

```csharp
using BenchmarkDotNet.Attributes;
using System;

namespace Experiments
{
    public class ComparingStringsAndEnums
    {
        public enum TestEnum { Foo }
        
        [Params("Foo", "foo")]
        public string MyString { get; set; }
        
        [Params(TestEnum.Foo)]
        public TestEnum MyEnum { get; set; }

        [Benchmark]
        public void ToStringComparison() => MyEnum.ToString().Equals(MyString, StringComparison.OrdinalIgnoreCase);

        [Benchmark]
        public void CompareComparison() => string.Compare(MyEnum.ToString(), MyString, StringComparison.OrdinalIgnoreCase);
        
        [Benchmark]
        public void LowercaseComparison() => MyEnum.ToString().ToLower().Equals(MyString.ToLower());

        [Benchmark]
        public void CastComparison() => MyEnum.Equals(Enum.Parse(typeof(TestEnum), MyString, true));
    }
}
```

The `Params` attribute on the properties tells **BenchmarkDotNet** to perform tests for each of the available options. In total there would be eight benchmarks to run - each comparison, with both *Proper Case* and *lowercase*.

**BenrchmarkDotNet** kindly outputs the results in `.html`, `.csv` and `.md` files for easy formatting - so let's get to the results!

## The Results

Did you guess correctly? 

|              Method | MyString | MyEnum |      Mean |    Error |   StdDev |
|-------------------- |--------- |------- |----------:|---------:|---------:|
|  **ToStringComparison** |      **Foo** |    **Foo** |  **38.63 ns** | **0.340 ns** | **0.302 ns** |
|   CompareComparison |      Foo |    Foo |  42.36 ns | 0.303 ns | 0.283 ns |
| LowercaseComparison |      Foo |    Foo |  96.75 ns | 0.798 ns | 0.747 ns |
|      CastComparison |      Foo |    Foo | 144.05 ns | 2.391 ns | 2.236 ns |
|  **ToStringComparison** |      **foo** |    **Foo** |  **38.10 ns** | **0.720 ns** | **0.673 ns** |
|   CompareComparison |      foo |    Foo |  45.99 ns | 0.655 ns | 0.613 ns |
| LowercaseComparison |      foo |    Foo |  86.28 ns | 0.887 ns | 0.830 ns |
|      CastComparison |      foo |    Foo | 143.58 ns | 2.418 ns | 2.262 ns |

In the end, it seems the original fix was the still fastest; though at least now we can say that with confidence.

Perhaps in the future we'll look into **why** it's the fastest, but that's for another day...