---
layout: post
title: Migrating a legacy service from Java 8 to Java 17
---

I just migrated a legacy service from Java 8 to 17 and it was a smooth experience, except for a few. Here are some lessons learnt in the process.

Java 17 has a lot of changes when compared to Java 8, of course. And the most important one seems to be the [strongly encapsulated JDK internals](https://blogs.oracle.com/javamagazine/post/a-peek-into-java-17-continuing-the-drive-to-encapsulate-the-java-runtime-internals).

So, how does it matter to you, as a service code author, who rarely does any playing around with JDK internal APIs?

Well, we probably don't use them directly. But they could be used through other useful libraries that we use: such as EasyMock/JaxB/PowerMock etc.

The process is quite simple though. And here I try to summarize:

**Use Top-Down migration**
- 

Code written and compiled in JDK8/11 will work as a dependency in a package compiled in 17. But we cannot have it the other way round.
So, if you have some top level package/service; start with that.

**Upgrade dependency to JDK-17 and fix broken packages**
- 
Certain packages that I found breaking were:
* PowerMock
* EasyMock-4.x

**Remove PowerMock**
-
PowerMock has been used primarily to mock:
1. Static methods
2. Final classes
3. Mocking constructors

While most of it could be achieved through `Mockito-5.*` and `mockito-inline`, it is a good opportunity to re-visit the necessity of such tests.
In general, if you need a bytecode mingling library such as PowerMock to test a static method; isn't that a sign of poor code design?

There are many articles on the web explaining why you shouldn't be using PowerMock. So, read through them and convince yourself.
Examples: [this](https://automationrhapsody.com/powermock-examples-better-not-use/), [this](https://medium.com/@inderivita/powermock-testing-tool-or-testing-obstacle-29f3303c9337) and [this](https://news.ycombinator.com/item?id=8775233).

So, in such cases, I would ask myself:
* Can I refactor the code structure to not have static utility classes?
* Can I *not mock* static methods and go ahead with testing them as part of unit tests.
* Can the `new Object(...)` code portions be abstracted out to dependency injection frameworks so that I dont have to test those?
* Do I really need final classes?

Many such cases could essentially be refactored out for a better manageable code that wouldn't really require such a library in the first place.

***Stateless simple static methods***

If static util method is something similar to `StringUtils.isNotNull()`, then don’t mock it and let it execute the real method.

***Complex static methods***

If the static util method does complex stuff within the utility method, such as n/w calls, FileIO, or some complicated computation: explore options of refactoring. Easiest way is to just make it non-static and have it as an object injected to the consumer class.

Static methods from Thread/System classes
If the static method is using System or Thread methods or [native methods](https://blog.frankel.ch/on-powermock-abuse/), such as `Thread.sleep()` , `mockito-inline` does not support mocking those methods and consider creating wrapper classes over such methods.

Example:

~~~
public class Delayer {
    public void delay(long millis) throws InterruptedException {
        Thread.sleep(millis);
    }
}
~~~

***Static methods using Date Times***

If the static methods rely on date time, consider using `mockMethods` in the real class and call those from the tests.
Credit where due: [Reference on the below snippet](https://dzone.com/articles/mock-java-datetime-for-testing)

~~~
private static Clock CLOCK = Clock.systemDefaultZone();
private static final TimeZone REAL_TIME_ZONE = TimeZone.getDefault();

// Call this method in your test cases
public static void useMockTime(LocalDateTime dateTime, ZoneId zoneId) {
    Instant instant = dateTime.atZone(zoneId).toInstant();
    CLOCK = Clock.fixed(instant, zoneId);
    TimeZone.setDefault(TimeZone.getTimeZone(zoneId));
}

// call this method to reset the clock back
public static void useSystemDefaultZoneClock() {
    TimeZone.setDefault(REAL_TIME_ZONE);
    CLOCK = Clock.systemDefaultZone();
}
~~~

**NOTE** : The above also introduced me to a very cool library [`archunit`](https://www.archunit.org/) library. Also, I spent at least 2 hours in getting the regex working for intellij test directory:
~~~
private static final Pattern IDEA_TEST_PATTERN = Pattern.compile(".*/out/test/([^/]+/)+?[a-zA-Z]+Tests?.*");
~~~

*CAUTION :*

In `junit4`  the expected exceptions are written at the top level such as this:

~~~
@Test(expected = SomeException.class) {
    useMockTime(currentDateTime, zoneId);
    ...
    ...
    useSystemDefaultZoneClock();
}
~~~

The above will __NOT__ work as the final statement will not get executed due to the exception above. To handle these:

* Use `try...catch`  and then reset
* Use AssertJ library method [assertThatThrownBy](https://www.javadoc.io/doc/org.assertj/assertj-core/3.8.0/org/assertj/core/api/Assertions.html#assertThatThrownBy-org.assertj.core.api.ThrowableAssert.ThrowingCallable-)
* OR simply use `Junit5`: `assertThrows(SomeException.class, () -> someMethodWhichThrowsException());`


***Mocking new instantiations***

Refactor to not have such cases and use Dependency Injection.
Take advantage of DI frameworks and try refactoring the code in a way that you wouldn’t need such cases.
If that’s not an option, see below.

Use Mockito [`mockConstruction`](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html#mocked_construction)
Note that mockConstruction doesn’t support argument captor. So, if you want to validate that a certain class was instantiated with a specific type params, you would need to pass a supplier to capture the constructor args, and then assert those. Sample as [this](https://groups.google.com/g/mockito/c/WqPlcNrvbOw/m/0WzhI51sAgAJ).

This approach looks un-maintanable, but I am not aware of any better approach yet.

**Removing EasyMock**
-

EasyMock-5.x is [supposed to work](https://github.com/easymock/easymock/pull/300) with JDK-17. But do you really need to have 2 separate mocking libraries in your dependency closure?
Choose one and stick to it preferably. That makes the test cases consistent and uniform across the project.

**Issues With ErrorProne**
-

ErrorProne doesnt work well with Java17 and there are a few open issues such as https://github.com/google/error-prone/issues/1250

This mostly comes from Lombok generated files not working well with ErrorProne. Some of them are not solved even after adding Generated annotation or using jvmargs specified in the above issue.


**CheckStyle**
-
The classic CheckStyle rules do not work with Java17. Some of the module hierarchy have changed. The rules need to be updated. Refer to the latest [checkstyle rules](https://checkstyle.sourceforge.io/index.html) and fix them!

***Formatting changes in git***

As part of checkstyle upgrade, if you are also reformatting your code, then you might want to ignore such commits appearing in your git blame  (IntelliJ annotations). To do that, you can use this git config changes. [Here](https://tekin.co.uk/2020/09/ignore-linting-and-formatting-commits-when-running-git-blame) is an article with instructions.

**Runtime tests**
-
When all the tests and builds are going fine, its time to do a runtime test. There could still be errors in runtime due to who knows what library accessing internal JDK APIs!
One such error I found was with Jaxb:

~~~
Caused by: java.lang.IllegalStateException: Unable to initialize jaxbContext
~~~

Solution is simple. Just add the dependencies to [Jaxb-api](https://mvnrepository.com/artifact/javax.xml.bind/jaxb-api).

Also, JaxB legacy properties have been deprecated long, and they need to be replaced with their upgraded counterparts. For example:
~~~
Marshaller marshaller = jaxbContext.createMarshaller();
marshaller.setProperty("com.sun.xml.internal.bind.xmlDeclaration", Boolean.FALSE);
~~~

the above snippet needs to be changed to

~~~
Marshaller marshaller = jaxbContext.createMarshaller();
marshaller.setProperty(Marshaller.JAXB_FRAGMENT, Boolean.FALSE);
~~~

Similarly, many of the old JVM args are now deprecated and that would cause the service to fail to start. Those can simply be ignored by using the flag `-XX:+IgnoreUnrecognizedVMOptions`

**Validation**

Verify how JDK17 is performing with live traffic. What should be the GC/ JVM params and so on.

**Conclusion**
-
There you have it. Service running with Java 17! Enjoy the optimizations and new language features.
Happy Coding!
