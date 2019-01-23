---
title: Speeding Up and Reducing Memory Usage in Dropwizard and WireMock Integration Tests
---

### {{ page.date | date: "%B %Y" }}

# {{page.title}}

## Problem

A project I was working on was suffering from flaky automated tests. The project was a Dropwizard service that was seeing frequent random failures, the failure originated from the service's client timing out when calling a mocked downstream service. The tests that were failing were integration tests, integration test is a bit of an overloaded term, what I mean by an integration test in this project is a test that runs against an actual instance of the application started using DropwizardAppRule, and against mock instances of downstream services, which are mocked using WireMock.

## Cause

I observed that the tests would start off running quickly and as the suite progressed would suddenly slow down. After investigation it looked like the tests were slowing down as they hit the JVMs memory limit. It appeared as if the tests were struggling to free memory and performing frequent garbage collection, leading to a slowdown. I didn't get to the bottom of why the tests were struggling to free memory, in the end I avoided this problem altogether. The slowdown would randomly cause some requests to take longer than the 500ms configured as a request timeout. The tests were consuming a lot of memory because each test class would start its own Dropwizard instance along with one or two WireMock instances. The entire test suite had over 100 integration test classes.

## Solution

I drastically reduced memory usage by sharing Dropwizard and WireMock instances between integration tests. This may not be desirable for all projects, if your test application has internal state, this would make your tests dependent on each other, this was not the case for my project.

The first step was to create a test suite, I used the [ClasspathSuite](https://github.com/takari/takari-cpsuite) project to create a suite that encompassed all of the integration tests in the project. When the suite is run it will first start up a Dropwizard instance and WireMock instance for each downstream service, then it will run all test classes in the `example.service.integration` package.

```
@RunWith(ClasspathSuite.class)
@ClasspathSuite.ClassnameFilters({"example.service.integration.*"})
public class IntegrationTests {

    @ClassRule
    public static final DropwizardAppRule EXAMPLE_SERVICE = new DropwizardAppRule<Configuration>(ExampleService.class);

    @ClassRule
    public static final WireMockClassRule MOCK_DOWNSTREAM_SERVICE = new WireMockClassRule();
}
```

The next step is to create tests to be run as part of the suite. Each test will share the same Dropwizard instance and WireMock instance.

```
public class ResourceTest {
    
    @ClassRule
    public static final DropwizardAppRule EXAMPLE_SERVICE = IntegrationTests.EXAMPLE_SERVICE;

    @ClassRule
    public static final WireMockClassRule MOCK_DOWNSTREAM_SERVICE = IntegrationTests.MOCK_DOWNSTREAM_SERVICE;

    @Test
    public void testExampleTestWithDownstreamService() throws Exception {
        // given
        MOCK_DOWNSTREAM_SERVICE.stubFor(
            get(urlPathEqualTo("/downstream")).willReturn(aResponse().withStatus(OK_200))
        );

        // when
        Response response = EXAMPLE_SERVICE.client().target("/endpoint").request().get();

        // then
        assertThat(response.getStatus()).isEqualTo(OK_200);
    }

    ...

```

The final step is to exclude individual integration tests from being run by Maven in `pom.xml`, as they will be run as part of the suite.

```

    ...

    <build>
        </plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <excludes>
                        <!-- These tests are run as part of the IntegrationTests suite.
                             Exclude them to ensure they run through the suite. -->
                        <exclude>
                            example.project.integration.**
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```