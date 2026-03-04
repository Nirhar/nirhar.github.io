---
title: List optimizations in functional programming languages.
layout: post
published: false
---

Found this in: http://sevangelatos.com/john-carmack-on/

> There will be a strong urge in many cases to just update a value in a complex structure passed in rather than making a copy of it and returning the modified version, but doing so throws away the thread safety guarantee and should not be done lightly.  List generation is often a case where it is justified.  The pure functional way to append something to a list is to return a completely new copy of the list with the new element at the end, leaving the original list unchanged.  Actual functional languages are implemented in ways that make this not as disastrous as it sounds, but if you do this with typical C++ containers you will die.