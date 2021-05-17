---
title: "FluentLenium et Cucumber JVM sont sur un bateau..."
date: 2013-03-14 16:39:36 +0100
comments: true
tags: 
- fluentlenium
- intégration continue
- java
- maven
- selenium
---
![left-small](http://4.bp.blogspot.com/-R3ORi15Ei3Y/UTmwoq5A2II/AAAAAAAAA3o/uaIMSSZlq38/s1600/logo.png)

Dans un [article précédent](/2013/03/demarrer-une-webapp-en-mode-embedded.html), j'avais abordé comment il était possible de démarrer une application web dans un conteneur de Servlet de manière embedded au sein de la phase integration de Maven. Bien sûr, cela n'a pas été fait que pour l'exercice de style et il y avait une petite idée derrière : pouvoir exécuter des tests d'acceptance en mode boite noire sur l'application.

Pour faire les tests d'acceptance, le choix de Cucumber JVM a été fait afin de permettre l'expression de tests d'acceptance avec une sémantique utilisant le pattern __Given/When/Then__ mais également afin de permettre à des non développeurs de comprendre/écrire les scénarii de test à exécuter.

L'application à tester étant une application web, un besoin s'est fait sentir de tester la partie rendue. Dans cet article, lorsque l'on parlera de tester la partie rendue, il sera question de vérifier que l'élément recherché se trouve bien dans le document html remonté dans le navigateur web. Pour rappel (cf. le paragraphe Contexte de ce [post](2013/03/demarrer-une-webapp-en-mode-embedded.html)), l'application testée s'appuie sur un framework web java de type Struts2.

Aussi, il ne sera pas question, ici, de tester le rendu dans différents navigateurs.

Il a été décidé de partir sur une solution s'appuyant sur un runtime à base de Selenium : en effet, un besoin latent étant, à terme, de tester le rendu de l'application web sur les différents navigateurs, cette solution semblait correspondre le mieux aux besoins.

Bref, passons ce besoin pour revenir à notre objectif premier, à savoir, vérifier la présence des éléments dans l'arbre DOM remonté par l'application web.

Pour résumer, il a été décidé de partir sur :

* Cucumber JVM pour la partie représentation/écriture des scénarii,
* Selenium pour la partie exécution des tests.

Cependant, la syntaxe sur la partie WebDriver de Selenium 2 étant assez verbeuse, il a été décidé d'utiliser le framework FluentLenium qui offre une API plus simple et plus naturelle (enfin plus fluent quoi! ;-) ). En outre, en plus d'une API plus facile d'utilisation, la notion native de Page de FluentLenium poussant à mieux découpler la représentation d'une page et son test, cela a joué en sa faveur ;-)

Ainsi, cet article présentera comment il a été possible d'intégrer Cucumber JVM avec FluentLenium afin de pouvoir faire tourner des tests avec Selenium.

A noter que je ne m'attarderai pas, dans cet article, à présenter exhaustivement les différents protagonistes mais seulement les quelques points qu'il est nécessaires de connaitre afin d'intégrer ensemble ces différents framework.

[update] Suite à discussion avec la créatrice de FluentLenium, un autre article a été initié et apporte de nombreux compléments mais également correction à cet article. Pour en savoir, plus, rendez vous [ici](/2013/04/fluentlenium-et-cucumber-jvm-complement.html)...

<!-- more -->

#Présentation des protagonistes

##Cucumber JVM
![center](http://2.bp.blogspot.com/-vb7zd_BqITk/UTmwyd4yWsI/AAAAAAAAA3w/tYVxH4hxlig/s1600/cucumber2.jpg)

[Cucumber JVM](https://github.com/cucumber/cucumber-jvm) est un fork Java de [Cucumber](http://cukes.info/) inialement développé en Ruby.

Tout comme [JBehave](http://jbehave.org/), il est orienté BDD (_Behaviour Driven Development_) et il permet d'écrire ses __scénarii__ de tests en suivant le pattern __Given/When/Then__ qui correspond à déterminer un ensemble de [Step](https://github.com/cucumber/cucumber/wiki/Step-Definitions).

Ces scénarii s'écrivent dans des fichiers __features__ qui sont lus par Cucumber JVM. Ce dernier se charge alors de faire correspondre les __Steps__ avec les fixtures associés. Ces __steps__ sont des méthodes Java annotées par :

* @Given(value = "")
* @When(value = "")
* @Then(value = "")

A noter que seule la valeur de l'annotation est utilisée par Cucumber JVM.

Alors que les steps à la sémantique Given permettront de poser les conditions nécessaires à l'exécution du scénario, les steps When exécuteront l'action à tester et les steps Then testeront que tout s'est bien passé en utilisant le framework d'assertion de son choix tels que JUnit, Fest-assert et/ou Harmcrest.

A noter également que pour passer un état d'une Step à une autre, Cucumber JVM nous oblige à stocker ces derniers dans des variables de classe ou à passer par son mécanisme d'injection à l'aide de framework IoC tels que [Picocontainer](http://picocontainer.codehaus.org/).

Ainsi, on peut résumer grossièrement en disant qu'un scénario est écrit dans un fichier __feature__ et est composé d'un ensemble de Step qui sont associés à des méthodes qui correspondent aux différentes __fixtures__.

Pour plus d'informations sur le BDD, je vous renvoie sur un [compte rendu](http://blog.soat.fr/2011/06/breizhcamp-behaviour-driven-development-par-olivier-billard-et-thierry-henrio/) d'une présentation d'Olivier Billard et de Thierry Henrio réalisé au BreizhCamp que j'avais fait à l'époque.

##Selenium 2

![center](http://4.bp.blogspot.com/-xOJkyRycXEU/UTmxGd1a-nI/AAAAAAAAA4A/kMPuTOc5tpY/s1600/big-logo.png)

Dans notre cas d'usage, il y a assez peu de chose à dire sur [Selenium](http://docs.seleniumhq.org/) si ce n'est qu'il permet, en fournissant différents __WebDriver__, de tester le rendu d'une page HTML en simulant différentes actions telles que le submit de formulaires, le clique d'un bouton et en allant chercher différents éléments dans la page rendue.

Il propose différentes implémentations de WebDriver tels que __FirefoxDriver__ ou __HtmlUnitDriver__.

Selenium offre également la possibilité d'exécuter les navigateurs qu'il lance sur différentes machines via Selenium Server mais, dans notre cas, cette fonctionnalité ne sera pas utile. De même, il ne sera pas abordé la partie Selenium IDE qui est peu exploitable car difficilement maintenable. En effet, il est courant et même fortement recommandé de séparer, pour des raisons évidentes, les scénarii à tester du rendu de la page (par exemple en utilisant le [Page Object Design Pattern](http://docs.seleniumhq.org/docs/06_test_design_considerations.jsp#page-object-design-pattern)).

Les liens suivants détaillent plus précisément ces différents points :

* http://fr.slideshare.net/MathildeLemee/selenium-testng-selenium-grid-best-practices
* http://docs.seleniumhq.org/docs/03_webdriver.jsp#selenium-webdriver-api-commands-and-operations

##FluentLenium

![center](http://1.bp.blogspot.com/-fyH1CNEBvIw/UTmxNk5WZTI/AAAAAAAAA4I/obgbEW5Ae6w/s1600/code.png)


[FluentLenium](https://github.com/FluentLenium/FluentLenium) est un framework utilisant Selenium mais proposant une API plus simple et plus naturelle que celle offerte par ce dernier.

Il a été pensé pour s'intégrer à des tests exécutés avec JUnit ou TestNG et se charge donc d'initialiser le __WebDriver__ Selenium à chaque fois qu'il exécute un test.

Pour ce faire, il utilise, au moment de l'écriture de ces lignes, le mécanisme de __Rule__ JUnit qui fait, grosso modo, comme le @Before de JUnit mais qui peut être partagé entre les différentes classes de test.

En outre, FluentLenium s'appuie sur la notion de __FluentPage__ et de __FluentTest__. En fait, pour faire simple, les classes de tests doivent étendre FluentTest, ce qui permet à toutes les méthodes annotées par @Test d'initialiser le WebDriver Selenium. La classe peut, de plus, bénéficier des méthodes portées par FluentTest.

La notion de FluentPage permet, quant à elle, de représenter une page (au sens HTML). Cette implémentation du [Page Object Design Pattern](http://docs.seleniumhq.org/docs/06_test_design_considerations.jsp#page-object-design-pattern) incite ainsi l'utilisateur à découpler le test du contenu de la page qui sera alors la seule à être garante du rendu.

Enfin, via l'annotation __Page__, les classes qui étendent FluentPage peuvent être injectées directement dans l'implémentation du FluentTest. Les liens suivants détaillent plus précisément ces différents points :

* http://fr.slideshare.net/MathildeLemee/fluentlenium
* https://github.com/FluentLenium/FluentLenium

#Etude sur la mise en oeuvre

On a vu dans le paragraphe précédent quelques-unes des notions nécessaires à l'intégration de nos trois comparses.

Cependant, il est intéressant de noter que Cucumber JVM et FluentLenium s'appuient sur deux paradigmes potentiellement opposés. En effet, alors que Cucumber JVM dispose d'une représentation par Step formalisé par des méthodes java, FluentLenium propose une granularité par méthode.

En outre, deux points sont primordiaux :

* FluentLenium s'appuie sur la notion de Rule JUnit pour instancier, démarrer puis stopper le webDriver,
* Cucumber JVM ne supporte pas la [notion de Rule](https://github.com/cucumber/cucumber-jvm/issues/393) et refuse tout autre Runner différent que le sien.

Bien sûr, on peut se douter qu'il est possible de contourner le problème sinon cet article serait un peu mensongé... ;-)

En fait, la solution qui a été mise en place pour faire fonctionner conjointement ces deux framework est assez simple : faire que les classes déclarant les __Steps__ Cucumber JVM délèguent à une classe étendant FluentTest les différentes actions et vérifications.

Cette classe pourra porter les différentes pages (au sens FluentLenium) et devra exposer les méthodes adéquates qui initialiseront et arrêteront le webDriver cible (méthodes fournies par FluentTest).

A titre informatif, le fait d'utiliser l'_Autocloseable_ (et plus précisément le _try-with-resources_) de Java 7 s'est traduit par un échec puisque le driver doit rester actif entre les différentes Steps.

De même, essayer d'injecter via Picocontainer les pages ne fonctionne pas car, à ce jour, l'implémentation même de FluentLenium fait que les annotations @Page qui permettent d'initialiser et d'instancier les __FluentPages__ doivent être dans une implémentation de __FluentTest__.

Ainsi, cela pourrait se traduire par le code suivant : 

`HomePageStep.java`

```java
import cucumber.api.java.en.Then;
import cucumber.api.java.en.When;
import org.fest.assertions.fluentlenium.FluentLeniumAssertions;

public class HomePageStep {

    private FluentTestDelegator fluentLeniumDelegate;

    /**
     * injection par pico donc pas besoin d'initialiser le driver (deja fait par le delegator)
     * @param fluentLeniumDelegate
     */
    public HomePageStep(FluentTestDelegator fluentLeniumDelegate) {
        this.fluentLeniumDelegate = fluentLeniumDelegate;
    }

    @When(value = "I go on home page")
    public void homePageIsDisplayed() {
            fluentLeniumDelegate.goTo(fluentLeniumDelegate.homePage).await().untilPage();
    }

    @Then(value = "home page is displayed")
    public void homePageIsDisplayed() {
        FluentLeniumAssertions.assertThat(fluentLeniumDelegate.homePage).isAt();
    }

    @When(value = "I submit the form")
    public void submitForm() {
        fluentLeniumDelegate.homePage.submit();
    }
}
```

`ResultPageStep.java`

```java
public class ResultPageStep {

    private FluentTestDelegator fluentLeniumDelegate;

    /**
     * injection par pico donc pas besoin d'initialiser le driver (deja fait par le delegator)
     * @param fluentLeniumDelegate
     */
    public ResultPageStep(FluentTestDelegator fluentLeniumDelegate) {
        this.fluentLeniumDelegate = fluentLeniumDelegate;
    }

    @Then(value = "I am on result page")
    public void isOnPage() {
        FluentLeniumAssertions.assertThat(fluentLeniumDelegate.resultPage).isAt();
    }
}
```

`CommonStep.java`
```java
import cucumber.api.java.en.When;

public class CommonStep {

    private FluentTestDelegator fluentLeniumDelegate;

    public StepHelper(FluentTestDelegator fluentLeniumDelegate) {
        this.fluentLeniumDelegate = fluentLeniumDelegate;
    }

    @When(value = "I stop my driver")
    public void webDrivercloser() {
        fluentLeniumDelegate.close();
    }
}
```

`FluentTestDelegator.java`
```java
import cucumber.api.java.en.When;
import org.fluentlenium.adapter.FluentTest;
import org.fluentlenium.core.annotation.Page;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.htmlunit.HtmlUnitDriver;
import org.openqa.selenium.remote.DesiredCapabilities;

import java.io.File;
import java.io.IOException;

public class FluentTestDelegator extends FluentTest implements AutoCloseable {
    @Page
    public HomePage homePage;

    @Page
    public ResultPage resultPage;

    public FluentTestDelegator() throws IOException {
        init();
    }

    /**
    * Appel des operations faites par le Rule Junit de FluentTest
    */
    public void init() throws IOException {
        initFluent(new HtmlUnitDriver()).withDefaultUrl("http://localhost:9090");
//        initFluent(getDefaultDriver()).withDefaultUrl("http://localhost:9090");
        initTest();
        setDefaultConfig();
    }

    @Override
    public void close() {
        if (getDriver() != null) {
            quit();
        }
    }
} 
```

`HomePage.jave`
```java
import cucumber.api.DataTable;
import org.fluentlenium.core.FluentPage;
import org.fluentlenium.core.domain.FluentList;
import org.fluentlenium.core.domain.FluentWebElement;

import java.util.List;

import static org.fest.assertions.Assertions.assertThat;
import static org.fluentlenium.core.filter.FilterConstructor.withText;
import static org.junit.Assert.assertEquals;

public class HomePage extends FluentPage {

    @Override
    public String getUrl() {
        return "/webapp/home";
    }

    @Override
    public void isAt() {
        assertThat(title()).containsIgnoringCase("homePage");
    }

    public void submit() {
        submit("#searchForm > form").await().untilPage();
    }
}
```


`ResultPage.java`
```java
public class ResultPage extends FluentPage {

    @Override
    public String getUrl() {
        return "/webapp/result";
    }

    @Override
    public void isAt() {
        assertThat(title()).containsIgnoringCase("resultPage");
    }
}
```


`scenario1.feature`
```text
# encoding: iso-8859-1

Feature: homepage test

  Scenario: homePage should be displayed
    When I go on home page
    Then home page is displayed
    Then I stop my driver
  
  Scenario: a submit on homePage should redirect to resultPage
    When I go on home page
    And I submit the form
    Then I am on result page
    Then I stop my driver
```


On constate que le code est un peu plus verbeux que ce qu'on aurait souhaité avoir mais cela fonctionne sans soucis.
A noter que via la méthode init de __FluentTestDelegator__, il est possible de préciser le __webDriver__ à utiliser (dans notre cas, HtmlUnitDriver).

#Mise en oeuvre

On a vu dans le paragraphe précédent comment il était possible de faire fonctionner conjointement Cucumber JVM et FluentLenium.

Du coup, il n'y a plus grand chose à rajouter dans ce paragraphe si ce n'est la configuration de notre chef d'orchestre (à savoir Maven) qui a été utilisé pour faire fonctionner tout ce beau monde... ;-)

```xml
    <dependencies>        
        <dependency>
            <groupId>org.hamcrest</groupId>
            <artifactId>hamcrest-all</artifactId>
            <version>1.3</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>batik</groupId>
            <artifactId>batik-ext</artifactId>
            <scope>test</scope>
        </dependency>        
        <!-- pas de montee de version de junit pour cause de conflit avec hamcrest -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <artifactId>hamcrest-core</artifactId>
                    <groupId>org.hamcrest</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>info.cukes</groupId>
            <artifactId>cucumber-junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>info.cukes</groupId>
            <artifactId>cucumber-java</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>info.cukes</groupId>
            <artifactId>cucumber-picocontainer</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.fluentlenium</groupId>
            <artifactId>fluentlenium-festassert</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>junit</groupId>
                    <artifactId>junit-dep</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.easytesting</groupId>
            <artifactId>fest-assert</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

  <build>
   <plugins> 
                    <plugin>
                        <artifactId>maven-failsafe-plugin</artifactId>
                        <executions>
                            <execution>
                                <goals>
                                    <goal>integration-test</goal>
                                    <goal>verify</goal>
                                </goals>
                            </execution>
                        </executions>
                        <configuration>
                            <includes>
                                <include>**/RunCucumberFeatures.java</include>
                            </includes>
                        </configuration>
                    </plugin>
 
        </plugins>             
  </build>   
```

```java
import cucumber.api.junit.Cucumber;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@Cucumber.Options(features = "classpath:fr/jetoile/webapp/acceptance", format = {"pretty", "html:target/cucumber", "json:target/cucumber.json"})
public class RunCucumberFeatures {
}
```

#Conclusion

On a vu dans cet article (qui est la suite logique d'un [article précédent](/2013/03/demarrer-une-webapp-en-mode-embedded.html)) comment il était possible de faire des tests d'acceptance en utilisant conjointement Cucumber JVM et FluentLenium.

A l'utilisation, cela s'avère agréable et rapide à écrire surtout avec quelques petits tweaks supplémentaires qui n'ont pas été exposés ici (profile Maven pour ne démarrer que le serveur embedded et exécution des scénarii Cucumber avec IntelliJ 12 avec possibilité de bénéficier du debugger que ce soit au niveau de l'exécution des tests (debugger ou utilisation d'un webDriver autre que HtmlUnitDriver) ou de l'application cible (via `mvnDebug`)).

Bref, en tout cas, même si la mise en oeuvre a été un peu galère, il a été possible de bénéficier du meilleur des deux framework sans avoir à se "taper" la lourdeux de Selenium... ;-).

Bien sûr, il est existe bien d'autres solutions mais je te laisse, précieux lecteur, le soin de les évaluer ;-)