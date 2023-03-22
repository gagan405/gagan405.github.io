---
layout: post
title: IntelliJ ambiguous method on Lombok
subtitle: Both method A and method A match
---

This is a note to self. A stupid error that was bugging me for sometime.
 
There is this nice Lombok plugin for IntelliJ which runs just fine. Of late, I did certain upgrades on IntelliJ and started seeing this error :
~~~~
"Ambigous method call. Both method() and method() match."
~~~~

As such, it is confusing. There were the normal fields as anyone would expect in a pojo class, with `@Data` annotated which results in generation of getters, setters and equals and hashcode methods. Nothing so special about it. So, why was it saying there were two getter methods or two setter methods ?

Later I realised there was another plugin installed which also does the lombok annotations processing, called [Hrisey Plugin](https://github.com/rohitbhatnagar85/garageplug-oms.git). 
Disabled that, and ready to go!   

