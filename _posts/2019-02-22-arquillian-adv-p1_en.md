---
title: Java EE 8 testing with Arquillian (part 1) 
date: 2019-02-22
lang: en
ref: arq_jee8_adv1
summary: There are lots of simple examples how to use Arquillian, but there are not so many easily available more complex examples. I decided to share some more advanced Arquillian exmaples which join knowledge from many sources TestNG/JUnit, Arquillian and Shrinkwrap tutorials and manuals.
---
# Java EE 8 testing with Arquillian (part 1)
There are lots of simple examples how to use Arquillian, but there are not so many easily available more complex examples. I decided to share some more advanced Arquillian exmaples which join knowledge from many sources: TestNG/JUnit, Arquillian and Shrinkwrap tutorials and manuals.

The first part will focus on creating self testing web archive (WAR) application using TestNG, Arquillian and ShringWrap Resolver to create testing archive almost identical as the created web application package. In the following parts I plan to adress testing multimodule application, REST services, simulating running tests as specified JAAS roles etc.


## Testing environment
* WildFly 14 -- Java EE 8 application server
* TestNG 6.14 -- testing framework
* Arquillian 1.4 -- integration testing framework
* Shrinkwrap Resolver 3.1 -- framework used to create archives used for testing

### Why I choose TestNG
JUnit is used as a testing framework in most examples I have found. Personally I prefer TestNG as a more advanced and more configurable testing framework. TestNG allows to declare dependencies between tests, easy test grouping and has many more useful features. Moreover when I worked on my Arquillian examples, JUnit 5 was not supported in Arquillian framework.

## What I will be testing?
In first part I assume that I need to test business method of an EJB contained in a web app (WAR archive). I want the test to be executed in an environment as close to a production environment as possible so I want my testing archive to be almost identical to project's target archive. The project is a Maven project.

## Test dependencies required in `pom.xml`
To use Arquillian we need to add some dependencies to our `pom.xml` file.

At the beginning we import BOMs (Bill Of Materials) for Shrinkwrap and Arquillian. BOMs are placed in `<dependencyManagement>`. It allows us to use (almost <sup>[1](#fn1)</sup>) consistent artifact versions needed by the frameworks.

The order of imports in `<dependencyManagement>` section is import. Earlier ones have greater priority then later ones. In our case we want to use newer Shrinkwrap than those declared by Arquillian BOM so the Shrinkwrap BOM is first.
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.jboss.Shrinkwrap.resolver</groupId>
            <artifactId>Shrinkwrap-resolver-bom</artifactId>
            <version>3.1.3</version>
            <scope>import</scope>
            <type>pom</type>
        </dependency>

        <dependency>
            <groupId>org.jboss.arquillian</groupId>
            <artifactId>arquillian-bom</artifactId>
            <version>1.4.0.Final</version>
            <scope>import</scope>
            <type>pom</type>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Next we need to declare test dependencies for TestNG, Arquillian and Shrinkwrap. Some tutorials suggest to import a whole suite of Arquillian and Shrinkwrap frameworks' dependencies but I have found out that it is better to add only those dependencies you really use and need. There will be less library conflicts this way.

Dependencies in our case are:

```xml
<dependencies>
    <!-- TestNG -->
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>6.14.3</version>
        <scope>test</scope>
    </dependency>

    <!-- Arquillian TestNG artifact. For JUnit one have to use corresponding one. -->
    <dependency>
        <groupId>org.jboss.arquillian.testng</groupId>
        <artifactId>arquillian-testng-container</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.arquillian.test</groupId>
        <artifactId>arquillian-test-spi</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Shrinkwrap -->
    <dependency>
        <groupId>org.jboss.Shrinkwrap.resolver</groupId>
        <artifactId>Shrinkwrap-resolver-api</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.Shrinkwrap.resolver</groupId>
        <artifactId>Shrinkwrap-resolver-api-maven</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.Shrinkwrap.resolver</groupId>
        <artifactId>Shrinkwrap-resolver-api-maven-archive</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.Shrinkwrap.resolver</groupId>
        <artifactId>Shrinkwrap-resolver-api-maven-embedded</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.Shrinkwrap.resolver</groupId>
        <artifactId>Shrinkwrap-resolver-spi</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.Shrinkwrap.resolver</groupId>
        <artifactId>Shrinkwrap-resolver-spi-maven</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.Shrinkwrap.resolver</groupId>
        <artifactId>Shrinkwrap-resolver-impl-maven</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.Shrinkwrap.resolver</groupId>
        <artifactId>Shrinkwrap-resolver-impl-maven-archive</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.Shrinkwrap.resolver</groupId>
        <artifactId>Shrinkwrap-resolver-impl-maven-embedded</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- I want to use external WildFly server as a test environment so I need appropriate remote container artifact -->
    <dependency>
        <groupId>org.wildfly.arquillian</groupId>
        <artifactId>wildfly-arquillian-container-remote</artifactId>
        <version>2.1.1.Final</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.arquillian.core</groupId>
        <artifactId>arquillian-core-api</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## An incredibly simple EJB
For our test example I have create a very simple EJB.
```java
@Stateless
public class BeanForTesting implements Serializable {

    private static final long serialVersionUID = 1L;

    /**
     * Simple method we will be testing.
     *
     * @return 33
     */
    public int someGreatMethod() {
        return 33;
    }
}
```

## A test class
Every test which uses Arquillian framework with TestNG has to:
* extend `org.jboss.arquillian.testng.Arquillian`;
* have static method annotated with `@Deployment` -- the method responsible for creating the test archive which will be deployed on the testing server.

When using Arquillian it is possible to inject EJB, EntityManager into test class. In my first example I inject only EJB.

### Creating the test archive
It is possible to assemble a test archive manually using Shrinkwrap by adding every class required for the test. In my example I want my tests to be executed in an environment as close to a production environment as possible. It is the reason why I assemble the test archive by building project inside deployment method using projects `pom.xml` and Shrinkwrap.

Additionally a test archive has to contain:
* test class (it is added automatically);
* runtime test dependencies -- in this case TestNG artifacts. The runtime test dependencies have to be added manually.

```java
public class BeanForTestingNGTest extends Arquillian {

    @EJB
    private BeanForTesting anEjb;

    /**
     * Archive assembly method.
     */
    @Deployment
    public static WebArchive createDeployment() {
        /* Creating embedded Maven object  */
        var maven = EmbeddedMaven
                .forProject("pom.xml") //use pom in CWD (which is project dir in maven build)
                .useLocalInstallation();

        /* Resolver for downloading artifacts from Maven repositories. */
        var resolver = Maven.configureResolver().loadPomFromFile("pom.xml");

        // Now we build our self testing project.
        WebArchive a = (WebArchive) maven
                .setGoals("package")
                .skipTests(true)
                .build()
                .getDefaultBuiltArchive();

        //Adding TestNG runtime dependencies
        File testNgLib1 = resolver.resolve("org.testng:testng:jar:6.14.3").withoutTransitivity().asSingleFile();
        File testNgLib2 = resolver.resolve("com.beust:jcommander:jar:1.72").withoutTransitivity().asSingleFile();
        return a.addAsLibraries(testNgLib1, testNgLib2);
    }

    /**
     * The test which should fail so we see that testing works.
     */
    @Test
    public void testSomeGreatMethodFail() {
        assertEquals(anEjb.someGreatMethod(), 22);
    }

    /**
     * The test which should pass.
     */
    @Test
    public void testSomeGreatMethodPass() {
        assertEquals(anEjb.someGreatMethod(), 33);
    }

}
```

To run our test we have to start WildFly server. If WildFly is running outside any container, with default port settings, everything should work out of the box. If WildFly is running in a container (e. g. docker) we need `arquillian.xml` configuration file in main test resources directory. The configuration file has to have information how to connect to management port in WildFly.

```xml
<arquillian xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="http://jboss.org/schema/arquillian"
    xsi:schemaLocation="http://jboss.org/schema/arquillian http://www.jboss.org/schema/arquillian/arquillian_1_0.xsd">

    <container qualifier="wildfly-remote" default="true">
        <configuration>
            <property name="managementAddress">127.0.0.1</property>
            <property name="managementPort">9990</property>
            <property name="username">admin</property>
            <property name="password">admin</property>
        </configuration>
    </container>

</arquillian>
```

## Summary
The full example as a Maven project is available [here](https://github.com/mpiotrowski-im/blog-examples/tree/master/ArquillianAdvanced).

In real world it can be considered to create integration tests as a separate project.

In following parts I will try to cover multimodule application testing, testing using JAAS roles etc.

##### Footnotes
<a name="fn1"></a><sup>1</sup> Developers running dependency convergence test know that many frameworks do not have consistent artifact versions.

<!--
Copyright 2019 Michał Piotrowski

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0
-->
