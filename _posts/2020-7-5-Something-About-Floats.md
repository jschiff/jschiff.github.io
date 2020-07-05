---
---
layout: post
title: Something I've always wondered about floats
---

In 1996 Nintendo released the N64. I was 6 years old at the time. I had no idea what a "bit" was, but the N64 has 64 of them and that was more than any previous game system. 64 bits was the future. Apparently the old Sony Playstation, which came out 3 years earlier only had 32 bits. This obviously meant the N64 was better, right?

Nintendo certainly used their "bit advantage" in their marketing. Forbes published an article in 1997 comparing the N64 and the Playstation, writing:
> Nintendo has a better chip, 64-bits versus 32-bits. Do consumers understand? Nope, but 64 is twice as much as 32, hence "whiter whites, brighter brights." Advantage: Nintendo.
 https://www.forbes.com/1997/09/19/feat.html#39e0486a7dd9

In 2003, AMD released the Athlon 64, the first mass-market consumer 64-bit CPU.
In 2009, Microsoft released Windows 7, and within a year [almost half of Windows PCs worldwide were running a 64-bit operating system](https://www.zdnet.com/article/windows-7-boosts-64-bit-adoption/).

By 2020, more than 90% of all PCs run [64 bit operating systems on 64 bit hardware](https://techtalk.pcmatic.com/64-bit-operating-systems/)

It would be a reasonly assumption that by now, flagship game engines would be using 64-bit floating points for basic game-world position, rotation, and scale. But you would be wrong!

Unreal Engine 4 uses [32 bit transforms](https://docs.unrealengine.com/en-US/API/Runtime/Core/Math/FTransform/index.html)
Unity 2020 uses [32 bit transforms](https://docs.unity3d.com/ScriptReference/Transform.html)
Amazon Lumberyard uses [32 bit transforms](https://docs.aws.amazon.com/lumberyard/latest/apireference/class_a_z_1_1_transform_interface.html#a7a08e0aa6935f4b6d4296f8e9fe9c9f0)

Why is this? I won't dig too deeply into the _why_ in this post, but generally it boils down to a few things:
* Legacy compatibility
* Performance of arithmetic operations
* Memory efficiency
* Network transfer efficiency

32 bit is a compromise between cost and realism. Through experimentation, people have generally decided that 16 bit is not enough precision to convincingly represent a 3D world, but 32 is. 64 is better but the costs are not worth the benefit. Large, open-world games quickly run into challenges related to 32-bit floating point precision, but they can be worked with clever tricks like "Origin Shifting"

Origin shifting is a trick that relies on a property of floating point numbers, which is that the precision of numbers you can represent with a float is higher the closer you are the origin. I don't remember where I originally read it, but a full _half_ of the possible values you can represent with a floating point number lie within the range [-1, 1]

That's right, of the 4,294,967,296 distinct values you can represent with a float, fully half (2,147,483,648) are between -1 and 1. The entire rest of the number line from -Infinity to Infinity have to get by on half of the possible values. Furthermore, that is split evenly between positive and negative numbers, such that positive numbers greater than 1 only get a quarter, or 1,073,741,824 possible values.

This immediately gets my brain thinking about the consequences. Take for example C#'s [Single.MaxValue](https://docs.microsoft.com/en-us/dotnet/api/system.single.maxvalue?view=netcore-3.1) This number is significantly larger than 1 billion, but it, and presumably many numbers smaller than it have to share bit-space with all the rest of the numbers greater than 1. With numbers like these in the mix, how many values can possible be reserved for areas like... between 500 and 501? What about between 1000 and 1001? Between 50,000 and 50,0001?

I've always wondered about this but could never really find a good explanation for how tightly distributed floats are. How quickly do they become really inaccurate? At what point can we no longer represent 1/10th of the difference between two integers? 1/2? At what point are entire whole numbers sacrificed to the pigeonhole principle?

I've spent a lot of time with Unity over the past couple of years, and I always particularly wondered what this means for a property like Unity's [Time.time](https://docs.unity3d.com/ScriptReference/Time-time.html). It's a 32-bit value that represents the time elapsed since the start of the game. Given the above information, we know that 3/4 of the precision of this value are thrown out after the first second of the game. How quickly does this number become completely useless?

I set out to find out. I know the answers to these questions can be answered by examining the [IEEE-754](https://en.wikipedia.org/wiki/IEEE_754) and doing some math. But first of all, I'm not that smart, and second, I'd rather get my hands dirty.



