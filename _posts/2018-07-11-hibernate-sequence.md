---
layout: post
title: Hibernate sequence generation
---

Some years ago I had this weird encounter with hibernate where ids were getting mixed up among 2 different tables, due to `last insert id` and `lazy commit`.

That was another year, another time and another hibernate version.

This time I got a new learning with the ID generation behavior.

To the point :

Make sure database sequence table has the same `increment by` as the `allocationSize` in `@SequenceGenerator`.
~~~sql
SELECT * FROM table_id_seq;
ALTER SEQUENCE table_id_seq INCREMENT BY 20; // 20 is the allocationSize in hibernate
~~~

### What happens if they are different ? 

* `allocationSize` (10) < `incrementValue` (20)

   Hibernate gets the next value = 100.
   
   Generates Ids till 110.
   
   Requests for nextValue. Gets 120.

   Works fine, but holes are created.

* `allocationSize` (10) > `incrementValue` (1)

   Hibernate gets the next value = 100.
   
   Generates Ids till 110.
   
   Requests for nextValue. Gets 101.
   
   Fails with `ValueExistException`.


### Why is it a big deal? Should be caught during testing.

Well, what makes it a bit obscure is, when worked with previous version, some properties (I assume `hibernate.id.new_generator_mapping`) were set in a way that even if there was mismatch, it relied with the `increment by` value of database.

With the 2.0 version of Spring Boot, it comes forward.

### What about identity type generation ?

Identity type generation is certainly more cleaner, may be not much performant. However, it also requires changing the table definition. 
