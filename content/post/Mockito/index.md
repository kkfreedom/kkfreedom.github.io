+++
author = "Leo Feng"
title = "Mockito"
date = "2019-03-11"
description = "Java 单元测试框架 Mockito 学习笔记"
tags = [
    "Java",
    "编程",
]
categories = [
    "编程",
]
series = ["Themes Guide"]
aliases = ["migrate-from-jekyl"]
image = "pawel-czerwinski-8uZPynIu-rQ-unsplash.jpg"
+++

# spy() @spy 和 mock() @mock的区别
 简单理解： 
-  spy是partial mock，spy需要初始化，如果不手动初始化，mocktio默认调用**无参构造**初始化。
-  如果Method没有被mock, spy默认调用真实方法，mock不会，对于有返回值的spy返回真实的返回值，mock返回null.
-   对于spy，通常建议是用```doReturn|Answer|Throw()```方式打桩（stubbing），否则可能由于调用真实的方法而抛异常。

  
```Java
    @Test
    public void testSpy()
    {
        // another option to spies a real object instead of annotation
        List<String> list = spy(new ArrayList<>(1));
        // java.lang.IndexOutOfBoundsException
        // when(list.get(0)).thenReturn("three");
        // assertEquals("three", list.get(0));

        // right way to mock
        doReturn("three").when(list).get(0);
        assertEquals("three", list.get(0));
        verify(list, atLeast(1)).get(0);

        // can pass, so spies is actually a copy of real instance
        assertTrue(list.add("one"));

        assertEquals("three", list.get(0));
        System.out.println(list.size()); // 1
        list.forEach(System.out::println); // one
    }

    @Test
    public void testMock()
    {
        List list = mock(List.class);
        // below both work fine
        when(list.get(0)).thenReturn("three");
        assertEquals("three", list.get(0));

        doReturn("three").when(list).get(0);
        assertEquals("three", list.get(0));

        list.add("one");
        // assertTrue(list.add("one")); // java.lang.AssertionError

        assertEquals("three", list.get(0));
        System.out.println(list.size()); // 0
        list.forEach(System.out::println); //
    }
```

但是，```doReturn```也有缺陷，```doReturn```可以接任意objects，所以是No type safety, [参考]( https://sangsoonam.github.io/2019/02/04/mockito-doreturn-vs-thenreturn.html)

# Mock一个object实际上会默认mock所有的方法
如果方法没有被打桩，有返回值的默认返回null, 返回boolean值的默认返回false，返回int/integer的默认返回0，void方法默认do nothing。
  
[以下例子引用自](https://www.cnblogs.com/softidea/p/4204389.html)
```java
    @Test
    public void spy_Procession_Demo() {
        Jack spyJack = spy(new Jack());
        //使用spy的桩实现实际还是会调用stub的方法，只是返回了stub的值
        when(spyJack.go()).thenReturn(false); // I say go go go!!
        assertFalse(spyJack.go()); 
        verify(spyJack).go();

        //不会调用stub的方法
        doReturn(false).when(spyJack).go();
        assertFalse(spyJack.go()); // nothing print
    }

    @Test
    public void callRealMethodTest() {
        Jerry jerry = mock(Jerry.class);

        doCallRealMethod().when(jerry).goHome();
        doCallRealMethod().when(jerry).doSomeThingB();
        // doCallRealMethod().when(jerry).returnInteger();

        jerry.goHome(); // good day

        assertEquals(0, jerry.returnInteger());
        verify(jerry).returnInteger();
        verify(jerry).doSomeThingA();
        verify(jerry).doSomeThingB();
    }

class Jack {
    public boolean go() {
        System.out.println("I say go go go!!");
        return true;
    }
}

class Jerry {
    public void goHome() {
        doSomeThingA();
        doSomeThingB();
    }

    // real invoke it.
    public void doSomeThingB() {
        System.out.println("good day");

    }

    // auto mock method by mockito
    public void doSomeThingA() {
        System.out.println("you should not see this message.");

    }

    public int returnInteger()
    {
        return 1;
    }
}
```
# doAnswer
```java
    @Test
    public void testDoAnswer()
    {
        Jack jack = mock(Jack.class);
        doAnswer((Answer<String>) inv -> {
            String name = inv.getArgument(0, String.class);
            name += "_suffix";
            return name;
        }).when(jack).rename(anyString());

        String name = "Leo";
        assertEquals(name + "_suffix", jack.rename(name));
    }
class Jack {
    private String name;

    public String rename(String name)
    {
        if ("Jack".equals(name)) {
            return "Jackson";
        }
        return name;
    }
    public boolean go() {
        System.out.println("I say go go go!!");
        return true;
    }
}
```
use spy then test doAnswer, 由以下例子可以看出对于spy()，```doAnswer```不会调用真实方法，而```thenAnswer```会调用真实方法。
```java
    @Test
    public void testDoAnswer()
    {
        Jack jack = spy(Jack.class);
        doAnswer((Answer<String>) inv -> {
            String name = inv.getArgument(0, String.class);
            name += "_suffix";
            return name;
        }).when(jack).rename(anyString());
        
        String name = "Leo";
        assertEquals(name + "_suffix", jack.rename(name)); // passed but print nothing
    }

    @Test
    public void testDoAnswer()
    {
        Jack jack = spy(Jack.class);

        when(jack.rename("Jack")).thenAnswer((Answer<String >) invocation -> {
            String name = invocation.getArgument(0, String.class);
            return name + "_suffix";
        }); // print: real invoke!!

        String name = "Leo";
        assertEquals(name + "_suffix", jack.rename(name)); 
        // tests failed, print real invoke!! expeced: Leo_suffix Actual: Leo
    }

class Jack {
    private String name;

    public String rename(String name)
    {
        System.out.println("real invoke!!");
        if ("Jack".equals(name)) {
            return "Jackson";
        }
        return name;
    }
}
```

# ArgumentCaptor
使用*ArgumentCaptor*可以断言方法的参数
```java
    @Test
    public void testArgCaptor()
    {
        Jack jack = mock(Jack.class);
        ArgumentCaptor<String > argument = ArgumentCaptor.forClass(String.class);
        String rename = jack.rename("Jack");
        verify(jack).rename(argument.capture());
        assertEquals("should be Jack", "Jack", argument.getValue());
    }
```
- 值得注意的是，通常*ArgumentCaptor*使用在verfify中而不是在打桩时候，Mockito官方文档给出的例子是，如果想在打桩时候使用**ArgumentMatcher**更适合。
```java
   ArgumentCaptor<Person> argument = ArgumentCaptor.forClass(Person.class);
   verify(mock).doSomething(argument.capture());
   assertEquals("John", argument.getValue().getName());
```
>**Warning:** it is recommended to use ArgumentCaptor with verification **but not** with stubbing. Using ArgumentCaptor with stubbing may decrease test readability because captor is created outside of assert (aka verify or 'then') block. Also it may reduce defect localization because if stubbed method was not called then no argument is captured.
In a way ArgumentCaptor is related to custom argument matchers (see javadoc for [`ArgumentMatcher`](https://javadoc.io/static/org.mockito/mockito-core/3.7.7/org/mockito/ArgumentMatcher.html "interface in org.mockito") class). Both techniques can be used for making sure certain arguments were passed to mocks. However, ArgumentCaptor may be a better fit if:
>*   custom argument matcher is not likely to be reused
>*   you just need it to assert on argument values to complete verification
>Custom argument matchers via [`ArgumentMatcher`](https://javadoc.io/static/org.mockito/mockito-core/3.7.7/org/mockito/ArgumentMatcher.html "interface in org.mockito") are usually better for stubbing.

以下引用的例子用来解释为什么打桩时候使用**ArgumentMatcher**更适合。
>Assuming the following method to test:
```public boolean doSomething(SomeClass arg);```
>Mockito documentation says that you should not use captor in this way:
```when(someObject.doSomething(argumentCaptor.capture())).thenReturn(true);```
``` assertThat(argumentCaptor.getValue(), equalTo(expected));```
>Because you can just use matcher during stubbing:
```when(someObject.doSomething(eq(expected))).thenReturn(true);```
