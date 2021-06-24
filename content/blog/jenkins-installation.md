---
title: "Jenkins Installation"
date: 2021-06-24T22:06:16+02:00
draft: false
tags:
 - jenkins
 - virtualbox
 - initd
---

Für meine Projekte möchte ich eine Jenkins-Instanz in einer virtuellen Maschine auf meinem Notebook einrichten.
Dieser Artikel beschäftigt sich mit der initialen Installation und der Einrichtung des Autostarts.

<!--more-->

# Das Betriebssystem
Aus der Erfahrung weiß ich, dass Jenkins sich unter Windows gerne ab und an zickig anstellt.
Das kann im einfachsten Fall der Verzeichnis-Trenner `\`, beziehungsweise `/` sein.
Deshalb möchte ich Jenkins auf jeden Fall unter Linux installieren.
Ich habe mich für **Linux Mint (Xfce) in einer VirtualBox VM** entschieden.

Hierbei sind erst einmal wenige Einstellungen nötig.
In den Netzwerkeinstellungen der virtuellen Maschine in VirtualBox wähle ich den Modus **Netzwerkbrücke**.
So ist später die Oberfläche des Jenkins auch aus dem Browser anderer Maschinen erreichbar.

Im Linux selbst soll der Jenkins unter einem eigenen **User** laufen.
Dieser wird mit den folgenden Befehlen einfach angelegt:

```
sudo -s
adduser jenkins
groupadd jenkins_group
usermod -a -G jenkins_group jenkins
```

Damit sind die Vorbereitungen abgeschlossen.

# Installation von Jenkins
Es gibt verschiedenste Methoden, Jenkins zum Laufen zu bringen.
Ich entscheide mich vorerst für die einfachste Variante&mdash;das direkte **Ausführen des Jar**.
Eine Alternative wäre beispielsweise Docker.
Gegebenenfalls möchte ich dies zu einem späteren Zeitpunkt probieren.

Als Installationsverzeichnis wähle ich `/home/jenkins/app`.
```
cd /home/jenkins/app
wget https://get.jenkins.io/war-stable/2.289.1/jenkins.war -O jenkins.2.289.1.war
ln jenkins.2.289.1.war jenkins.war
```
Das Archiv speichere ich mit der Version im Dateinamen und erstelle einen Link mit dem Namen `jenkins.war`
So kann ich mehrere Versionen lokal vorhalten und muss in den Startskripten nicht den Dateinamen anpassen.
`2.289.1` ist die aktuelle <abbr title="Long Term Support">LTS</abbr>-Version für Jenkins.

Starten lässt sich die Instanz nun mit dem Einzeiler `java -jar /home/jenkins/app/jenkins.war`.
Diesen habe ich im Skript `/home/jenkins/app/start_jenkins.sh` untergebracht.
Hierbei sollte nicht vergessen werden, Ausführungsrechte mittels `chmod +x start_jenkins.sh` zu gewähren.

Jenkins ist nun im Browser unter `http://jugong:8080`[^1] erreichbar.
Zur Authentisierung wird eine Zeichenkette benötigt, welche nach dem Start im Terminal angezeigt wird.
Die weitere Installation ist selbsterklärend, zunächst wird ein Nutzer generiert und anschließend **Plugins** installiert.
Bei letzterem habe ich bewusst nicht die Empfehlungen gewählt.
Stattdessen habe ich alles abgewählt und nur einige wichtige, etwa [Folders](https://plugins.jenkins.io/cloudbees-folder), [Git](https://plugins.jenkins.io/git) und [Pipeline](https://plugins.jenkins.io/workflow-aggregator) installiert.
Mit den dazugehörigen Abhängigkeiten landet dann eine gute Anzahl an Plugins im System.

# Autostart

Zu guter Letzt möchte ich, dass Jenkins beim Systemstart verfügbar ist.
Dazu verwendet Linux Mint **init.d**.
Ich muss aktuell gestehen, dass ich das System noch nicht ausreichend ergründet und verstanden habe.
Aber die hier und dort zusammengesammelten Dinge funktionieren erst einmal.

Zum Starten und Stoppen von Jenkins habe ich das **Skript** `/etc/init.d/jenkins` angelegt.
Es sollte dem Nutzer _root_ gehören und ausführbar sein.

{{< highlight bash "linenos=table">}}
#! /bin/sh
### BEGIN INIT INFO
# Provides:          MeinProgramm
# Required-Start:
# Required-Stop:
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Starts & Stops Jenkins
# Description:       Starts & Stops Jenkins
### END INIT INFO

JENKINS_USER=jenkins

#Switch case fuer den ersten Parameter
case "$1" in
    start)
 #Aktion wenn start uebergeben wird
        echo "Starting Jenkins" > /dev/console
        su --preserve-environment $JENKINS_USER -c /home/jenkins/application/start_jenkins.sh
        ;;

    stop)
 #Aktion wenn stop uebergeben wird
        echo "Stopping Jenkins" > /dev/console
        JENKINS_PID=$(ps -A -f | grep "^jenkins" | tr -s " " | cut -d" " -f2,8- | grep "java -jar /home/jenkins/application/jenkins.war" | cut -d" " -f1)
        su --preserve-environment root -c "kill -s TERM $JENKINS_PID"
        ;;

    restart)
 #Aktion wenn restart uebergeben wird
        echo "Restarting Jenkins" > /dev/console
        $0 stop
        $0 start
        ;;
 *)
 #Standard Aktion wenn start|stop|restart nicht passen
 echo "(start|stop|restart)"
 ;;
esac

exit 0
{{< / highlight >}}

Aktiviert wird das Ganze nun mittels der beiden Befehle
```
sudo update-rc.d jenkins defaults
sudo update-rc.d jenkins enable
```
Beim letzten Befehl erhalte ich aktuell noch eine Fehlermeldung, deren Bedeutung ich noch nicht klären konnte.
Jedoch funktioniert es soweit.
```
Failed to enable unit: Unit /run/systemd/generator.late/jenkins.service is transient or generated.
```

# Fazit
In diesem Artikel habe ich kurz beschrieben, wie ich Jenkins unter Linux Mint installiere und den Autostart einrichte.
Dies soll die Grundlage für alle weiteren Hobbyprojekte darstellen.

[^1]: Ich benenne meine Rechner in der Regel nach Pokémon der ersten Generation.
Die virtuelle Maschine trägt den Namen *Jugong*, ein Wasser/Eis-Pokémon.
