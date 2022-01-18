# Docker Mastery

[Container](#container)<br>
[Docker Networks](#docker-networks)<br>
[Notizen](#notizen)

## Container

### Erstellen eines Containers

```console
docker container run -p HOST_PORT:CONTAINER_PORT --name CONTAINER_NAME IMAGE_NAME 
```

Mit -d wird der Container im detached mode, sprich im Hintergrund ausgeführt.<br>
Mit -e können zusätzliche Umgebungsvarablen angegeben werden.<br>

<br>

Beispiel nginx:

```console
$ docker container run -d --name webhost -p 80:80 nginx:latest
```

:latest gibt die Version des nginx images an (später dazu mehr).

### Wichtige CLI-Befehle

-------

```console
$ docker container ls 
```
Zeigt alle derzeit laufende Container an. Mit der flag -a werden alle Container, auch gestoppt angezeigt.

<br>

```console
$ docker container stop CONTAINER_NAME
```

Stopt den Container mit dem angegebenen Namen.

<br>

```console
$ docker rm CONTAINER_NAME
```

Entfernt unwiderruflich den Container mit dem angegeben Name. Der Container muss dafür bereits gestoppt sein, dies kann allerdings mit der flag -f umgangen werden.

<br>

```console
$ docker container top CONTAINER_NAME
```

Zeigt einige Informationen über den angegebenen Container an.

<br>

```console
$ docker container logs CONTAINER_NAME
```

Gibt die logs des Containers aus.

### Inspizieren von Containern

--------

```console
$ docker container stats CONTAINER_NAME
```

Zeigt die von dem Container derzeit genutzten Ressourcen an.

<br>


```console
$ docker container inspect CONTAINER_NAME
```

Gibt die sämtlichen Metadaten aus darüber wie der Container erstellt wurde.

### Mit Containern interagieren

--------

Zum interagieren mit Containern wird kein SSH benötigt. <br>
Folgender Befehl startet eine interaktive Shell im Container:

```console
$ docker exec -it CONTAINER_NAME bash
```

Die flag -it kann auch mit *docker container run* kombiniert werden um direkt beim erstellen des Containers mit diesem zu interagieren. *Bash* ist die Shell die im Container gestartet wird (Bash ist für gewöhnlich in fast jedem Container installiert und kann somit genutzt werden).

## Docker Netzwerke

- Jeder Container verbindet sich mit einem privaten Virtuellen Netzwerk 'bridged'.
- Jedes virtuelle Netzwerk nutzt die Firewall des Hosts.
- Alle Container des selben virtuellen Netzwerks können miteinander kommunizieren, ohne das weiter Ports mit -p gemappt werden.
- Best practice ist, für jede App ein neues Netzwerk zu erstellen
    - Netzwerk 'my_web_app' für mysql und php/apache Container.
    - Netzwerk 'my_api' für mongodb und nodejs Container.

### CLI-Management

```console
$ docker network ls                 //Netzwerke anzeigen
$ docker netwrok inspect            //Netzwerke inspizieren
$ docker network create --driver    //neues Netzwerk erstellen
$ docker network connect            //Netzwerk zu container hinzufügen
$ docker network disconnect         //Netzwerk von container entfernen
```

## Notizen

### Port vergabe

```console
-p 8080:80
```

linke Seite: Port der Host Maschine <br>
rechte Seite: Port des Containers <br>

<br>

Der Container wäre also über 127.0.0.1:8080 erreichbar.
