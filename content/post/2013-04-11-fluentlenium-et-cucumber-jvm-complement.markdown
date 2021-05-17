---
title: "FluentLenium et Cucumber JVM... complément et precision"
date: 2013-04-11 20:38:54 +0100
comments: true
tags: 
- java
- intégration continue
- fluentlenium
- selenium
- maven
---

![left-small](http://4.bp.blogspot.com/-R3ORi15Ei3Y/UTmwoq5A2II/AAAAAAAAA3o/uaIMSSZlq38/s1600/logo.png)

Dans mon article précédent, j'avais tenté d'expliquer comment il était possible d'intégrer les frameworks [Cucumber JVM](https://github.com/cucumber/cucumber-jvm) et [Selenium](http://docs.seleniumhq.org/) au travers de [FluentLenium](https://github.com/FluentLenium/FluentLenium).


En effet, pour rappel, FluentLenium permettait d'abstraire Selenium en lui offrant une API plus _fluent_ mais également en lui apportant nativement ce qu'il préconise, à savoir le [Page Object Design Pattern](http://docs.seleniumhq.org/docs/06_test_design_considerations.jsp#page-object-design-pattern).

Pour ce faire, j'avais proposé d'utiliser la délégation de l'initialisation de FluentLenium à une classe tierce injectée via le mécanisme d'injection de Cucumber JVM.

Cependant, suite à discussion avec la créatrice de FluentLenium (à savoir [Mathilde](https://twitter.com/mathildelemee)), on s'est rendu compte que l'axe utilisé était légèrement biaisé (même s'il fonctionnait...).

Cet article revient donc sur ce point en proposant une solution plus simple mais présentera également comment il est possible de tester le scénario Cucumber avec différents navigateurs et il y aura un petit mot sur l'utilisation de navigateurs déportés (via les RemoteWebDriver de Selenium 2).

Pour ce faire, il sera découpé en 3 parties qui couvriront des usecases différents se traduisant donc par des implémentations différentes :

* cas de tests pour un site simple,
* cas de tests pour un site complet,
* cas de tests multi-navigateurs pour un site complet.

A noter que je ne reviendrai pas sur les principes des différents frameworks/concepts mais juste sur comment il est possible d'implémenter ces différents usecases.

A noter également que l'article précédent aurait pu être modifié mais qu'en raison du nombre important de changements, il était plus simple d'en initier un autre...

<!-- more -->

#Cas de tests pour un site simple

##Présentation et proposition d'implémentation

Ce premier cas d'usage couvre le cas : "j'ai un site que je veux tester avec Cucumber JVM et l'ensemble des steps peut être réuni dans une seule et même classe."

Bon je vois déjà la levée de bouclier : pourquoi réunir toutes les steps dans une seule et même classe. En fait, la raison du pourquoi sera expliquée un peu plus tard dans le paragraphe _Limites_ de ce chapitre donc patience... ;-)

Contrairement à la façon que j'avais présentée dans mon article précédent, il n'est pas obligatoire de déléguer la déclaration des pages `FluentPage` à une autre classe étendant `FluentTest`. En fait, il suffit juste de faire étendre la classe contenant les steps cucumber de `FluentAdapter`, d'y déclarer les pages et d'appeler dans la méthode annotée par @Before (celui de Cucumber JVM bien sûr) les méthode d'initialisation du contexte de FluentLenium.

Pour rappel, cette initialisation instancie le WebDriver utilisé par Selenium 2 mais également les pages (annotées par l'annotation `@Page`) présentes dans la classe courante (ou ses parentes) qui doit, au minimum, étendre de `FluentAdapter`. Cela se fait au travers des méthodes `initFluent()` et `initTest()`.

Le code est donc extrêmement simple puisqu'il suffit de faire quelque chose du style : 

```java
import cucumber.api.java.After;
import cucumber.api.java.Before;
import cucumber.api.java.en.Given;
import cucumber.api.java.en.Then;
import cucumber.api.java.en.When;
import org.fluentlenium.core.FluentAdapter;
import org.fluentlenium.core.annotation.Page;
import org.openqa.selenium.htmlunit.HtmlUnitDriver;

import static org.fest.assertions.api.Assertions.assertThat;

public class SimpleStep extends FluentAdapter {

 @Page
 BingPage page;

 @Before
 public void before() {
  this.initFluent(new HtmlUnitDriver());
  this.initTest();
 }

 @Given(value = "j accede a bing")
 public void step1() {
  goTo(page);
 }

 @When(value = "je recherche ([^ ]*) ")
 public void step2(String keyword) {
  fill("#sb_form_q").with(keyword);
  submit("#sb_form_go");
 }

 @Then(value = "le titre est ([^ ]*) ")
 public void step3(String keyword) {
  assertThat(title()).contains(keyword);
 }

 @After
 public void after(){
  this.quit();
 }
}
```

où BingPage est :

```java
import org.fluentlenium.core.FluentPage;

public class BingPage extends FluentPage {

 @Override
 public String getUrl() {
  return "http://www.bing.com";
 }
}
```

et la feature, la suivante :

```text
Feature: basic

  Scenario: scenar1
    Given j accede a bing
    When je recherche toto
    Then le titre est toto 
```

Il est intéressant de remarquer la simplicité de la chose (et rien à voir avec l'implémentation que j'avais proposé précédemment!).

A part cela, peu de choses à ajouter : le code parle de lui même...

##Limites

On vient de voir comme il était simple de faire cohabiter FluentLenium et Cucumber JVM.

Bien sûr, il y a un mais... (ça serait trop simple sinon) : comme on peut le constater, actuellement, toutes les steps se trouvent être dans la même classe. Cependant, dans le cas d'un site web un peu plus complexe, il est courant et même encouragé de séparer les steps dans différentes classes.

Dans l'implémentation précédente, l'annotation `@Before` a été utilisée pour initialiser le contexte (et plus particulièrement le webDriver et les pages pour ensuite injecter le webDriver dans ces dernières).

Cependant, dans le cas où les steps se trouvent être dans plusieurs classes, cela pose potentiellement un problème.

En effet, Cucumber JVM instancie la classe qui contient la définition de la step dès qu'il en a besoin et appelle la méthode annotée par `@Before` à l'instanciation de cette classe. Ainsi, dans notre cas, si les steps s'étaient trouvées dans deux classes, chacune étendant `FluentAdapter` et appelant `initFluent()` et `initTest()` dans la méthode annotée par @Before, alors cette instanciation aurait été faite deux fois et non une seule fois comme on aurait pu s'y attendre pour un même scénario donné...

Pire, les pages déclarées dans les classes n'auraient pas eu la même instance du webDriver et elles ne se seraient pas vu l'une l'autre...

Pas glop tout ça... :'(

Ainsi, l'implémentation précédente fonctionne pour des cas "simples" mais si la partie test d’acceptante/intégration avait été plus complexe, alors cela aurait empêché la réutilisation et le découplage.

#Cas de tests pour un site complet

##Présentation et proposition d'implémentation

Il a été vu dans le paragraphe précédent qu'il pouvait être utile de disposer de plusieurs classes disposant des implémentations des fixtures.
Cependant, la question principale est de trouver comment il est possible de n'instancier qu'une seule fois par scénario le webDriver et de l'injecter dans des instances de pages propres au scénario.

La proposition présentée dans l'article précédent (modulo qu'il ne faut pas étendre de `FluentTest` mais de `FluentAdapter`) reste viable, mais il y a plus simple.

En effet, dans la proposition faite précédemment, la classe `FluentTestDelegator` avait à sa charge, à la fois la déclaration des pages, et l'instanciation et l'initialisation du contexte de FluentLenium. Pour rappel, cette instanciation/initialisation était réalisée par [Pico Container](http://picocontainer.codehaus.org/) lors de l'injection de l'instance de cette classe dans la classe contenant les fixtures.

En fait, il est plus propre, d'un point de vue séparation des concepts, de laisser à cette classe le soin de proposer les fixtures d'initialisation du webDriver tout en séparant la déclaration des pages.

Cela peut être réalisé en créant une classe (`FluentPageInjector`) étendant de `FluentAdapter` qui définit les pages et de la faire étendre d'une classe (`FluentLeniumStepInitilizer`) qui définit les fixtures d'instanciation du webDriver.

Cela offre deux avantages : les classes qui définissent les steps de navigation n'ont qu'à étendre de `FluentPageInjector` pour avoir une visibilité sur les pages (tout en continuant d'injecter via pico l'instance de `FluentLeniumStepInitilizer`) et il devient alors possible de variabiliser le webDriver à utiliser. 

```text
Feature: browser

  Scenario: navigation version firefox
    Given I connect on url http://localhost:8080 with firefox
    Given j accede a la homePage
    And je suis sur homePage
    When je submit
    Then je suis sur la page result
    And driver is closed


  Scenario: navigation version chrome
    Given I connect on url http://localhost:8080 with chrome with parameters webdriver.chrome.driver:/opt/chromedriver/chromedriver
    Given j accede a la homePage
    And je suis sur homePage
    When je submit
    Then je suis sur la page result
    And driver is closed


  Scenario: navigation version phantomjs
    Given I connect on url http://localhost:8080 with phantomjs with parameters phantomjs.binary.path:/opt/phantomjs-1.9.0-linux-x86_64/bin/phantomjs 
    Given j accede a la homePage
    And je suis sur homePage
    When je submit
    Then je suis sur la page result
    And driver is closed
```
<br/>
```java
package step;

import org.fluentlenium.core.FluentAdapter;
import org.fluentlenium.core.annotation.Page;
import page.HomePage;
import page.ResultPage;

public class FluentPageInjector extends FluentAdapter {
    @Page
    protected HomePage homePage;

    @Page
    protected ResultPage resultPage;
} 
```

On remarque, comme dit plus haut, que cette classe n'a qu'un seul rôle qui est de déclarer les pages tout en étendant de `FluentAdapter`. 

```java
public class HomePageStep extends FluentPageInjector {

     public HomePageStep(FluentLeniumStepInitilizer delegator) {
      this.homePage = delegator.homePage;
      this.resultPage = delegator.resultPage;
     }


     @Given("^j accede a la homePage$")
     public void j_accede_a_homePage() {
      goTo(homePage);
     }

     @Given("^je suis sur homePage$")
     public void je_suis_sur_homePage() {
      homePage.isAt();
     }

     @When("^je submit$")
     public void je_submit() throws Throwable {
      homePage.submit();
     }
}
```

<br/>

```java
public class ResultPageStep extends FluentPageInjector {

    public ResultPageStep(FluentLeniumStepInitilizer delegator) {
        this.resultPage = delegator.resultPage;
    }

    @When("^je suis sur la page result$")
    public void je_suis_sur_la_page_result() throws Throwable {
        resultPage.isAt();
    }

} 
```

Ces classes correspondent aux classes qui définissent les fixtures. Elles étendent de `FluentPageInjector` de façon à pouvoir bénéficier de la visibilité sur les pages. Par contre, il est intéressant de constater que, dans son constructeur, la classe `FluentLeniumStepInitializer` est injecté via Pico Container. Cela permet d'affecter la valeur des pages.

```java
public class FluentLeniumStepInitilizer extends FluentPageInjector {

    @Given("^I connect on url ([^ ]*) with ([^ ]*) with parameters ([^ ]*)$")
    public void browser_connect(String host, String browser, String parameters) {
        init(host, browser, parameters);
    }

    @Then("^drivers are closed")
    public void close() {
        this.quit();
    }

    @After
    public void afterClose() {
        this.quit();
    }

    private void init(String host, String browserName, String parametersLine) {
        Browser browser = null;

        DesiredCapabilities capabilities = new DesiredCapabilities();

        String[] parameters = parametersLine.slip(";");
        for (String parameter : parameters) {
            if (!parameter.isEmpty()) {
                String[] key_value = parameter.split(":");
                capabilities.setCapability(key_value[0], key_value[1]);
            }
        }

        browser = Browser.getBrowser(browserLine.get(0));
        capabilities.setBrowserName(browser.getName());

        initWebDriver(host, browserHost, browser, capabilities);
    }

    private void initWebDriver(String host, String browserHost, Browser browser, DesiredCapabilities capabilities) {
        Fluent fluent = null;
        WebDriver driver = null;
        if (browser != null) {
            driver = BrowserMapper.getDriver(browser, capabilities);
            fluent = this.initFluent(driver);
        }

        fluent.withDefaultUrl(host);
        this.initTest();
    }
}
```


Cette classe étend par transitivité `FluentAdapter` et dispose donc de la visibilité sur les méthodes d'initialisation de `FluentLenium`. En outre, en étendant `FluentPageInjector` (qui étend de `FluentAdapter`), cela lui permet, à l'appel de `initTest()`, d'initialiser les pages.
Concernant l'initialisation des webDriver, cela est fait au niveau de la classe `BrowserMapper`. 

```java
public enum Browser {
    HTMLUNIT("default"),
    FIREFOX("firefox"),
    CHROME("chrome"),
    PHANTOMJS("phantomjs");

    private String name;

    Browser(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }

    public static Browser getBrowser(String name) {
        for (Browser browser : values()) {
            if (browser.getName().equalsIgnoreCase(name)) {
                return browser;
            }
        }
        return HTMLUNIT;
    }
}
```
<br/>
```java
import org.openqa.selenium.Capabilities;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.htmlunit.HtmlUnitDriver;
import org.openqa.selenium.phantomjs.PhantomJSDriver;

import java.util.HashMap;
import java.util.Map;

public class BrowserMapper {
 private String name;

 private static Map<Browser, WebDriverLazyLoader> browserMapper = new HashMap<Browser, WebDriverLazyLoader>();

 static {
  browserMapper.put(Browser.CHROME, new WebDriverLazyLoader(ChromeDriver.class));
  browserMapper.put(Browser.FIREFOX, new WebDriverLazyLoader(FirefoxDriver.class));
  browserMapper.put(Browser.HTMLUNIT, new WebDriverLazyLoader(HtmlUnitDriver.class));
  browserMapper.put(Browser.PHANTOMJS, new WebDriverLazyLoader(PhantomJSDriver.class));
 }

 public static WebDriver getDriver(Browser browser, Capabilities capabilities) {
  WebDriverLazyLoader webDriverLazyLoader = browserMapper.get(browser);
  if (webDriverLazyLoader != null) {
            if (browser == Browser.PHANTOMJS) {
                return webDriverLazyLoader.getWebDriverClass(capabilities);

            } else if (browser == Browser.CHROME) {
                System.setProperty("webdriver.chrome.driver", (String) capabilities.getCapability("webdriver.chrome.driver"));
                return webDriverLazyLoader.getWebDriverClass(capabilities);

            } else {
                return webDriverLazyLoader.getWebDriverClass();
            }
  }
  return browserMapper.get(Browser.HTMLUNIT).getWebDriverClass();
 }
} 
```

Cette classe permet de faire le pont avec les webDriver qu'il est possible d'utiliser. Cependant, la petite astuce consiste à instancier de manière "Lazy" ces derniers.

En effet, appeler le constructeur d'un webDriver l'instancie mais le démarre également (ie. que la fenêtre du navigateur s'ouvre réellement). Du coup, la petite classe présentée ci-dessous a été utilisée. 

```java
import org.openqa.selenium.Capabilities;
import org.openqa.selenium.WebDriver;

class WebDriverLazyLoader {
    private Class webDriverClass;


    public WebDriverLazyLoader(Class webDriverClass) {
        this.webDriverClass = webDriverClass;
    }

    public WebDriver getWebDriverClass() {
        try {
            return (WebDriver)this.webDriverClass.newInstance();
        } catch (ReflectiveOperationException e) {
            e.printStackTrace();
        }
        return null;
    }

    public WebDriver getWebDriverClass(Capabilities capabilities) {
        try {
            return (WebDriver)this.webDriverClass.getConstructor(Capabilities.class).newInstance(capabilities);
        } catch (ReflectiveOperationException e) {
            e.printStackTrace();
        }
        return null;
    }
} 
```

##Limites

Comme on a pu le voir dans ce chapitre, il est aisé de partager les fixtures Cucumber JVM dans des classes différentes tout en bénéficiant de FluentLenium.

Cependant, pour certains besoins, il peut être utile de vouloir lancer les tests d'acceptance/intégration sur différents navigateurs.

Bien sûr, chaque step pourrait boucler sur l'ensemble des navigateurs sur lesquels les tests doivent être exécutés, mais cela induirait des problématiques d'entrelacement des actions et donc soulèverait des problématiques comme la gestion d'un contexte par webDriver, l'accès à un rapport "illisible" ou un manque de contrôle sur les préconditions du test qui sont, généralement, lié au scénario et non à une Step.

Le chapitre suivant tentera de répondre à cette problématique en proposant un moyen de "boucler" sur le scénario avec différents navigateurs. 


#Cas de tests multi-navigateurs pour un site complet

##Présentation et proposition d'implémentation

Il a été vu dans le chapitre précédent comment il était possible d'exécuter des tests d'acceptances/intégration sur un navigateur donné.

Ce chapitre présentera, pour sa part, une façon de les lancer sur différents navigateurs sans avoir à faire de copier/coller ;-).

En fait, Cucumber JVM permet nativement de boucler sur un scénario en utilisant différents paramètres. Cela se fait par le mécanisme de `scenario outline`. 

```text

Feature: multibrowser

  Scenario Outline: multi browser navigation version 1:
    Given I connect on url http://localhost:8080 with <browser> with parameters <parameters>
    Given j accede a la homePage
    And je suis sur homePage
    When je submit
    Then je suis sur la page result
    And driver is closed
  Examples:
    | browser | parameters                                             | 
    | firefox |                                                        | 
    | default |                                                        | 
    | chrome  | webdriver.chrome.driver:/opt/chromedriver/chromedriver | 
```

Et... c'est tout!

Le code n'a pas à être modifié : Cucumber JVM s'occupe de tout! ;-) 

##Limites

On a vu dans le paragraphe précédent comment il était possible d'exécuter facilement des tests d'acceptance/intégration en s'appuyant sur la notion de scénario outline offerte nativement par Cucumber JVM.

Coté limitation, je n'en vois pas trop...

Peut être le fait de ne pas instancier le webDriver pour chaque scénario (opération assez coûteuse en temps) mais cela est aisément résolvable en utilisant une sorte de cache de webDriver fonctionnant sur le principe de singleton qui serait réinialisé lors de l'appel à la step drivers are closed qui serait isolée dans son propre `scenario outline` :

```text
  Scenario Outline: browsers are closed:
    Then driver is closed
  Examples:
    | browser   | parameters                                                            | 
    | firefox   |                                                                       | 
    | default   |                                                                       | 
    | phantomjs | phantomjs.binary.path:/opt/phantomjs-1.9.0-linux-x86_64/bin/phantomjs | 
    | chrome    | webdriver.chrome.driver:/opt/chromedriver/chromedriver                |     
```

#Conclusion

Il a été présenté dans cet article comment il était possible d'implémenter l'intégration de Cucumber JVM et de Selenium à l'aide de FluentLenium.

Cet article n'a, cependant, pas fait mention de l'exécution des tests sur des navigateurs distants via les __RemoteWebDriver__ mais cela est tout à fait possible (même si le code montré ici ne le présente pas) et est même totalement fonctionnel : pour ce faire (code disponible [ici](https://github.com/jetoile/fluentlenium-cucumber/tree/multiNav)), il suffit de fournir, entre autre, des paramètres supplémentaires comme l'url de connexion au hub Selenium et d'instancier un `RemoteWebDriver` plutôt que le webDriver.

De même, le code permettant d'instancier à la mode singleton les webDriver est disponible [ici](https://github.com/jetoile/fluentlenium-cucumber/blob/multiNav/src/test/java/step/FluentLeniumStepInitilizer.java) (voir la méthode `initCachedWebDriver()` de la classe `FluentLeniumStepInitilizer`).

Enfin, un dernier mot sur la façon dont il est possible d'exécuter tout ce beau monde (comme ça, je réponds à la remarque très pertinente de [José](https://twitter.com/josepaumard) ;-) ) parce que faire des tests, c'est bien, les jouer, c'est mieux!

Il est possible de jouer les tests d'au moins trois manières distinctes : une orienté "vraie vie" (ie. utilisable au sein d'un build maven et donc exécutable via une usine d'intégration continue) et deux autres plutôt orientés développement.

Ainsi, pour jouer les tests via maven, il suffit de le déclarer dans le `pom.xml` le plugin failsafe en le branchant sur la "bonne phase", à savoir le runner Cucumber JVM : 

```xml
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
            <include>**/BasicRunner.java</include>
        </includes>
    </configuration>
</plugin>
```

Coté jouabilité des tests sur un environnement de développement  il est possible d'exécuter le __Runner__ Cucumber JVM directement (comme lors de l'exécution d'une classe de test unitaire) ou d'utiliser le plugin Cucumber JVM proposé par notre IDE préféré (pour moi IntelliJ, pour les autres, je ne sais pas...). 

```java
import cucumber.api.junit.Cucumber;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@Cucumber.Options(features = "classpath:fr/jetoile/webapp/acceptance", format = {"pretty", "html:target/cucumber", "json:target/cucumber.json"})
public class RunCucumberFeatures {
}
```

![medium](http://4.bp.blogspot.com/-RGW7oSAChOQ/UWXmqPsYh6I/AAAAAAAAA4Y/cDH-YyVcpgs/s1600/screenshot01.png)

<br/>

![medium](http://3.bp.blogspot.com/-FkF8ZO6r6co/UWXnUNjqjII/AAAAAAAAA4g/obtSC7BS6kI/s1600/screenshot02.png)

Enfin, pour rappel, mon usecase étant de tester mon application web en boite noire, un prérequis était que mon application web soit démarrée au préalable.

Pour ce faire, le plugin maven Jetty (ou Tomcat au choix) a été utilisé et branché sur la phase de pré-integration.

Lors de l'exécution des tests en mode développement (ie. en les lançant comme un TU ou à l'aide du plugin Cucumber JVM via l'IDE), un profil n'exécutant pas le plugin failsafe mais uniquement le démarrage du jetty/tomcat embarqué a été utilisé.

#Pour aller plus loin...

* article sur les limitations de Cucumber JVM pour le partage de données entre steps : http://zsoltfabok.com/blog/2012/09/cucumber-jvm-hooks/
* page de fluentLenium : https://github.com/FluentLenium/FluentLenium
* page des webDriver de Selenium : http://docs.seleniumhq.org/docs/03_webdriver.jsp
* code : http://github.com/jetoile/fluentlenium-cucumber/