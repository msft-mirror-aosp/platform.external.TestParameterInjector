TestParameterInjector
=====================

[Link to Javadoc.](https://google.github.io/TestParameterInjector/docs/latest/)

## Introduction

`TestParameterInjector` is a JUnit4 test runner that runs its test methods for
different combinations of field/parameter values.

Parameterized tests are a great way to avoid code duplication between tests and
promote high test coverage for data-driven tests.

There are a lot of alternative parameterized test frameworks, such as
[junit.runners.Parameterized](https://github.com/junit-team/junit4/wiki/parameterized-tests)
and [JUnitParams](https://github.com/Pragmatists/JUnitParams). We believe
`TestParameterInjector` is an improvement of those because it is more powerful
and simpler to use.

[This blogpost](https://opensource.googleblog.com/2021/03/introducing-testparameterinjector.html)
goes into a bit more detail about how `TestParameterInjector` compares to other
frameworks used at Google.

## Getting started

To start using `TestParameterInjector` right away, copy the following snippet:

```java
import com.google.testing.junit.testparameterinjector.TestParameterInjector;
import com.google.testing.junit.testparameterinjector.TestParameter;

@RunWith(TestParameterInjector.class)
public class MyTest {

  @TestParameter boolean isDryRun;

  @Test public void test1(@TestParameter boolean enableFlag) {
    // ...
  }

  @Test public void test2(@TestParameter MyEnum myEnum) {
    // ...
  }

  enum MyEnum { VALUE_A, VALUE_B, VALUE_C }
}
```

And add the following dependency to your `.pom` file:

```xml
<dependency>
  <groupId>com.google.testparameterinjector</groupId>
  <artifactId>test-parameter-injector</artifactId>
  <version>1.4</version>
</dependency>
```

or see [this maven.org
page](https://search.maven.org/artifact/com.google.testparameterinjector/test-parameter-injector)
for instructions for other build tools.


## Basics

### `@TestParameter` for testing all combinations

#### Parameterizing a single test method

The simplest way to use this library is to use `@TestParameter`. For example:

```java
@RunWith(TestParameterInjector.class)
public class MyTest {

  @Test
  public void test(@TestParameter boolean isOwner) {...}
}
```

In this example, two tests will be automatically generated by the test framework:

-   One with `isOwner` set to `true`
-   One with `isOwner` set to `false`

When running the tests, the result will show the following test names:

```
MyTest#test[isOwner=true]
MyTest#test[isOwner=false]
```

#### Parameterizing the whole class

`@TestParameter` can also annotate a field:

```java
@RunWith(TestParameterInjector.class)
public class MyTest {

  @TestParameter private boolean isOwner;

  @Test public void test1() {...}
  @Test public void test2() {...}
}
```

In this example, both `test1` and `test2` will be run twice (once for each
parameter value).

#### Supported types

The following examples show most of the supported types. See the `@TestParameter` javadoc for more details. 

```java
// Enums
@TestParameter AnimalEnum a; // Implies all possible values of AnimalEnum
@TestParameter({"CAT", "DOG"}) AnimalEnum a; // Implies AnimalEnum.CAT and AnimalEnum.DOG.

// Strings
@TestParameter({"cat", "dog"}) String animalName;

// Java primitives
@TestParameter boolean b; // Implies {true, false}
@TestParameter({"1", "2", "3"}) int i;
@TestParameter({"1", "1.5", "2"}) double d;

// Bytes
@TestParameter({"!!binary 'ZGF0YQ=='", "some_string"}) byte[] bytes;
```

For non-primitive types (e.g. String, enums, bytes), `"null"` is always parsed as the `null` reference.

#### Multiple parameters: All combinations are run

If there are multiple `@TestParameter`-annotated values applicable to one test
method, the test is run for all possible combinations of those values. Example:

```java
@RunWith(TestParameterInjector.class)
public class MyTest {

  @TestParameter private boolean a;

  @Test public void test1(@TestParameter boolean b, @TestParameter boolean c) {
    // Run for these combinations:
    //   (a=false, b=false, c=false)
    //   (a=false, b=false, c=true )
    //   (a=false, b=true,  c=false)
    //   (a=false, b=true,  c=true )
    //   (a=true,  b=false, c=false)
    //   (a=true,  b=false, c=true )
    //   (a=true,  b=true,  c=false)
    //   (a=true,  b=true,  c=true )
  }
}
```

If you want to explicitly define which combinations are run, see the next
sections.

### Use a test enum for enumerating more complex parameter combinations

Use this strategy if you want to:

-   Explicitly specify the combination of parameters
-   or your parameters are too large to be encoded in a `String` in a readable
    way

Example:

```java
@RunWith(TestParameterInjector.class)
class MyTest {

  enum FruitVolumeTestCase {
    APPLE(Fruit.newBuilder().setName("Apple").setShape(SPHERE).build(), /* expectedVolume= */ 3.1),
    BANANA(Fruit.newBuilder().setName("Banana").setShape(CURVED).build(), /* expectedVolume= */ 2.1),
    MELON(Fruit.newBuilder().setName("Melon").setShape(SPHERE).build(), /* expectedVolume= */ 6);

    final Fruit fruit;
    final double expectedVolume;

    FruitVolumeTestCase(Fruit fruit, double expectedVolume) { ... }
  }

  @Test
  public void calculateVolume_success(@TestParameter FruitVolumeTestCase fruitVolumeTestCase) {
    assertThat(calculateVolume(fruitVolumeTestCase.fruit))
        .isEqualTo(fruitVolumeTestCase.expectedVolume);
  }
}
```

The enum constant name has the added benefit of making for sensible test names:

```
MyTest#calculateVolume_success[APPLE]
MyTest#calculateVolume_success[BANANA]
MyTest#calculateVolume_success[MELON]
```

### `@TestParameters` for defining sets of parameters

You can also explicitly enumerate the sets of test parameters via a list of YAML
mappings:

```java
@Test
@TestParameters({
  "{age: 17, expectIsAdult: false}",
  "{age: 22, expectIsAdult: true}",
})
public void personIsAdult(int age, boolean expectIsAdult) { ... }
```

The string format supports the same types as `@TestParameter` (e.g. enums). See
the `@TestParameters` javadoc for more info.

`@TestParameters` works in the same way on the constructor, in which case all
tests will be run for the given parameter sets.

## Advanced usage

### Dynamic parameter generation for `@TestParameter`

Instead of providing a list of parsable strings, you can implement your own
`TestParameterValuesProvider` as follows:

```java
@Test
public void matchesAllOf_throwsOnNull(
    @TestParameter(valuesProvider = CharMatcherProvider.class) CharMatcher charMatcher) {
  assertThrows(NullPointerException.class, () -> charMatcher.matchesAllOf(null));
}

private static final class CharMatcherProvider implements TestParameterValuesProvider {
  @Override
  public List<CharMatcher> provideValues() {
    return ImmutableList.of(CharMatcher.any(), CharMatcher.ascii(), CharMatcher.whitespace());
  }
}
```

Note that `provideValues()` dynamically construct the returned list, e.g. by
reading a file. There are no restrictions on the object types returned, but note
that `toString()` will be used for the test names.

### Dynamic parameter generation for `@TestParameters`

Instead of providing a YAML mapping of parameters, you can implement your own
`TestParametersValuesProvider` as follows:

```java
@Test
@TestParameters(valuesProvider = IsAdultValueProvider.class)
public void personIsAdult(int age, boolean expectIsAdult) { ... }

static final class IsAdultValueProvider implements TestParametersValuesProvider {
  @Override public ImmutableList<TestParametersValues> provideValues() {
    return ImmutableList.of(
      TestParametersValues.builder()
        .name("teenager")
        .addParameter("age", 17)
        .addParameter("expectIsAdult", false)
        .build(),
      TestParametersValues.builder()
        .name("young adult")
        .addParameter("age", 22)
        .addParameter("expectIsAdult", true)
        .build()
    );
  }
}
```
