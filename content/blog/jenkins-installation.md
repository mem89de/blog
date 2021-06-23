---
title: "Jenkins Installation"
date: 2021-06-23T22:06:16+02:00
draft: true
---

Für meine Projekte möchte ich eine Jenkins-Instanz in einer virtuellen Maschine auf meinem Notebook einrichten.
Dieser Artikel beschäftigt sich mit der initialen Installation und der Einrichtung des Auto-Starts.

<!--more-->

# Das Host-System
Für Jenkins möchte ich gerne ein unterliegendes Linux verwenden.
Aus der Erfahrung weiß ich, dass Jenkins sich unter Windows gerne ab und an zickig anstellt.
Das kann im einfachsten Fall der Verzeichnis-Trenner `\`, beziehungsweise `/ sein`.
Ich habe mich für **Linux Mint (Xfce) in einer VirtualBox** entschieden.

Hierbei sind erst einmal wenige Einstellungen nötig.
In den Netzwerkeinstellungen der virtuellen Maschine in VirtualBox wähle ich den Modus **Netzwerkbrücke**.
So ist später die Oberfläche des Jenkins auch aus dem Browser anderer Maschinen erreichbar.

Im Linux selbst soll der Jenkins unter einem eigenen **User** laufen.
Dieser wird mit den folgenden Befehlen einfach angelegt:

{{< highlight bash "linenos=table">}}
sudo -s
adduser jenkins
groupadd jenkins_group
usermod -a -G jenkins_group jenkins
{{< / highlight >}}

Damit sind die Vorbereitungen abgeschlossen.

# Installation von Jenkins
Es gibt verschiedenste Methoden, Jenkins zum Laufen zu bringen.
Ich entscheide mich vorerst für die einfachste Variante&mdash;das direkte **Ausführen des Jar**.
Eine Alternative wäre beispielsweise Docker.
Gegebenenfalls möchte ich dies zu einem späteren Zeitpunkt probieren.

Als Installationsverzeichnis wähle ich `/home/jenkins/app`.
Das Jar lade ich mit `wget https://get.jenkins.io/war-stable/2.289.1/jenkins.war -O jenkins.2.289.1.war` herunter.
Anschließend erstelle ich einen Link mit `ln jenkins.2.289.1.war jenkins.war`.
So kann ich mehrere Versionen lokal vorhalten und muss in den Startskripten nicht den Dateinamen anpassen.
2.289.1 ist die aktuelle <abbr title="Long Term Support">LTS</abbr>-Version für Jenkins.

Starten lässt sich die Jenkins-Instanz nun mit dem Einzeiler `java -jar /home/jenkins/app/jenkins.war`.
Diesen habe ich in der Skript-Datei `/home/jenkins/app/start_jenkins.sh` untergebracht.
Hierbei sollte nicht vergessen werden, Ausführungsrechte mittels `chmod +x start_jenkins.sh` zu gewähren.
