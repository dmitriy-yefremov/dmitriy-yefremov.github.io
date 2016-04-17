---
layout: post
title: How not to use mocks
---

# Test doubles

Before talking about mocks I want to define what is a mock. In our everyday lives we often use the term mock
for any object replacing a real production object in a test. That is not correct and may cause confusion.
 
Let's look at different kinds of test doubles (those replacement objects) on a simple example. Imagine we have
a phone book object that allows to store and retrieve phone numbers.

```java
public interface Phonebook {

  String setNumber(String name, String number);
  
  String getNumber(String name);

}
```

And also there is something else that uses Phonebook. It's that something else we are testing, not Phonebook itself. 

The real Phonebook implementation would probably save and retrieve number from a file or a database. It may be impossible
or impractical to use the real implementation of this object in unit tests. We need to introduce a replacement.

## Stubs

A stub always returns the same previously set value(s) and has no logic.

```java
public class PhonebookStub {

  private final String number;
  
  public PhonebookStub(String number) {
    this.number = number;
  }

  String setNumber(String name, String number) {
    // do nothing
  }
    
  String getNumber(String name) {
    return number;
  }

}
```

As you can see PhonebookStub will always return the same number each time you ask. If you don't ask at all it's fine too.

## Fakes

A fake contains a special implementation tailored for testing. Our fake is going to use an in-memory map to store and 
retrieve phone numbers.

```java
public class PhonebookFake {

  private final Map<String, String> numbers = new HashMap<>;
  
  String setNumber(String name, String number) {
    numbers.put(name, number)
  }
    
  String getNumber(String name) {
    return numbers.get(name);
  }

}
```

In this example the fake implementation is very simple, but sometimes creating a fake requires a significant amount of
coding (e.g. an in-memory SQL database).

## Mocks

A mock allows to set expectations about how it should be called together with the answers expected. Typically mocks are
created using a mocking framework such as [EasyMock](http://easymock.org/) or [Mockito](http://mockito.org/).

```java
import static org.mockito.Mockito.*;

// create a mock
Phonebook phonebook = mock(Phonebook.class);
// set expectations
when(phonebook.getNumber("Alice")).thenReturn("1234567890");

...

// verify expectations (optional)
verify(phonebook).getNumber("Alice");
```

If you find yourself creating a mock class (e.g. PhonebookMock), most likely what you are really doing is creating a
stub or a fake.

# Why mocks are bad?

They are not. Mocking is a great tool when used appropriately. But too much mocking makes tests hard to read and
maintain. Somehow the presence of a mocking framework on the classpath makes people go crazy and mock everything without
much thinking. Here are some typical consequences of mocking overused:

* Hard to read, noisy tests.
  Modern mocking frameworks provide pretty nice DSLs. But still they are limited by the syntax of the language used and
  are not perfect. A lot of expectation statements and behavior verifications increase test methods size and distract
  from testing logic. It gets much worse when advanced techniques such as
  [ArgumentCaptor](http://docs.mockito.googlecode.com/hg/org/mockito/ArgumentCaptor.html) are used.    
* Tight coupling of test code with the implementation being tested.
  Unless you do [behavioural testing](http://googletesting.blogspot.com/2013/03/testing-on-toilet-testing-state-vs.html)
  you want your tests to stay the same as you change the implementation. That is one of the benefits of having tests -
  you refactor the implementation and if the tests still pass you can be confident you didn't break anything. With mocks
  used for testing you may end up almost repeating your implementation logic in your tests as you set all expected calls
  one by one. It's almost certain that your tests will fail if you change the implementation. You lose the refactoring
  safety net and increase the maintenance cost.
* Compromised implementation quality.
  This one is not quite obvious. How can the choice of testing tools impact your production code? It is well known that
  one side effect of testing is a better design of your production code. To make code testable you need to break it into
  separate modules (e.g. classes and functions), introduce some reasonable abstraction layers, organize dependencies
  between components. You want to do this anyway, but testing reinforces it. Now, going back to mocking. This tool is
  almost too powerful. It allows you to write tests for any kind of messy code. Given that nowadays you can mock classes
  and not only interfaces, create partial mocks and even
  [mock static methods](http://www.michaelminella.com/testing/how-to-mock-static-methods.html).

# Less mocking
 
Now I want to talk about some typical strategies to use less mocking. Again I'm not saying you should completely stop
using mocks. I just want to suggest strategies that will make testing simpler (and simple tests do not need much mocking).

To illustrate these strategies I created an interface to fetch stock price quotes given a stock symbol.

```java
public interface Ticker {

  double getQuote(String symbol) throws IOException;
  
}
```
 
We are going to have a simple implementation as well. It is absolutely not the way to implement this kind of
functionality in production code. I just wanted to show a more or less real life example.

```java
public class SimpleTicker implements Ticker {

  private final HtmlCleaner htmlCleaner;

  public SimpleTicker(HtmlCleaner htmlCleaner) {
    this.htmlCleaner = htmlCleaner;
  }

  @Override
  public double getQuote(String symbol) throws IOException {
    URL url = new URL("http://finance.yahoo.com/q?s=" + symbol);
    String html = IOUtils.toString(url);
    TagNode rootNode = htmlCleaner.clean(html);
    TagNode tickerNode = rootNode.findElementByAttValue("class", "time_rtq_ticker", true, true);
    String text = tickerNode.getText().toString();
    return Double.parseDouble(text);
  }
}
```

The `getQuote` method is rather simple, but has some logic to test. Let's create a test for this method using the most
direct approach with mocking. From my past experience this is what a lot of software developers would actually do.

```java
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;
import static org.powermock.api.mockito.PowerMockito.mockStatic;

@RunWith(PowerMockRunner.class)
@PrepareForTest(IOUtils.class)
public class SimpleTickerTest {

  @Test
  public void getQuote() throws Exception {
    // create mocks
    mockStatic(IOUtils.class);
    HtmlCleaner htmlCleaner = mock(HtmlCleaner.class);
    TagNode rootNode = mock(TagNode.class);
    TagNode tickerNode = mock(TagNode.class);
    // set expectations
    URL url = new URL("http://finance.yahoo.com/q?s=LNKD");
    String html = "<html><span class='time_rtq_ticker'><span>116.25</span></span></html>";
    when(IOUtils.toString(url)).thenReturn(html);
    when(htmlCleaner.clean(html)).thenReturn(rootNode);
    when(rootNode.findElementByAttValue("class", "time_rtq_ticker", true, true)).thenReturn(tickerNode);
    when(tickerNode.getText()).thenReturn("116.25");
    // run actual code
    SimpleTicker ticker = new SimpleTicker(htmlCleaner);
    double quote = ticker.getQuote("LNKD");
    assertEquals(116.25, quote, 0.1);
  }
}
```

The resulting test code if rather bad and suffers from all issues of mocking described above. What can we do to make it
better?

## Separate logic and side effects

This one is my favorite. It goes back to [pure functions](https://en.wikipedia.org/wiki/Pure_function) and functional
programming.

The basic idea is to break the code under test into pure functions and functions with side effects. This makes testing
so much easier because pure functions do not need mocking. Let's rewrite the ticker class.

```java
public class GoodTicker implements Ticker {

  private final HtmlCleaner htmlCleaner;

  public GoodTicker(HtmlCleaner htmlCleaner) {
    this.htmlCleaner = htmlCleaner;
  }

  @Override
  public double getQuote(String symbol) throws IOException {
    URL url = constructUrl(symbol);
    String html = fetchUrl(url);
    return parseHtml(html);
  }

  @VisibleForTesting
  URL constructUrl(String symbol) throws MalformedURLException {
    return new URL("http://finance.yahoo.com/q?s=" + symbol);
  }

  @VisibleForTesting
  double parseHtml(String html) {
    TagNode rootNode = htmlCleaner.clean(html);
    TagNode tickerNode = rootNode.findElementByAttValue("class", "time_rtq_ticker", true, true);
    String text = tickerNode.getText().toString();
    return Double.parseDouble(text);
  }

  @VisibleForTesting
  String fetchUrl(URL url) throws IOException {
    return IOUtils.toString(url);
  }
}
```

As you can see above it is the same exact code but broken into more fine grained methods. That gives us the ability to
test all these methods separately. Which is a good thing because they represent different areas of logic. Thus the 
resulting tests are not only simpler but they are better too.
  
Here is how the quote page URL creation logic test may look like.

```java
  @Test
  public void constructUrl() throws Exception {
    GoodTicker ticker = new GoodTicker(null);
    URL url = ticker.constructUrl("LNKD");
    assertEquals(new URL("http://finance.yahoo.com/q?s=LNKD"), url);
  }
```

As you can see the testing code is really simple and requires absolutely no mocking.

One may say that my production code is shaped by the way it is tested. I think it is shaped in a good way. A good code 
is a code that is easy to read. And this new rewritten code is much easier to read and understand.
 
Another argument could be that I exposed some private methods to make the class more testable. This is true, but should
not be a real problem because all clients access this class by the `Ticker` interface. Additionally this issue is
mitigated by using the [@VisibleForTesting](https://google-collections.googlecode.com/svn/trunk/javadoc/com/google/common/annotations/VisibleForTesting.html)
annotation. The annotation together with a [Findbugs detector](https://writeoncereadmany.wordpress.com/2016/04/08/how-to-find-bugs-part-3-visiblefortesting/)
guarantees that the newly exposed methods are not called outside of the testing scope.

## Use real classes where appropriate

Instead of creating mocks real objects can be used as long as they are fast and have no side effects. In our example it
is totally fine to use a real instance of [HtmlCleaner](http://htmlcleaner.sourceforge.net/) to test HTML parsing logic.

```java
  @Test
  public void parseHtml() throws Exception {
    GoodTicker ticker = new GoodTicker(new HtmlCleaner());
    String html = "<html><span class='time_rtq_ticker'><span>116.25</span></span></html>";
    double quote = ticker.parseHtml(html);
    assertEquals(116.25, quote, 0.1);
  }
```


## Do not test everything

It turns out that after extracting all logic into separate methods and testing them independently there is often nothing
else left to test. Our main method `getQuote` only contains the high level sequence of operations. It is so simple that
requires no testing.

To make a decision regarding what is simple enough to have no tests we need to go back to the reasons we write unit
tests at all. I think it boils down to catching bugs (present or future). What kind of bugs / mistakes do we expect to
catch in a method as simple as the new refactored `getQuote`? I'd say it's unlikely to have problems there. It is simple
enough to leave it untested. Not testing it does not decrease overall test coverage.

Whether some code needs testing or not is a judgement call we need to make each time. What is better to create a complicated
test or leave a piece of simple code untested? There is no universal answer.

Let's say I did not convince you and you still want `getQuote` to be tested. There are two ways we can go about it:

1. Create an anonymous class extending the class under test and override the side-effect method.
2. Create a partial mock changing the behavior of the side-effect method.

Please see both methods implemented below.
 
```java
  @Test
  public void getQuoteOverriding() throws Exception {
    URL url = new URL("http://finance.yahoo.com/q?s=LNKD");
    String html = "<html><span class='time_rtq_ticker'><span>116.25</span></span></html>";
    GoodTicker ticker = new GoodTicker(new HtmlCleaner()) {
      @Override
      String fetchUrl(URL actualUrl) throws IOException {
        assertEquals(url, actualUrl);
        return html;
      }
    };
    double quote = ticker.getQuote("LNKD");
    assertEquals(116.25, quote, 0.1);
  }

  @Test
  public void getQuoteMocking() throws Exception {
    // create a partial mock
    GoodTicker ticker = spy(new GoodTicker(new HtmlCleaner()));
    // set expectations
    URL url = new URL("http://finance.yahoo.com/q?s=LNKD");
    String html = "<html><span class='time_rtq_ticker'><span>116.25</span></span></html>";
    when(ticker.fetchUrl(url)).thenReturn(html);
    // run actual code
    double quote = ticker.getQuote("LNKD");
    assertEquals(116.25, quote, 0.1);
  }
```

Both approaches are fine and pretty much identical. Depending on how comfortable is your team with magic of mocking
frameworks you can pick one or another.

The important part to notice here is that these tests duplicate existing tests for individual methods (same assertions).
This is the reason I suggested earlier that they are redundant.

## Create stubs/fakes for infrastructure components

It is beneficial to create stubs or fakes for infrastructural components that are reused a lot in the project. This will
save you from duplicated code setting up mocks in each test involving those components.

This strategy can not be used to test the ticker class itself. But we can create a stub for it instead (assuming this
ticker is used in many other parts of the project).
 
```java
public class StubTicker implements Ticker {

  private final Map<String, Double> quotes;

  public StubTicker(String symbol, double quote) {
    this(ImmutableMap.of(symbol, quote));
  }

  public StubTicker(Map<String, Double> quotes) {
    this.quotes = quotes;
  }

  @Override
  public double getQuote(String symbol) throws IOException {
    Double quote = quotes.get(symbol);
    if (quote == null) {
      throw new IOException("No quote is defined for symbol " + symbol);
    }
    return quote;
  }
}
```

With this class testing other classes that use a ticker requires no mocking.

```java
  @Test
  public void testSomething() throws Exception {
    Ticker ticker = new StubTicker("LNKD", 116.25);
    Something smthg = new Something(ticker);
    ...
  }
```

# Outro
 
As a reminder, a quote from [Mockito web site](http://site.mockito.org/#remember):

* Do not mock types you don’t own
* Don’t mock value objects
* Don’t mock everything
* Show love with your tests!