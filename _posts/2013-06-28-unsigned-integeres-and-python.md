---
layout: post
title: "Unsigned integers and Python"
description: ""
category: 
tags: []
---

{% include JB/setup %}

The easiest (and portable!) way to get unsigned integers in Python is to import
the required types from the [ctypes
module](http://docs.python.org/2/library/ctypes.html). However, sometimes you
need to convert Pythons 'normal' integers to unsigned values. As will be
explained, the values are still signed integers (or long), but the effect is the same.

If you google _python unsigned integer_ or similar terms, you will find several
posts telling you that in order to convert a signed to an unsigned value, it is
sufficient to do something similar to:

    val & 0xff (val is a 8 bit variable)

However, few explain why this works. In order to understand why, you first have
to know how Python stores a number.  Numbers in Python are stored as either
signed integers or long, depending on which Python version you use (follow the
links
[here](http://stackoverflow.com/questions/2104884/how-does-python-manage-int-and-long)
for more information). For example, the following is the case on my machine
(which is 64 bit):
    
    >>> val = 9223372036854775807 (maximum value of int 64)
    >>> type(val)
    <type 'int'>
    >>> val += 1
    >>> type(val)
    <type 'long'>

By increasing the value of val by 1, I exceed the limit of a signed 64 bit
integer and the value is converted to a long. If Python had used or converted to
an unsigned integer, val would still have been an int. Or, not long.

Signed integers are represented by a bit, [usually the most significant
bit](http://en.wikipedia.org/wiki/Signed_number_representations), being set to 0
for positive numbers or 1 for negative numbers. What `val & 0xff` does is
actually `val & 0x000000ff` (on a 32 bit machine). In other words, the signed
bit is set to 0 and an unsigned value is emulated.

An example:

    >>> val = -1
    >>> val & 0xff
    255

