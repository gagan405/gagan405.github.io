---
layout: post
title: Learning scala and play framework
---

In an attempt to learn a bit more scala and some [play-framework](https://www.playframework.com/), I wrote this a bit messy code over the last weekend.

Its a snake ladder board game with REST interface to start a game or roll a dice. The APIs return simple strings and there is no fancy UI.
In fact, my major effort went into writing [this logic](https://github.com/gagan405/snake-ladder/blob/master/app/in/umlaut/entities/Board.scala#L103-L148) which prints the status of the Game board in ASCIIs.

The Board cells expand or contract as required depending on the cell content length.

Sample:

~~~
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+
| 100 | 99  | 98  | 97  | 96  | 95  | 94   | 93   | 92  | 91  |
|     |     |     | S2s |     |     | S9s  | L10e |     |     |
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+
| 81  | 82  | 83  | 84  | 85  | 86  | 87   | 88   | 89  | 90  |
|     |     | L9e | L3e |     |     | L10s | L8e  | S8s |     |
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+
| 80  | 79  | 78  | 77  | 76  | 75  | 74   | 73   | 72  | 71  |
| L9s |     |     |     |     |     |      |      |     |     |
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+
| 61  | 62  | 63  | 64  | 65  | 66  | 67   | 68   | 69  | 70  |
| S6s |     | S9e | L6e |     | S7s |      |      | L7e | L8s |
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+
| 60  | 59  | 58  | 57  | 56  | 55  | 54   | 53   | 52  | 51  |
|     |     |     |     | L5e |     | L7s  |      | L4e |     |
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+
| 41  | 42  | 43  | 44  | 45  | 46  | 47   | 48   | 49  | 50  |
|     | L6s |     | S5s | S1s | S7e | S2e  |      |     |     |
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+
| 40  | 39  | 38  | 37  | 36  | 35  | 34   | 33   | 32  | 31  |
| S8e | S4s |     | L5s | S6e |     |      |      |     | L4s |
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+
| 21  | 22  | 23  | 24  | 25  | 26  | 27   | 28   | 29  | 30  |
|     |     | L3s |     | S3s |     | S5e  | L2e  |     |     |
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+
| 20  | 19  | 18  | 17  | 16  | 15  | 14   | 13   | 12  | 11  |
|     | L1e |     | S4e | S3e |     |      |      |     |     |
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+
| 1   | 2   | 3   | 4   | 5   | 6   | 7    | 8    | 9   | 10  |
|     | L1s |     |     | S1e |     | L2s  |      |     |     |
+-----+-----+-----+-----+-----+-----+------+------+-----+-----+

~~~

Code here : https://github.com/gagan405/snake-ladder
