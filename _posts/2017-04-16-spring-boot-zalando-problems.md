---
layout: post
title: SpringBoot -- Exceptions to Responses
subtitle: Making sense of exceptions using Zalando Problems
---

As applications grow, number of APIs increase, number of consumers increase, it is extremely critical to have a uniform response structure across all APIs. Different APIs behaving differently, conflicting error codes or a `HttpStatus` of 500 for something like a invalid credit card number or email in the request payload is highly undesirable.

The standard way of responding error/exceptional situations is well documented in Spring. This roughly can be done like :

* In your resource method, return a `ResponseEntity`, with the desired Object body.
This will look something like this :

~~~java
return ResponseEntity.status(401).body("{\"message\" : \"Invalid password\"}");
~~~

* Or the other way could be, and more preferable way, is to throw Exceptions and write Exception mappers which then convert the exceptions to appropriate response objects. Google for it and numerous examples are available on this.

Quite some time ago, I found this [Zalando Problems] library, that takes the second approach, and generates nice reponse structures for various exception types, and is easy integrated with [Spring Boot].

Their package, by default handles quite a number of exception types. But, as the need to have custom exception types come, we have to provide `AdviceTrait`s for the new exception types. Which is done like something as below :

A custom exception class :
```java
public class InvalidEmailException extends RuntimeException {
  public InvalidEmailException(String msg){
    super(msg);
  }
}
```

An advice trait that converts it to a response object:
```java
public interface InvalidEmailAdviceTrait extends AdviceTrait {

    @ExceptionHandler
    default ResponseEntity<Problem> handleInvalidEmail(
            final InvalidEmailException exception,
            final NativeWebRequest request) {
        return create(Status.BAD_REQUEST, exception, request);
    }
}
```

Now we plug this in along with the other traits:
(`ProblemHandling` is the packaged interface from zalando library.)
```java
public interface CustomProblemHandling extends ProblemHandling, InvalidEmailAdviceTrait{
}
```

We need to provide this as a `ControllerAdvice` so that spring makes use of it.

```java
@ControllerAdvice
public class CustomExceptionHandler implements CustomProblemHandling{
}
```
And thats it! You raise a `InvalidEmailException` and the appropriate response is returned.

```java
throw new InvalidEmailException("Invalid email format.");
```

Response :

```
{
	"title": "Bad Request",
	"status": 400,
	"detail": "Invalid email format.."
}
```

To be noted that the response will have a `Content-Type` of [`application/problem+json`](https://tools.ietf.org/html/rfc7807).

   [zalando problems]: https://github.com/zalando/problem
   [spring boot]: https://github.com/zalando/problem-spring-web
   
