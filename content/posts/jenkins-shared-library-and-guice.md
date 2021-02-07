---
title: "Jenkins Shared Library mit Guice"
date: 2021-01-17T22:17:57+01:00
lastmod: 2021-02-07T11:39:43+01:00
draft: false
tags:
 - google guice
 - jenkins
 - jenkins shared library
 - groovy
---
Jenkins bietet mit seinen Shared Libraries die Möglichkeit, globalen Groovy-Code für alle Pipelines einer Jenkins Installation zur Verfügung zu stellen.
Auch hier gehört es zum guten Ton, einzelne Klassen soweit wie möglich voneinander zu entkoppeln.
Ein probates Mittel dazu ist das Entwicklungsmuster *Dependency Injection*.
Mit Guice bietet Google ein erprobtes Framework zur Umsetzung hierfür.

Dieser Artikel zeigt, wie man in einer [Jenkins Shared Library](https://www.jenkins.io/doc/book/pipeline/shared-libraries/) Dependency Injection mittels Google Guice implementiert und testet.

<!--more-->
Grundsätzlich ist keine **Konfiguration** zur Nutzung von Guice notwendig.
Jenkins stellt alle Abhängigkeiten selbst im Klassenpfad zur Verfügung.
Zu sehen ist dies unter _Jenkins verwalten ➔ Über Jenkins_ oder _https://\<jenkins>/about/_.
Jenkins 2.263.2 verwendet Guice 4.0.

In diesem **Beispiel** soll ein Pipeline Step implementiert werden, welcher den aktuellen Job abbricht und gleichzeitig die Build-Information ändert.
Auf diese Weise lässt sich in der Auflistung der Jobs schnell sehen warum ein Job abgebrochen ist.
Dies lässt sich auf dem herkömmlichen Wege, wie folgt umsetzen:

{{< highlight groovy "linenos=table,hl_lines=8-9">}}
pipeline {
  agent any

  stages {
    stage('Fail build') {
      steps {
        def errorMessage = 'Something, somewhere went terribly wrong'
        error errorMessage
        currentBuild.description = errorMessage
      }
    }
  }
}
{{< / highlight >}}

**Aufrufe der Pipeline-Steps** kapseln wir mit unserem Interface `ISteps`.
Deren Implementierung ist gemäß Single-Responsibility-Prinzip vorerst nicht von belang.

{{< highlight groovy "linenostart=3,linenos=table,hl_lines=9 15" >}}
/**
 * Provides access to Jenkins pipeline steps.
 */
interface ISteps {
    /**
     * Signals an error. Useful if you want to conditionally abort some part of your program.
     * @param message the error message
     */
    void error(String message)

    /**
     * Sets the build description
     * @param buildDescription the build description
     */
    void setBuildDescription(String buildDescription);
}
{{< / highlight >}}

Wir möchten die Funktionalität von `ISteps` nun in unserer Klasse `PipelineUtils` nutzen.
Gemäß Dependency Injection erzeugt `PipelineUtils` keine eigene Instanz dieser Schnittstelle, sondern lässt sie sich injizieren.
Dies erledigt Guice für uns, wenn wir den Konstruktor entsprechend mit `@Inject` annotieren.

{{< highlight groovy "linenostart=8,linenos=table,hl_lines=4" >}}
class PipelineUtils {
    private final ISteps steps

    @Inject
    protected PipelineUtils(ISteps steps) {
        this.steps = steps
    }

    /**
     * Sets the build description to <code>errorMessage</code> and aborts the build.
     * @param errorMessage the error message
     */
    void abortBuild(String errorMessage) {
        steps.setBuildDescription(errorMessage)
        steps.error(errorMessage)
    }
}
{{< / highlight >}}

Damit die Injektion einer Instanz von `ISteps` funktioniert, muss Guice wissen, wie es eine Instanz konstriuert.
Dafür benötigen wir zunächst eine naive Implementierung `Steps`

{{< highlight groovy "linenostart=3,linenos=table" >}}
class Steps implements ISteps, Serializable {
    private final steps

    Steps(Object steps) {
        this.steps = steps
    }

    @Override
    void error(String message) {
        steps.error(message)
    }

    @Override
    void setBuildDescription(String buildDescription) {
        steps.currentBuild.description = buildDescription
    }
}
{{< / highlight >}}

Mithilfe eines Moduls binden wir nun `ISteps` an eine konktrete Instanz von `Steps`:

{{< highlight groovy "linenostart=6,linenos=table,hl_lines=11" >}}
class StepsModule extends AbstractModule implements Serializable {
    private final steps

    StepsModule(steps) {
        this.steps = steps
    }

    @Override
    @NonCPS
    protected void configure() {
        bind(ISteps.class).toInstance(new Steps(steps))
    }
}
{{< / highlight >}}

Nun kann in _vars/abortBuild.groovy_ ein Step definiert werden, der bei Guice eine Instanz von `PipelineUtils` beantragt und `abortBuild` aufruft.
Wichtig ist, dass bei der Erzeugung des `Injector` die Referenz auf die aufrufende Pipeline übergeben wird.

{{< highlight groovy "linenostart=10,linenos=table,hl_lines=3 7-10" >}}
void call(String errorMessage) {
    PipelineUtils pipelineUtils = getPipelineUtils()
    pipelineUtils.abortBuild(errorMessage)
}

private PipelineUtils getPipelineUtils() {
    Injector injector = Guice.createInjector(
            new StepsModule(this)
    )
    return injector.getInstance(PipelineUtils.class)
}
{{< / highlight >}}

Die Erzeugung von `PipelineUtils` muss nicht explizit mithilfe eines Moduls definiert werden.
Dies tut Guice von selbst.

Das initiale Beispiel einer Pipeline sieht nun, wie folgt aus.
Der zusätzliche `script`-Step ist eine Notwendigkeit bei der Nutzung von Shared-Library-Steps in deklarativen Pipelines.

{{< highlight groovy "linenos=table,hl_lines=8" >}}
pipeline {
  agent any

  stages {
    stage('Fail build') {
      steps {
        script {
          abortBuild 'Something, somewhere went terribly wrong'
        }
      }
    }
  }
}
{{< / highlight >}}

An nächster Stelle sollte nun zumindest ein Unit-Test stehen.
Wir orientieren uns am [Blog-Artikel von Adrian Kuper](https://loglevel-blog.com/how-to-setup-and-develop-jenkins-shared-pipeline-library/) und testen nur unsere Groovy-Klassen.
Alle Steps, also Code unter _vars/_ wird bewusst minimal belassen, sodass ein gesonderter Test nicht zwingend notwendig ist.
Ein **Test in Spock** könnte, wie folgt aussehen:

{{< highlight groovy "linenostart=5,linenos=table,hl_lines=8-9 14" >}}
class PipelineUtilsTest extends Specification {
    private static final String ERROR_MESSAGE = "ERROR_MESSAGE"

    private PipelineUtils pipelineUtils
    private ISteps steps

    def setup() {
        steps = Mock()
        pipelineUtils = new PipelineUtils(steps)
    }

    def "abort build with build description"() {
        when:
        pipelineUtils.abortBuild(ERROR_MESSAGE)

        then:
        1 * steps.setBuildDescription(ERROR_MESSAGE)

        then:
        1 * steps.error(ERROR_MESSAGE)
    }
}
{{< / highlight >}}

Spätestens an dieser Stelle sollte ein Build-Tool nach eigenem Belieben konfiguriert sein, um den Code zu kompilieren und zu testen.
Bei mir ist die Wahl auf Maven gefallen.
Die dazugehörige _pom.xml_ ist im  [Github-Repository](https://github.com/mem89de/jenkins-shared-library-with-guice) zu finden.
