---
title: "Jenkins Shared Library mit Guice"
date: 2021-01-17T22:17:57+01:00
draft: false
tags:
 - google guice
 - jenkins
 - jenkins shared library
 - groovy
---
Dieser Artikel zeigt, wie man in einer Jenkins Shared Library Dependency Injection mittels Google Guice implementiert und testet.

<!--more-->

Jenkins bietet mit seinen Shared Libraries die Möglichkeit, globalen Code für alle Pipelines einer


Grundsätzlich ist keine Konfiguration zur Nutzung von Guice notwendig.
Jenkins stellt alle **Abhängigkeiten** selbst im Klassenpfad zur Verfügung.
Zu sehen ist dies unter _Jenkins verwalten ➔ Über Jenkins_ oder _https://\<jenkins>/about/_.
Jenkins 2.263.2 verwendet Guice 4.0.

In diesem **Beispiel** soll ein Pipeline Step implementiert werden, welcher den aktuellen Job abbricht und gleichzeitig die Build Information ändert.
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

Der direkte **Zugriff auf die Pipeline** wird über das Interface `ISteps` ermöglicht.
Die dazugehörige Implementierung heißt `Steps`.
Sie speichert die Referenz auf die Pipeline und delegiert Methodenaufrufe an diese weiter.
Diese erhält sie über den Konstruktor.

{{< highlight groovy "linenostart=3,linenos=table,hl_lines=5 10 15" >}}
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

Die Klasse `PipelineUtils` ist **Konsument von `ISteps`.**
Sie erhält diese über den Konstruktor.
Damit Guice weiß, dass die Dependency Injection hier durchgeführt werden soll, muss diese mittels `@Inject` annotiert werden.
Die Methode `abortBuild` führt den bereits angesprochenen Step `error` und die Zuweisung der Build Descriptoin durch.

{{< highlight groovy "linenostart=8,linenos=table,hl_lines=5 13" >}}
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

Bevor Guice nun eine Instanz von `PipelineUtils` zur Verfügung stellen kann, muss noch geklärt werden, wie die benötigte **Abhängigkeit zu `IStep`** konstruiert werden kann.
Dies geschieht über via `StepsModule` in der Methode `configure()`.

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

# Interessante Links
## Beispiele zu diesem Artikel
* [jenkins-shared-library-with-guice](https://github.com/mem89de/jenkins-shared-library-with-guice) auf github.com
## Externe Quellen
* [Extending with Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/) auf jenkins.io
* [How-To: Setup a unit-testable Jenkins shared pipeline library](https://dev.to/kuperadrian/how-to-setup-a-unit-testable-jenkins-shared-pipeline-library-2e62) auf dev.to
