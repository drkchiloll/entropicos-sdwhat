---
templateKey: blog-post
title: "{uccx scripting tips & tricks: valid date checking} "
author: samwomack
date: 2014-06-09T09:01:13-05:00
categories: ["CCX Tips/Tricks", "UCCX"]
---

**Product**: UCCX version 8.x-10.x (but probably applies to 7.x and below as well)

**Scenario**: A Caller is allowed to enter a date into the keypad of his/her phone (for example, a payment date, date of birth, etc); The Caller enters and Illegal Date like June 31st


**Problem**: If no error checking is performed and the String (Get Digit _String_) is converted to a Date using auto conversion of a CCX _Set Step_ the application will throw an exception and the caller will hear the Error Message. (I may repeat this below because this is a new thing I'm adding to the tips/trick posts.)

If you are new to my blog, then you may not know that my day job is more or less as a Cisco Contact Center Express IVR application developer; you can read a small piece I wrote concerning the development of UCCX apps [here](http://samiamsam.com/2013/12/31/ccx-developer/ "{ccx developer}"). In this post of tips and tricks about developing on top of UCCX, I want to show you a method to validate a date without the potentiality of having an Exception thrown in your Application. If you've ever develop an IVR application, you've probably had some sort of Date input in your application such as a caller entering their Date of Birth, date they want to pay a bill, and/or date they want to go somewhere (think schedules for planes, trains, automobiles, and even ferries/boats). As a note, your UCCX system should have at least an Enhanced IVR Port/CCX seat license. Let's look at an example that encounters a problem if used on the system (think of the string as been given its value when a caller entered the digits on a keypad).


```js
  String sDt = "01672014"
  //You are Reading Correctly
  //This is a Set Step in the CCX Editor 
  Set date = sDt
  // Unfortunately Your Caller Just Got The
  // System Error Message and Can't Continue
```

That's right, the Editor's _Set Step_ can perform auto conversion and I've spoken about my anathema of those procedures [here](http://samiamsam.com/2014/03/07/code-the-data-conversion-black-hole/ "{Code: The Data Conversion Black Hole}"). So how do we get around this Exception and accompanying "application crash". Well you could use the Editor's _On Exception Step _(oh did I mention I don't like these too?). First lets look at the Exception that is thrown when using the _"auto converter__"_ ![Set_ParseException](/img/set_parseexception.png)

Below is the manual way of using the _Set _step with the exception that with the _setLenient_ method "commented out" you won't actually get the _Exception_ in your application.

```java
String sDt = "01672014";
//You are Reading Correctly 
//Throw an Exception if the Date is Not VALID
//sdf.setLenient( false );
date = sdf.parse( sDt );
//This Code Runs but the Other Code Doesn't
//New Date is March 8th, 2014!
//I encourage you to try this
```

The image below is the _Exception_ that is thrown whenever you are explicitly using the SimpleDateFormat Object and you are using the Default _Leniency_ of the String to Date conversion process that occurs in the _Set Step_.


 ![SDF_ParseException1](/img/sdf_parseexception1.png)


As you can see the built-in code that is run underneath the hood of the Editor's _Set Step_ is essentially the same code that I'm writing explicitly (you can tell because the Java (Exception) Classes are the Same). If you've read my Dates post then you know _sdf_ is a reference to a SimpleDateFormat object :) and in this example it takes the user input date (which is always in String format) and converts it to an actual Date object (you need/should understand the differences between these objects String/Date). However, you would never (Eclipse wouldn't let you) call the _sdf.parse(string)_ method without wrapping the code in a _try/catch_ block or in the case of writing this code in Eclipse you could add a _Throws_ statement in the method you writing this code within: _public static void main(String\[\] args) Throws ParseException{ _ Let's stop playing paddy cake now and get to the point of how I would write this (you can write it the way you like, but my method is better). 


```java
//First declare a Boolean Type
boolean bIsValid = true;
//Assume the Date is Good
//This makes it where an Exception is Thrown
sdf.setLenient( false );
//The Try Block throws an Exception
//So the Appropriate Value if Assigned to the Boolean
try {
  date = sdf.parse( sDt ); 
} catch (java.text.ParseException pe) {
  bIsValid = false;
}
if ( bIsValid ) { 
  //Go Play with your Date
} else { 
  //Please Enter a Valid Date
}
```

That's all there is to it..I'll leave the code for those of you who might want to take a look (it does look different than what is pasted here);  [HERE](https://samiamsamdotcom.files.wordpress.com/2014/06/datevalidity.zip).
 
```javascript
return sam;
```