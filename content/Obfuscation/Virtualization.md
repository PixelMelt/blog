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


Ideas that I mostly stole from elsewhere.

use constants defined in block A to decrypt B and C which branch from A

Rotate opcode indexs around during execution, could just use a seeded random function

also using seeded random function to decrypt bytecode

Eric says
> The **OPS_FUNCTIONS** was originally an _array_ of ops functions but in the pretty version it was converted to an object with each key corresponding to the index in the original _array_. It would have been too difficult to find each op in an **array** of hundreds of ops visually when working along ShapeSecurity's VM. For this reason alone it was converted into an _object_ instead of an _array_.

What makes shape actually hard to reverse?
- opcodes aren't atomic and are split into multiple "actions" what when run in sequence have the same effect
- strings get decrypted at runtime using 2 indexes of an array of base64 strings that get xored together
- who knows (not me)