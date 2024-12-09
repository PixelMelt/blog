---
title: Virtualization
permalink: virtualization
description: Notes on virtualization obfuscation
date: 2024-12-02
tags:
  - obfuscation
  - programming
  - virtualization
  - compilers
---
Coming soon

![[real one.png]]


# Braindump
kinda at a loss for how to make my stuff better so here's random ideas that I mostly stole from elsewhere.

use constants defined in block A to decrypt B and C which branch from A

Spin opcode indexs around during execution, could just use a seeded random function

also using seeded random function to decrypt bytecode

Eric says
> The **OPS_FUNCTIONS** was originally an _array_ of ops functions but in the pretty version it was converted to an object with each key corresponding to the index in the original _array_. It would have been too difficult to find each op in an **array** of hundreds of ops visually when working along ShapeSecurity's VM. For this reason alone it was converted into an _object_ instead of an _array_.

I guess this means I should try to shove everything onto one array, my vm already kinda does this tho so I guess im good, unless I add more shit into the opcode array, might make things more confusing if ops and important functions just continuously get shuffled around together.

