---
title: Arquillian i Java EE 8 - przykład zaawansowany - cz. 1 
date: 2019-02-22
lang: pl
ref: arq_jee8_adv1
summary: W sieci można bardzo łatwo znaleźć proste przykłady jak korzystać z Arquilliana do wykonywania testów integracyjnych. Jednak jeśli ktoś chce przygotować sobie pełne testy integracyjne naszej aplikacji webowej -- takiej jaka jest tworzona na koniec -- to już nie jest takie proste. Trzeba złożyć wiedzę dotyczącą co najmniej TestNG/JUnit, Arquilliana i ShrinkWrapa, które są rozbite na wiele małych dokumentów. Postaram się takie bardziej zaawansowane przykłady złożyć razem.
---
# Arquillian i Java EE 8 cz. 1
W sieci można bardzo łatwo znaleźć proste przykłady jak korzystać z Arquilliana do wykonywania testów integracyjnych. Jednak jeśli ktoś chce przygotować sobie pełne testy integracyjne aplikacji webowej -- pełnego archiwum WAR -- to już nie jest takie proste. Trzeba złożyć wiedzę dotyczącą co najmniej: TestNG/JUnit, Arquilliana i ShrinkWrapa, które są rozbite na wiele małych dokumentów. Do tego te dokumenty nie zawsze są ze sobą zgodne.

Dlatego postanowiłem podzielić się zaawansowanym przykładem, który pokazuje jak użyć Arquilliana z TestNG w celu przetestowania  na WildFly'u pełnej aplikacji webowej. A w kolejnych częściach integracja i testowanie aplikacji wielomodułowych (aplikacji korzystającej z usług/mikroserwisów REST) oraz jak korzystając z CDI można podmienić na potrzeby testów konfigurację różnych zasobów (Persistence itp.) czy emulowanie uruchamiania metod EJB jako określone role.

## Środowisko testowe używane w przykładach
* WildFly 14 -- serwer aplikacji Java EE 8
* TestNG 6.14 -- framework do testów
* Arquillian 1.4 -- framework do testów integracyjnych
* Shrinkwrap Resolver 3.1 -- framework do tworzenia archiwów testowych w oparciu o projekty Mavena

### Dlaczego TestNG
W większości przykładów jako framework do wykonywania testów używany jest JUnit. Osobiście uważam, że TestNG jest nowocześniejsze i bardziej zaawansowane. Pozwala na deklarowanie zależności wykonywania testów, łatwe grupowanie metod testowych do tego zawiera wiele innych przydatnych w bardziej zaawansowanych przypadkach narzędzi i API. Należy też zwrócić uwagę na to, że w momencie gdy pracowałem na przykładami Arquillian, działał wyłącznie z JUnit 4. Jak ostatnio sprawdzałem to twórcy Arquilliana uważali, że nie jest możliwe napisania wsparcia na obecne wersje JUnit 5.

## Co testujemy?
Zakładam, że w moich przykładach będę chciał przetestować metody biznesowe zawarte w EJB w aplikacji webowej, czyli że artefaktem projektu, który podlega testom będzie plik WAR. Projekt budowany jest Mavenem. W 1. części będziemy testować tylko zwykłe EJB z wewnątrz projektu z tym EJB.

## Zależności testowe w projekcie
Aby móc użyć Arquilliana z TestNG potrzebujemy dodać do `pom.xml` odpowiednie zależności.

Na początek importujemy BOM-y (Bill Of Materials) dla Shrinkwrapa i Arquilliana jako `<dependencyManagement>`. Pozwala to zachować (względnie<sup>[1](#fn1)</sup>) spójny zestaw wersji wszytkich bibliotek -- wersje bibliotek zadeklarowane przez twórców frameworków.

Kolejność importu jest ważna. Ten który jest podany wcześniej ma priorytet. Chcemy użyć nowszej wersji Shrinkwrapa, więć on jest na początku.
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.jboss.shrinkwrap.resolver</groupId>
            <artifactId>shrinkwrap-resolver-bom</artifactId>
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

Dalej dodajemy zależność od TestNG oraz frameworków Arquillian i Shrinkwrap. W różnych tutorialach jest sugestia aby zaimportować komplet zależności tych dwóch frameworków ale moje doświadczenie wskazuje, że lepiej dodać tylko to co rzeczywiście potrzebujemy. W tym przypadku będzie to:
```xml
<dependencies>
    <!-- TestNG -->
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>6.14.3</version>
        <scope>test</scope>
    </dependency>

    <!-- Zależności Arquillian TestNG. Jeśli ktoś chce użyć JUnit to musi użyć odpowiednik dla JUnit. -->
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

    <!-- Zależności ShrinkWrap -->
    <dependency>
        <groupId>org.jboss.shrinkwrap.resolver</groupId>
        <artifactId>shrinkwrap-resolver-api</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.shrinkwrap.resolver</groupId>
        <artifactId>shrinkwrap-resolver-api-maven</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.shrinkwrap.resolver</groupId>
        <artifactId>shrinkwrap-resolver-api-maven-archive</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.shrinkwrap.resolver</groupId>
        <artifactId>shrinkwrap-resolver-api-maven-embedded</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.shrinkwrap.resolver</groupId>
        <artifactId>shrinkwrap-resolver-spi</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.shrinkwrap.resolver</groupId>
        <artifactId>shrinkwrap-resolver-spi-maven</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.shrinkwrap.resolver</groupId>
        <artifactId>shrinkwrap-resolver-impl-maven</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.shrinkwrap.resolver</groupId>
        <artifactId>shrinkwrap-resolver-impl-maven-archive</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.jboss.shrinkwrap.resolver</groupId>
        <artifactId>shrinkwrap-resolver-impl-maven-embedded</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Zależność dla kontenera zewnętrznego WildFly jako środowiska wykonywania testów -->
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

## EJB do przetestowania
Na potrzeby przykładu w apliakcji testowej utworzymy proste sesyjne bezstanowe EJB.
```java
@Stateless
public class BeanForTesting implements Serializable {

    private static final long serialVersionUID = 1L;

    /**
     * Prosta metoda dla przykładu o testowwaniu EJB.
     *
     * @return 33
     */
    public int someGreatMethod() {
        return 33;
    }
}
```

## Klasa testująca
Klasa testująca, która korzysta z Arquilliana musi mieć następujące cechy:
* dziedziczyć po `org.jboss.arquillian.testng.Arquillian`;
* zawierać statyczną metodę oznaczoną adnotacją `@Deployment`. To jest metoda, która tworzy archiwum, które potem będzie wrzucone na serwer i na którym będą wykonywane testy.

Do klasy testowej możemy wstrzyknąć bez problemu m. in. EJB, Entity Managera itd. Tym razem wstrzykniemy tylko EJB.

### Tworzenie archiwum testowego
Archiwum do testowania można utworzyć całkowicie ręcznie -- dodając wszystkie klasy i zasoby jakie potrzebujemy w teście. Ja jednak przyjmuję, że chcę aby testy wykonywały się w środowisku jak najbliższym docelowemu. Zatem w moim przykładzie archiwum testowe tworzę budując projekt na podstawie pliku projektu Maven -- wtedy na serwerze znajdzie się archiwum prawie takie samo jak archiwum wynikowe. Prawie takie samo, ponieważ będą do niego dodane:
* klasa testowa (dodaje się automatycznie);
* zależności runtime TestNG (dodawane ręcznie).

```java
public class BeanForTestingNGTest extends Arquillian {

    @EJB
    private BeanForTesting anEjb;

    /**
     * To opisana wyżej metoda tworząca archiwum testowe.
     */
    @Deployment
    public static WebArchive createDeployment() {
        /* Tworzę obiekt wbudowanego Mavena  */
        var maven = EmbeddedMaven
                .forProject("pom.xml") //używamy pom.xml z bieżącego katalogu (katalog budowania projektu)
                .useLocalInstallation();

        /* A to obiekt pozwalający szukać biblioteki w repozytoriach Mavena. */
        var resolver = Maven.configureResolver().loadPomFromFile("pom.xml");

        // Budujemy nasz "samotestujący" się projekt
        WebArchive a = (WebArchive) maven
                .setGoals("package")
                .skipTests(true)
                .build()
                .getDefaultBuiltArchive();

        //Pobieramy z repozytorium Mavena i dodajemy zależności TestNG
        File testNgLib1 = resolver.resolve("org.testng:testng:jar:6.14.3").withoutTransitivity().asSingleFile();
        File testNgLib2 = resolver.resolve("com.beust:jcommander:jar:1.72").withoutTransitivity().asSingleFile();
        return a.addAsLibraries(testNgLib1, testNgLib2);
    }

    /**
     * Ten test ma się nie udać.
     */
    @Test
    public void testSomeGreatMethodFail() {
        assertEquals(anEjb.someGreatMethod(), 22);
    }

    /**
     * A ten test ma się udać.
     */
    @Test
    public void testSomeGreatMethodPass() {
        assertEquals(anEjb.someGreatMethod(), 33);
    }

}
```

Aby wykonać test musimy najpierw uruchomić nasz serwer WildFly. Jeśli jest uruchomiony bezpośrednio w systemie to wszystko powinno zadziałać bez dodatkowej konfiguracji. Jeśli jest izolowany (np. w kontenerze dockera) to potrzebujemy dodać jeszcze jeden plik w głównym katalogu zasobów testowych: `arquillian.xml`, w którym musimy podać informacje jak się podłączyć do serwera.

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

## Podsumowanie
Cały przykład, jako projekt Mavena, jest też dostępny [tu](https://github.com/mpiotrowski-im/blog-examples/tree/master/ArquillianAdvanced).

Na początek to tyle. W następnych częściach poruszę tematy testowania wielu modułów na raz, symulowania wykonwyania kodu w określonych rolach itp.

Ten przykład to samotestujący się projekt. W rzeczywistości warto rozważyć wydzielenie testów integracyjnych do oddzielnego projektu.

##### Przypisy
<a name="fn1"></a><sup>1</sup> Względnie ponieważ jeśli ktokolwiek próbował wykonać sprawdzenie spójności wersji zależności w większym projekcie na pewno odkrył, że wiele zewnętrznych bibliotek i frameworków jest niespójna sama ze sobą...

<!--
Copyright 2019 Michał Piotrowski

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0
-->
