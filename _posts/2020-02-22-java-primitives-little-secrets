---
layout: post
category: programming
---
Here is a bunch of java primitives' interesting behaviour that we ofter ignore or forget! Recap time in 3 - 2 - 1...   

### Data type promotions
When you try to perform a numerical operation on a short type for example, the result will be promoted to int, therefore it wont compile if you try to store the result in a short variable unless you explicitly cast it (check line x).  

```java
short x = 5;
short y = 2;
short z = x*y // does NOT compile
short z = (short)(x*y) // does compile

//////////////////////////////////////

short x = 5;
short y = 2;
int u = x+y // it does

```

The same applies to byte. Adding two numbers of type byte they will automatically be promoted to short.

What about int? Adding two number of type int together wont result to a long promotion (of course)...

Same as int if you try to to add together two float numbers, the result wont be promoted to double.

All the above are great... but... There is an interesting behaviour when you do the following

```java
byte x = 5;
byte y = 3;
y+=x;
```

the above is perfectly valid, but the bellow results in a compilation error:

```java
byte x = 5;
byte y = 3;
y=y+x; // doe NOT compile
```

Yes indeed!

an easier one now:
Please welcome the infamous Numeric Overflow!


```java 
System.out.println(Integer.MAX_VALUE);
//output 2147483647

System.out.println(Integer.MAX_VALUE + 1);
//output -2147483648

System.out.println(Integer.MIN_VALUE); 
//output -2147483647

System.out.println(Integer.MIN_VALUE - 1); 
//output 2147483648
```
