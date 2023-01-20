---
layout: post
title: How I Explain Violating the Law of Demeter
categories: programming
---

[The Law of Demeter](https://en.wikipedia.org/wiki/Law_of_Demeter) is a programming concept people are often aware of, but that can be very easy to skip over or not notice as its happening. I've found a metaphor that has worked fairly well at helping think about it and explain why we should avoid it.

I got started thinking about this idea after a comment on [an episode of The Bike Shed](https://www.bikeshed.fm/353) where they talked about considering a person reaching in to another's backpack to take something from it. This concept has worked well for me in discussing LoD with others.

If you need a pen, you can ask me if I have one I can give you.

I might give you a pen, or I might tell you I can't. You don't get to know if I have pens available and decide for yourself to take one.

If I do give you a pen, you don't need to know where it came from. It shouldn't matter if I got it from my backpack, my desk, or if I asked another person to give it to me.

I hope you wouldn't dig through my backpack, rifle through my desk, or take my phone and text my friends to ask for a pen.

And in the opposite direction, I hope you wouldn't put a pen (or anything else) in my backpack. While I might appreciate the gift, I should be able to accept the pen from you directly and then decide where I want to put it.

The Law of Demeter wants us to show similar restraint in how we get and set data between objects. Delegate responsibility to the objects themselves, rather than reaching in to them directly. Avoid surprises, side effects, and letting objects know too much about each other.
