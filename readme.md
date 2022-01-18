# Docker

- [Docker](#docker)
  - [Container](#container)
    - [Erstellen eines Containers](#erstellen-eines-containers)
    - [Wichtige CLI-Befehle](#wichtige-cli-befehle)
    - [Inspizieren von Containern](#inspizieren-von-containern)
    - [Mit Containern interagieren](#mit-containern-interagieren)
  - [Docker Netzwerke](#docker-netzwerke)
    - [CLI-Management](#cli-management)
    - [DNS](#dns)
    - [DNS Round Robin](#dns-round-robin)
  - [Images](#images)
    - [Tags](#tags)
    - [Erstellen von Images](#erstellen-von-images)
    - [Build Images](#build-images)
    - [Extending Images](#extending-images)
  - [Volumes](#volumes)
    - [Container Lifetime & Persistent Data](#container-lifetime--persistent-data)
    - [Named Volumes](#named-volumes)
- [Docker Compose](#docker-compose)
- [Docker Swarm](#docker-swarm)
- [Kubernetes](#kubernetes)
- [Notizen](#notizen)
  - [Port vergabe](#port-vergabe)
  - [System bereinigen](#system-bereinigen)

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

- Jeder Container verbindet sich mit einem privaten Virtuellen Netzwerk 'bridge'.
- Jedes virtuelle Netzwerk nutzt die Firewall des Hosts.
- Alle Container des selben virtuellen Netzwerks können miteinander kommunizieren, ohne das weiter Ports mit -p gemappt werden.
- Best practice ist für jede App ein neues Netzwerk zu erstellen:
    - Netzwerk 'my_web_app' für mysql und php/apache Container.
    - Netzwerk 'my_api' für mongodb und nodejs Container.

### CLI-Management

```console
$ docker network ls                 // Netzwerke anzeigen
$ docker netwrok inspect            // Netzwerke inspizieren
$ docker network create --driver    // neues Netzwerk erstellen
$ docker network connect            // Netzwerk zu container hinzufügen
$ docker network disconnect         // Netzwerk von container entfernen
```

### DNS

Befinden sich zwei Container im selben Netzwerk ist es ein Anti Pattern diese mit der jeweiligen IP anzusprechen.
Die beiden Container können mittels des jeweiligen Container Namens kommunizieren.
<br>
Beispiel:

```console
$ docker network create web_app
$ docker container run -d -p 80:80 --name webhost --network web_app nginx
$ docker container run -d -p 3306:3306 --name database --network web_app mysql
```

Verbindung von webhost und database mittels Doctrine als beispiel 'mysql://user:password@web_app/db_name'

### DNS Round Robin

Mit '--network-alias' kann dem Container zusätzlicher DNS vergeben werden. Es können mehrere Container im selben Netzwerk den gleichen alias haben. Somit kann die Lastverteilung per DNS (Round Robin) genutzt werden.

## Images

Images enthalten die binaries und dependencies der App. Images enthalten *keinen* Kernel, dieser kommt vom Hosts System. Deswegen wird auch unter Windows beispielsweise WSL benötigt um Docker zu nutzen.
Die größte Quelle an Images ist [Docker Hub](hub.docker.com). Eigene Images können auch auf Docker Hub hochgeladen werden. Offizielle Images haben nur einen Namen. Images von Usern haben vorher noch den Username (USERNAME/IMAGENAME).

### Tags

Für gewöhnlich (vorallem bei offiziellen) habe Images verschiedene Tags. Diese geben beispielsweise an welche Version der Software im jeweiligen Image genutzt wird oder welches Image als Basis verwendet wird.
Somit kann man sicher gehen, dass die Software im auf dem Stand ist wie angegeben. Wird kein Tag angegeben wird das latest, sprich das aktuellste Image verwendet.<br>

Retag Images:

```console
$ docker image tag SOUCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

### Erstellen von Images

<pre>
FROM debian:jessie                      // required: specifies the base image

ENV NGINX_VERSION 1.11.10-1~jessie      // sets env variables

RUN apt-get update \
    && apt-get install ...              // runs shell commands inside the container

EXPOSE 80 443                           // exposes the ports to the host machine

CMD ["nginx", "-g", "daemon off"]       // required: run this command when conainer is launched
                                        // only one cmd allowed per Dockerfile
</pre>

### Build Images

```console
$ docker image build -t TAG .
```

Docker verfügt über einen sehr mächtigen Cache, was bedeutet, dass beim rebuilden eines images nur die veränderten Zeilen und die darauf folgenden ausgeführt werden, deshalb ist es best practice die Befehle die sich selten ändern am Anfang zu schreiben und später die die sich häufiger ändern.

### Extending Images

Hier ist ein [Beispiel](01_images/dockerfile-sample-2/Dockerfile).

<pre>
// this shows how we can extend/change an existing official image from Docker Hub

FROM nginx:latest
// highly recommend you always pin versions for anything beyond dev/learn

WORKDIR /usr/share/nginx/html
// change working directory to root of nginx webhost
// using WORKDIR is preferred to using 'RUN cd /some/path'

COPY index.html index.html

// I don't have to specify EXPOSE or CMD because they're in my FROM
</pre>

## Volumes

### Container Lifetime & Persistent Data

- Normalerweise sind Container unveränderlich (immutable) und flüchtig (ephemeral)
- "immutable infrastructure": container werden nur neu deployed, nicht verändert
- problematisch bei beispielweise Datenbanken, da die binaries im Container nicht mit unseren Daten vermischt werden sollten
- Es gibt zwei Möglichkeiten dieses Problem zu lösen: Volumes & Bind Mounts
- Volumes: spezieller Ort außerhalb des filesystem des Containers
- Bind Mounts: Verlinkung von Container Paths mit Host Paths

### Named Volumes

Wird nichts genauers angegeben werden volumes standardmäßig mit einem zufällig Namen generiert.

![named volumes image 1](assets/img/named_volumes_1.PNG)

# Docker Compose

# Docker Swarm

# Kubernetes

# Notizen

## Port vergabe

```console
-p 80:8080
```

linke Seite: Port der Host Maschine <br>
rechte Seite: Port des Containers <br>

<br>

Der Container wäre also über 127.0.0.1:80 erreichbar.

## System bereinigen

```console
$ docker system prune
$ docker image prune
```

Mit --help können weitere Infos zu breinigen ausgegeben werden.