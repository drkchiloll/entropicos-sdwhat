---
templateKey: blog-post
title: "{uccx problem solving series: XML and the outdated SAX parser} "
author: samwomack
date: 2014-04-13T10:04:48-05:00
categories: ["UCCX"]
tags:
  - UCCX
---

A lot of you reading the title of this post may be asking yourself..what does a "SAX" have to do with UCCX...I'm sure there are many bugs to pick with how Cisco's products operate (if that was a pun then it was definitely intended) especially when things go sideways..such as parsing XML documents. When I use the _Create XML Document_ normally followed up with the _Get XML Document Data_ what is occuring under the hood..besides supplying the Java Beans with Document, XPath Expression (what the H is that), and Output(String)..e.g. the "return" or the Data we really want from that document. That my dear readers is a mystery that you will "never" find out(never say never)..until an exception is thrown, you look deep into the bowls of the log files, and/or you "hack" the box and download the JARs Cisco uses for the product. But back to SAX: simply put \*I believe\*(based on debug outputs I will show) it is the parser Cisco uses behind the scenes to help Aid us (or hinder us in the case today) in the Parsing of XML Documents..and the version of it would be SAX1 (and I surmise this because v1 doesn't handle _XML Namespaces_ well and SAX2 addressed those issues).

 This post is/was inspired by a Cisco supportforum thread in which an XML Document was being downloaded, "parsed"(Create XML Document actually sets up the Parser), and an attempt to extract data "failed" on the DOC (_null _String; the doc in that post wasn't a well-formed doc btw)..Lets take a look at what the well-formed XML is supposed to be:

```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <string xmlns="http://schemas.microsoft.com/2003/10/Serialization/">
    <storeInfo>
      <storeID>654</storeID>
      <storeType>Main</storeType>
    </storeInfo>
  </string>
```

There nestled in the _root_ tag/element (**<string>**) is the surreptitious NameSpace. Again as I stated earlier using UCCX's standard procedure for parsing XML \*DOES NOT WORK:

![xml_1](/img/xml_11.png)

Before we move on..Go out to the provided link and read an overview or all of the information pertaining to XPath ([http://www.w3schools.com/XPath/](http://www.w3schools.com/XPath/)). Because of Cisco Documentation the XPath Expression that Looks Like the Below is going to be hard to kill off, but with every chance I get I try!

```javascript
// This is not genealogy..This is Code! 
"//descendant::storeInfo/child::storeType"
// Can Be Written in a More Compact Form Below
"//storeInfo/storeType"
// Or If you Don't have an "Array" of Store Type
// Elements, you could use this form:
"//storeType"
```

From here on out..we will write the XPath Expression in the form of the former "style". Let's dive right in on a method to fix the issue we are having "right up front." If we can manipulate what is returned from the Web Service (let's say it resides on internal servers with internal staffing) we could ask the developers there if they can modify the returned XML and the only thing on the doc to modify to make it work is below:

```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!--XMLNS:STRING NAMESPACE PREFIX-->
  <!--NAMESPACE:ELEMENT-->
<string xmlns:string="http://schemas.microsoft.com/2003/10/Serialization/">
  <storeInfo>
    <storeID>654</storeID>
    <storeType>Main</storeType>
  </storeInfo>
</string>
```

If that occurs then we are done! Done! Let me verbalize it completely; when we move through a debug session of the application as denoted in the top image of this post, the String(s) we are trying to assign value to based on the text nodes in the _storeID_ and _storeType_ elements will be _null _after evaluation. However, if we add the Namespace **Prefix** (prefix being the key here) to the Root Element/TAG (<string>), those Strings will be properly assigned the value as extracted from the xml (when debugged). However, this cannot always be done though..because that Web Service we are linking to doesn't always just spit out data for your IVR. Now I will discuss the tragedy that was with the first solution that I would go with..then we will play a bit..and after that provide the ultimate solution to dealing with the NS like they aren't even there with 1 big giant Caveat that leads me to leave what "works" with no modifications whatsoever for last! First, let me "puke" out the first solution:

```java
// In CCX You have to Include the Package Name when Declaring
// The Objects: org.w3c.dom.Document doc;
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
DocumentBuilder builder = factory.newDocumentBuilder();
Document domDoc = builder.parse(doc);
XPath xpath = XPathFactory.newInstance().newXPath();
String storeId = "//storeID/text()";
String storeType = "//storeType/text()";
String storeId = xpath.evaluate(storeId,domDoc);
String storeType = xpath.evaluate(storeType,domDoc);
```

I would say all Java developers would steer you away from utilizing the standard DOM that ships with java as a standard builtin library because it is slow and memory intensive (although in this case we are merely opening the document and loading the document into memory where it can be operated on by the XPath Library which is compatible with DOM Documents); I thought about making a statement about our "forefathers", their 20k of RAM...but I don't think it worth it :).

Shall we have fun now..and perform an operation that you will not use under any circumstances? We are going to convert the XML Document into a String..and proceed with surgery as quickly as possible:

```java
// CCXes SET Step Converts a DOC to String "for you"
String myString = doc
// Get the Integer Location of first the '>'
// Char in the String
int index = myString.indexOf(">");
// Create a "new" Value for the String
// Immutability says it will be a completely new string fwiw
// Go 3 Chars Past the first '>' == Char (\\r\\n)
myString = myString.substring(index + "3");
// Removing Closing </string> TAG with EMPTY Str
myString = myString.replace("</string>", "");
doc = TEXT\[myString\];
// This is not a very MEMORY Efficient Process AND
// You Will not Use This Method
```

Before we look at how our document is transformed lets talk briefly about _"Immutability."_ In several languages besides Java you can get into real trouble with the String type especially when you are performing manipulation like the above because of this term: immutability.  What that means is that every time a String is seemingly modified (you set the value of a String to another different value such as String concatenation, substring modifications, removing characters from a String, changing the case of a character in a String to say toUpperCase(), etc etc a NEW String is created and the Old Version is abandoned on the Heap occupying space in memory. If you read the Cisco provided CCX Best Practice document they do make warnings about how you use the String type as do most applications that allow the use of Languages with Immutable String types such as Java, Python, C#, et al.  But I digress.. I would ask you to guess what the resulting document looks like..but I'm just going to go ahead and show the result.

```xml
  <?xml version="1.0" encoding="UTF-8"?>
<storeInfo>
  <storeID>654</storeID>
  <storeType>Main</storeType>
</storeInfo>
```

That was fun..but don't do it! One of the responding posters sugguested using this methodology..and I would say..just because you can..well you know the rest..(and oddly the same may apply to the DOM Document code I posted previously esp. for some out there). As promised, the "proper" answer and an evasive caveat (what caveat isn't though right?):

```java
String stoTypeExpr = "//\*\[local-name()='storeType'\]";
String stoIdExpr = "//\*\[local-name()='storeID'\]"; 
// If you were testing these expressions in XMPlify you would
// End those Expression with "/text()"
```

Here is an example screenshot of what the expression looks like..and perhaps a view into the potential caveat:

![xpathexprsn](/img/xpathns_expr.png)

I'm going to leave you there..thinking about that little gem of a "problem"; how would you handle a document like this that returns multiple instances of the _"storeInfo"_ TAG (especially with CCX not handling the expressions properly "out of the box."). Given a Choice, I would Hope for Solution Number 2 Being the Answer..and then using the normal provided methods (asides from _descendants _and _children_of them...

```javascript
return sam; 
```