+++
title = "A VHDL Code Generator for Wallace Tree"
date = "2014-06-30"

[taxonomies]
tags=["java", "tools", "vhdl", "hardware"]
+++

[Wallace Tree multiplier](https://en.wikipedia.org/wiki/Wallace_tree) is essentially a hierarchical structure of Carry Save Adders, organized in a manner, where each CSA takes care of getting the product bits of some weightage. While the structure takes just 1 cycle to multiply two integers, writing the VHDL code is a pain. A 32 bit wallace tree multiplier would have around 3k lines of code. Add to that, the pain of organizing signals and connecting them individually.

Not a recent venture though. I had written this round about 2 years ago, (in a lousy Bavarian village of Moosburg) when I learned about this problem statement from a friend of mine. He coded it using Java. I did it in C++.

The code is [here](https://github.com/gagan405/WallTree) if anyone is interested.