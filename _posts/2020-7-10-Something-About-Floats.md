---
layout: post
title: Something I've always wondered about floats
---

## More Bits Is More Good

In 1996 Nintendo released the N64. I was 6 years old at the time. I had no idea what a "bit" was, but the N64 has 64 of them and that was more than any previous game system. 64 bits was the future. Apparently the old Sony Playstation, which came out 3 years earlier only had 32 bits. This obviously meant the N64 was better, right?

Nintendo certainly used their "bit advantage" in their marketing. Forbes published an article in 1997 comparing the N64 and the Playstation, writing:
> Nintendo has a better chip, 64-bits versus 32-bits. Do consumers understand? Nope, but 64 is twice as much as 32, hence "whiter whites, brighter brights." Advantage: Nintendo.
 https://www.forbes.com/1997/09/19/feat.html#39e0486a7dd9

In 2003, AMD released the Athlon 64, the first mass-market consumer 64-bit CPU.
In 2009, Microsoft released Windows 7, and within a year [almost half of Windows PCs worldwide were running a 64-bit operating system](https://www.zdnet.com/article/windows-7-boosts-64-bit-adoption/).

By 2020, more than 90% of all PCs run [64 bit operating systems on 64 bit hardware](https://techtalk.pcmatic.com/64-bit-operating-systems/)

## Nowadays...

It would be a reasonable assumption that by now, flagship game engines would be using 64-bit floating points for basic game-world position, rotation, and scale. But you would be wrong!

Unreal Engine 4 uses [32 bit transforms](https://docs.unrealengine.com/en-US/API/Runtime/Core/Math/FTransform/index.html)  
Unity 2020 uses [32 bit transforms](https://docs.unity3d.com/ScriptReference/Transform.html)  
Amazon Lumberyard uses [32 bit transforms](https://docs.aws.amazon.com/lumberyard/latest/apireference/class_a_z_1_1_transform_interface.html#a7a08e0aa6935f4b6d4296f8e9fe9c9f0)

Why is this? I won't dig too deeply into the _why_ in this post, but generally it boils down to a few things:
* Legacy compatibility
* Performance of arithmetic operations
* Memory efficiency
* Network transfer efficiency

32 bit is a compromise between cost and realism. Through experimentation, people have generally decided that 16 bit is not enough precision to convincingly represent a 3D world, but 32 is. 64 is better but the costs are not worth the benefit. Large, open-world games quickly run into challenges related to 32-bit floating point precision, but they can be worked with clever tricks like "Origin Shifting"

## Precision

Origin shifting is a trick that relies on a property of floating point numbers, which is that the precision of numbers you can represent with a float is higher the closer you are the origin. I don't remember where I originally read it, but a full _half_ of the possible values you can represent with a floating point number lie within the range [-1, 1]

That's right, of the 4,294,967,296 distinct values you can represent with a float, fully half (2,147,483,648) are between -1 and 1. The entire rest of the number line from -Infinity to Infinity have to get by on half of the possible values. Furthermore, that is split evenly between positive and negative numbers, such that positive numbers greater than 1 only get a quarter, or 1,073,741,824 possible values.

### Pigeonholes

This immediately gets my brain thinking about the consequences. Take for example C#'s [Single.MaxValue](https://docs.microsoft.com/en-us/dotnet/api/system.single.maxvalue?view=netcore-3.1) This number is significantly larger than 1 billion, but it, and presumably many numbers smaller than it have to share bit-space with all the rest of the numbers greater than 1. With numbers like these in the mix, how many values can possible be reserved for areas like... between 500 and 501? What about between 1000 and 1001? Between 50,000 and 50,0001?

I've always wondered about this but could never really find a good explanation for how tightly distributed floats are. How quickly do they become really inaccurate? At what point can we no longer represent 1/10th of the difference between two integers? 1/2? At what point are entire whole numbers sacrificed to the pigeonhole principle?

I've spent a lot of time with Unity over the past couple of years, and I always particularly wondered what this means for a property like Unity's [Time.time](https://docs.unity3d.com/ScriptReference/Time-time.html). It's a 32-bit value that represents the time elapsed since the start of the game. Given the above information, we know that 3/4 of the precision of this value are thrown out after the first second of the game. How quickly does this number become completely useless?

## The Experiment

I set out to find out. I know the answers to these questions can be answered by examining the [IEEE-754](https://en.wikipedia.org/wiki/IEEE_754) and doing some math. But first of all, I'm not that smart, and second, I'd rather get my hands dirty.

### First Pass - Increment By One

I wrote some quick Java code to check how many float values exist between two whole numbers:

```java
public static PrecisionPair numberOfValues(float from) {
    float cursor = from;
    float target = from + 1;
    long iterations = 0;
    while (cursor < target) {
        cursor = Math.nextUp(cursor);
        iterations++;
    }

    return new PrecisionPair(from, target, iterations);
}
```
I also wrote a quick [harness](https://github.com/jschiff/BlogExperiments/blob/master/src/com/jschiff/math/fpprecision/FloatingPointPrecision.java) for testing an arbitrary set of whole numbers and tested the first 10 just to see what I was working with. This was the result:
```
Low     High    Values In Between
0       1       1065353216
1       2       8388608
2       3       4194304
3       4       4194304
4       5       2097152
5       6       2097152
6       7       2097152
7       8       2097152
8       9       1048576
9       10      1048576
10      11      1048576
```

A couple of things we can take away from this right away:
1. As we expected, the space between 0 and 1 contains an enormous number of possible values
1. The number of values drops quickly after that
1. After An anomaly between the ranges [0, 1] and [1, 2] there is a consistent pattern
1. The number of values changes seems to stay the same for consecutive numbers until we reach a power of 2
1. Every power of 2 seems to cut the number of possible values between two whole numbers in half

### Second Pass - Doubling

Given point #2, it stands to reason that we can skip any number that is not a power of 2 in our analysis and still be able to see the trend without missing anything. So I made a small change to my test harness. Rather than incrementing our "low" value by [one](https://github.com/jschiff/BlogExperiments/blob/ff748930087d92782eebae7d02887d6b1d8a3b5d/src/com/jschiff/math/fpprecision/FloatingPointPrecision.java#L45), we can now [double it](https://github.com/jschiff/BlogExperiments/blob/ff748930087d92782eebae7d02887d6b1d8a3b5d/src/com/jschiff/math/fpprecision/FloatingPointPrecision2.java#L45). Here are the results from that experiment, up to 512:

```
Low     High    Values In Between
0       1       1065353216
1       2       8388608
2       3       4194304
4       5       2097152
8       9       1048576
16      17      524288
32      33      262144
64      65      131072
128     129     65536
256     257     32768
512     513     16384
```

This is nice! Eventually though, something very strange happens in this series:

```
Low         High        Values In Between
1048576     1048577     8
2097152     2097153     4
4194304     4194305     2
8388608     8388609     1
16777216    16777216    0
33554432    33554432    0
67108864    67108864    0
```

Our code is surely doing something wrong. Why are Low and High showing as the same number? Let's look back at our code:

```java
public static PrecisionPair numberOfValues(float from) {
    float cursor = from;
    float target = from + 1;
    long iterations = 0;
    while (cursor < target) {
        cursor = Math.nextUp(cursor);
        iterations++;
    }

    return new PrecisionPair(from, target, iterations);
}
```

We know that `iterations` is returning as `0`. That must mean we're never entering our while loop. Why wouldn't we enter our while loop? The only explanation is that `cursor` is never less than `target`. How is this possible? Remember, we're running this code right before the while loop:

```java
float cursor = from;
float target = from + 1;
```

The amazing implication here is that at a certain value for a floating point number `x`: `x + 1 <= x`. In fact what is happening is that the precision of the floating point number around the value `16777216` has become so sparse that it is impossible to represent the next whole number, `16777217` accurately using a 32 bit float. So in float world, `16777216 + 1 = 16777216`.

### Third Pass: Math.nextUp()

So let's make a small change to our code to keep things sane. Rather than simply adding 1, let's try to add 1, and if that hasn't done anything, let's call `Math.nextUp` instead and see what the actual next possible number is.
```java
public static PrecisionPair2 numberOfValuesWithFix(float from) {
    float cursor = from;
    float target = from + 1;

    // This happens if the resolution of the floating point is too fuzzy to represent from + 1
    if (target == from) {
        target = Math.nextUp(from);
    }

    long iterations = 0;
    while (cursor < target) {
        cursor = Math.nextUp(cursor);
        iterations++;
    }

    return new PrecisionPair2(from, target, iterations);
}
```

I've also made a `PrecisionPair2` result type which simply has a [slightly different toString() method](https://github.com/jschiff/BlogExperiments/blob/ff748930087d92782eebae7d02887d6b1d8a3b5d/src/com/jschiff/math/fpprecision/PrecisionPair2.java#L20). This will allow us to also print the difference in magnitude between the `high` and `low` numbers.

This leads us to this result:
<a name="chart"></a>
```
Low         High        Values In Between   Distance Between Values
0           1           1065353216          1
1           2           8388608             1
2           3           4194304             1
4           5           2097152             1
8           9           1048576             1
16          17          524288              1
32          33          262144              1
64          65          131072              1
128         129         65536               1
256         257         32768               1
512         513         16384               1
1024        1025        8192                1
2048        2049        4096                1
4096        4097        2048                1
8192        8193        1024                1
16384       16385       512                 1
32768       32769       256                 1
65536       65537       128                 1
131072      131073      64                  1
262144      262145      32                  1
524288      524289      16                  1
1048576     1048577     8                   1
2097152     2097153     4                   1
4194304     4194305     2                   1
8388608     8388609     1                   1
16777216    16777218    1                   2
33554432    33554436    1                   4
67108864    67108872    1                   8
134217728   134217744   1                   16
268435456   268435488   1                   32
536870912   536870976   1                   64
1073741824  1073741952  1                   128
2147483648  2147483904  1                   256
4294967296  4294967808  1                   512
```
There is an interesting turnaround at value `16777216` where we go from talking about "How many float values are there between two whole numbers?" to "How many whole numbers are there between two floats?". 

### So what's up with the range [0, 1)?

Why is it not exactly twice as large as the range from [1, 2]? Well remember, the actual ranges covered are powers of two. [2, 4) all have the same number of values, as do [4, 8). So in reality, the range [0, 1) contains many different powers of two that are less than 1. [.5, 1) is one such range, as is [.25, .5). I will leave breaking this range down to smaller powers of two as an exercise for the reader.

## So what does this all mean?##

In general? Not a lot, as long as you keep the action of your game within reasonable bounds, such as [-1000, 1000] you still get pretty decent precision. Roughly 1000 subdivisions per unit. This is probably good enough for most games. Consider that the typical high-end monitor on the market today has about 4k resolution per line. At coordinates about 1000 away from the origin, there are still about 1000 subdivisions of each whole number. So let's say you have a game object around that area in the world, and a camera which is viewing exactly one game unit. Then a user with a 4K monitor may notice an object being misplaced by about +/- 2 pixels in that situation. That's an extremely unusual situation.

### Unity Pitfalls

I'm going to focus on Unity here since I have the most experience with it. I think there are some common pitfalls in which Unity games may fall into a trap where they are extremely sensitive to floating point precision error.

#### Time.time

[Time.time](https://docs.unity3d.com/ScriptReference/Time-time.html) is Unity's go-to, default value for "How long has the game been running, in seconds". It is a 32 bit `float` value. So right off the bat, you know, based on our analysis above, that 3/4s of the precision of this value is thrown away after the first second of the game running.

Let's say you make a racing game. A cursory googling of ["racing games"](https://lmgtfy.com/?q=racing+games&t=i) shows image after image of interfaces showing millisecond-resolution lap timers. According our [chart](#chart), after about 16,000 seconds of gameplay, `Time.time` can no longer accurately represent the current game time to a millisecond of precision. 16,000 seconds is about 4.5 hours. It isn't healthy, but a 5 hour gaming session isn't exactly unheard of. So you should definitely not be using Time.time to calculate your current race time.

Another common pitfall of `Time.time` is to use it for Lerping. Let's say you want to Lerp an object from point A to point B. So you write this coroutine:
```csharp
IEnumerator LerpObject(Transform t, Vector3 destination, float duration) {
  Vector3 origin = t.position;
  float startTime = Time.time;
  float endTime = startTime + duration;
  while (Time.time < endTime) {
    float t = (Time.time - startTime) / duration;
    t.position = Vector3.Lerp(origin, destination, t);
    yield return null;
  }
  
  t.position = destination;
} 
```

This looks fine at first. Then a player leaves your game on overnight to get an achievement, and `Time.time` only has 128 subdivisions per second. Suddenly the movement looks jerky on your player's 144hz monitor because the refresh rate of the game is beyond the resolution of your Lerp.

To be fair, Unity warns against using [Time.time] for such purposes in the manual:

> Regular (per frame) calls should be avoided: Time.time is intended to supply the length of time the application has been running for, and not the time per frame.

But that doesn't stop beginners from doing so anyway!

#### Vector3

If the action of your game extends past about 1024 units away from the origin, the precision of object positions can really start to suffer.  In Unity, the Physics engine considers one Unity unit to be one meter. Imagine a flight simulator game where you are flying an airplane traveling at 500 miles per hour, or a game like [Kerbal Space Program](https://www.kerbalspaceprogram.com/) (which was written in Unity!) where you can travel at orbital velocities of thousands of meters per second. You can _very quickly_ get outside of the reasonable range of precision. It will also wreak havoc on your camera and cause unsightly jittering. Any kind of large/open world game must therefore use a [floating origin](https://wiki.unity3d.com/index.php/Floating_Origin) technique to remain stable over large distances.
