---
layout: post
title: Testing in Scala
---

Testing our code, before, during, and after we build it is as important a part of the software development lifecycle as anything else.  For those of us with who fall squarely in the Test Driven Development camp, testing is an integral part of the design of software systems. Not just designing so that we can test, but testing so that we can design better.

In this short post, I'll introduce a couple of tools that are available for testing in Scala.  Because Scala runs on the JVM, virtually all of the familiar java tools are available.  Here, I'll focus on just two that are favorites of many Scala developers: ScalaTest and ScalaCheck.

## A Little Theory ##
The applications we create are designed to solve problems and they are (hopefully) designed to solve those problems in a reasonably reliable way.  We hope that our solutions work reliably and as expected even in the face of boundary conditions and unexpected inputs.  The "as expected" part can be thought of as an informal definition of a *specification*.  A specification is a some sort of a definition or illustration of what we expect the program to do.  The specification may be quite formal and laced with set-theoretical notation and very precise language.  More typically it is a few sentences or paragraphs clearly indicating what inputs are expected and what outputs or side-effects we expect from the application.

Like a polynomial function beautifully defines a smooth curve through space, a specification might equally describe the clean and ideal behavior of our application.  In reality, our finished application takes discrete values as inputs and does something specific with them.  In some sense, our applications take points and generate points...all very discrete instances in time.  Our specifications once committed to running code are tested with these discrete inputs and hopefully, given enough inputs, we gain some confidence that our application's output lives up to the fluid appearance of our specification without any hiccups or discontinuities.

But how do we test?  The traditional and most widely used approach is to test individual cases, often thought of as unit tests.  Does this function generate this output or have this side-effect when given these inputs?  Unit tests are at the core of testing in general.  Unit tests can be used to help define behavior before code is written, that is, "I want to write a function that will pass this specific test." Unit tests can be used to give some confidence in code after it is written. Unit tests can also be used to document bugs we've found. Once reproduced in a test, a bug can be forever remembered and checked.  But, unit tests are specific checks on inputs and outputs.  The test builder must be skilled enough to consider the edge cases, the cases the developer might not have thought of, as well as the common cases. To be sure, a skilled test designer can do this.  Most of us are not skilled (enough).  

Property-based testing is a slightly different take on unit testing.  It's not radically different in many respects, but it can help us be more thorough. In property-based testing, we define tests that are intended to evaluate a bit of code over a wide range of possible inputs.  In the place of specific inputs and checks that a specific output was generated, we build more of a specification of what the outputs should be given any inputs.  This is much more like testing the specification of the program rather than specific inputs.  But, in the end, it's much more like having an underlying test harness that can run the unit tests over and over again with lots of different inputs. This is kind of like approximating the curve with lots of points. The more points the better. However, well chosen points (as with an expert unit test writer) can often convey as much valuable information about the shape of the curve.

## The Code ##
Imagine we are given the lofty and difficult task of writing a method that fits a string into a spot on a user interface in such a way that no more than _**N**_ characters are taken up by the string.  Anyone who has built desktop applications knows that screen space is limited. Some spot on the screen can really only be 20 characters wide and sometimes we need to put a 100 character string in there.  The easy choice is to chop it off at 20 characters.  A little better choice is to print a few characters of the start of the string, indicate that there is some hidden stuff too wide to display in the middle (usually with ...) and finish off the text with the last part of the string.  So we may need to display: "The rain in Spain falls many on the plains."  But we can only fit: "The rain...on the plains" on the allotted screen real estate.  

Given this horrendously complex task (just kidding of course), you might come up with the following:

{% highlight scala %}
package com.trc.blog

object Utils {

  def DotifyString(s: String, maxLength: Int) : String = {
    val dotString = "..."
    var result = s
    if (s.length() > maxLength) {
      val dotStringLength = dotString.length()
      val partsLength = (maxLength - dotStringLength) / 2;
      val left = s.substring(0,partsLength)
      val right = s.substring(s.length() - partsLength)
      result = left + dotString + right
    }
    result
  }
}
{% endhighlight %}
 
Here we take a string and a max length that is available for the string.  We try to evenly chop off the first and last parts of the string and stuff the ... characters in between.  Not altogether clever nor efficient, but it works...or does it?

## Unit Tests ##

Your first approach, of course, is traditional unit testing. In Scala the ScalaTest framework is popular.  You can, of course, the JUnit because Scala runs on the JVM.  But, you're a dedicated Scala developer so you choose to stick with your own kind.  Here's what you might come up with:

{% highlight scala %}
package com.trc.blog

import org.scalatest.FunSuite

class DotifyUnitTests extends FunSuite {

  test("string that doesn't need to be dotified") {
    val testString = "Hello World!"
    assert(Utils.DotifyString(testString, 15).length() == testString.length())
  }

  test("string that does need to be dotified") {
    val testString = "Hello this is a really long string that should work in our cruel World!"
    assert(Utils.DotifyString(testString, 15).length() <= 15)
  }
}

{% endhighlight %}

In ScalaTest individual tests are introduced and documented with test() method. The body of the test does all of the appropriate work for the test including asserting some final result.

If you mixin _**BeforeAndAfter**_, you can also use _**before**_ {} to do any work you need to set up the individual tests and you can use _**after**_ {} to clean up after the individual tests.  In our example, things are simple enough that we don't have any work for these two phases.

What happens when we run these tests?  

{% highlight tcsh %}
ff62ps1:ScalaTesting ltenny$ sbt test
[info] Set current project to ScalaTesting (in build file:/Users/ltenny/Blog/repositories/ScalaTesting/ScalaTesting/)
[info] DotifyUnitTests:
[info] - string that doesn't need to be dotified
[info] - string that does need to be dotified
[info] ScalaCheck
[info] Passed: Total 0, Failed 0, Errors 0, Passed 0
[info] ScalaTest
[info] Run completed in 203 milliseconds.
[info] Total number of tests run: 2
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 2, failed 0, canceled 0, ignored 0, pending 0
[info] All tests passed.
[info] Passed: Total 2, Failed 0, Errors 0, Passed 2
[success] Total time: 1 s, completed Jun 22, 2015 1:11:39 PM
ff62ps1:ScalaTesting ltenny$ 

{% endhighlight %}

Wonderful news! Your code passed the unit tests. Ship it. Wait a minute. Really? 

## ScalaCheck ##

No, unfortunately, life is not always that easy. After all, we really just checked two cases.  Does that really tell us much about how well the code meets the specification?  Do we really know much about the curve from just two points...especially these poorly chosen points?

With ScalaCheck we depend on generators that produce a potentially wide range of inputs for our test.  A little like using a wide domain of randomly chosen points to help us understand the range of outputs.  

Here's what the test would look like in ScalaCheck.

{% highlight scala %}
package com.trc.blog

import org.scalacheck.Properties
import org.scalacheck.Prop

class DotifyCheckTests extends Properties("Utils") {

  property("dotify") = Prop.forAll { (s: String, maxLength: Int) =>
    val result = Utils.DotifyString(s, maxLength)
    result.length() <= maxLength
  }
}

{% endhighlight %}

ScalaCheck is all about properties and generators. Here we define a property "dotify" that uses the universal qualifier and the default generator of strings and integers to test a large number of inputs.  In our validation we simply check if the resulting string is no more than the max length that the generator tested with.  And what happens when this test is run?

{% highlight tcsh %}
ff62ps1:ScalaTesting ltenny$ sbt test
[info] Set current project to ScalaTesting (in build file:/Users/ltenny/Blog/repositories/ScalaTesting/ScalaTesting/)
[info] ! Utils.dotify: Exception raised on property evaluation.
[info] > ARG_0: ""
[info] > ARG_0_ORIGINAL: "裀"
[info] > ARG_1: -1
[info] > ARG_1_ORIGINAL: -129897156
[info] > Exception: java.lang.StringIndexOutOfBoundsException: String index out of range: -2
[info] ScalaCheck
[info] Failed: Total 1, Failed 0, Errors 1, Passed 0
[info] ScalaTest
[info] Run completed in 179 milliseconds.
[info] Total number of tests run: 0
[info] Suites: completed 0, aborted 0
[info] Tests: succeeded 0, failed 0, canceled 0, ignored 0, pending 0
[info] No tests were executed.
[error] Error: Total 1, Failed 0, Errors 1, Passed 0
[error] Error during tests:
[error]         com.trc.blog.DotifyCheckTests
[error] (test:test) sbt.TestsFailedException: Tests unsuccessful
[error] Total time: 1 s, completed Jun 22, 2015 1:21:53 PM
ff62ps1:ScalaTesting ltenny$ 
{% endhighlight %}

Oops...well, that didn't go well.  Hold off on shipping the code.  When ScalaCheck finds a failing case, it attempts to simplify the case as much as possible. Here it found that when the string was one unicode character and the maxLength was -1, the java.lang.StringIndexOutOfBoundsException was thrown.  It tried again, this time with just the maxLength set to -1 and found the simpler case with the same exception.  We didn't consider the case where maxLength was negative.  Our unit tests didn't check for it.  A better unit test writer probably would have checked for it.

The moral of the story: Write better unit tests **and** use ScalaCheck.

#### References ####
A wonderful reference for ScalaCheck is: _ScalaCheck, The Definitive Guide_ by Rickard Nilsson, Artima Press, Walnut Creek, CA.  This is available on Amazon. It covers ScalaCheck in depth and provides some great examples as well as very good insight into testing scala applications.

##### The code for this post can be found <a href="https://github.com/ltenny/ScalaTesting">here</a>.#####




 