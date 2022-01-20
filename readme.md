# Docker Mastery


## Content

- [Docker](#docker)
- [Docker Compose](#docker-compose)
- [Docker Swarm](#docker-swarm)
- [Container Registries](#container-registries)
- [Kubernetes](#kubernetes)
- [Notizen](#notizen)
- [Inhaltsverzeichnis](#inhaltsverzeichnis)

# Docker

<!-- no toc -->
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
  - [Bind Mounts](#bind-mounts)

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

-------

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

CMD ["nginx", "-g", "daemon off"]       // required: run this command when container is launched
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

Konfigurations Beispiel in Dockerfile:

<pre>
...

VOLUME /var/lib/mysql

...
</pre>

### Named Volumes

Wird nichts genauers angegeben werden volumes standardmäßig mit einem zufällig Namen generiert.

![named volumes image 1](assets/img/named_volumes_1.PNG)

Für die Übersicht und um besser damit arbeiten zu können, kann dem volume ein bestimmter Name gegeben werden.
Das Schema lautet dabei `VOLUME_NAME:VOLUME_PATH_CONTAINER`.

![named volumes image 2](assets/img/named_volumes_2.PNG)

### Bind Mounts

- Mapped eine Host Datei oder Ordner auf eine Container Datei oder Ordner
- Im Prinzip zwei Ort die auf die selbe Datei zeigen
- Kann *nicht* in der Dockerfile angegeben werden, sondern muss bei *Container run* angegeben werden
- `docker container run -v HOST_PATH:CONTAINER_PATH`

```console
$ docker run -v //c/Users/username/stuff:/path/container    // windows
$ docker run -v /Users/username/stuff:/path/container       // mac/linux
```

Einfach ist es mit folgendem Befehl:

```powershell
$ docker run -v ${pwd}:/path/container                      // windows
```

```bash
$ docker run -v $(pwd):/path/container                      // mac/linux
```

Dafür muss man sich jedoch mit der Konsole im gewünschten Host Verzeichnis befinden.

# Docker Compose

- [Übersicht Docker Compose](#übersicht-docker-compose)
- [docker-compose.yml](#docker-composeyml)
- [docker-compose CLI](#docker-compose-cli)
- [Beispiel Konfiguration](#beispiel-konfiguration)
- [Build Images mit Compose](#build-images-mit-compose)
  - [Beispiel](#beispiel)

-------

## Übersicht Docker Compose

- konfigurieren der Beziehungen zwischen Containern
- speicher von `docker container run` Einstellungen in einer leicht verständlichen Datei
- Besteht aus zwei seperaten aber verbundenen Teilen
  - YAML Konfiguartionsdatei welche folgende Teile der Konfiguration beschreibt:
    - Container
    - Networks
    - Volumes
  - docker-compose CLI tool zum verarbeiten der YAML Dateien

*docker-compose ist hauptsächlich für lokale Entwicklung konzipiert, in der Produktion kann es jedoch mit docker swarm genutzt werden.*

## docker-compose.yml

Aufbau einer docker-compose Datei:

```yaml
version: 3.1
# wird keine Version angegeben, wird automatisch v1 verwendet
# empfohlen ist mindestens v2 zu nutzen

services:       # container, indentisch zu docker run
  servicename:  # DNS name innerhalb des Netzwerks
    image:      # optional wenn build verwendet wird
    command:    # optional, ersetzt standard CMD des images
    enviorment: # optional, indentisch zu -e bei docker run
    volumes:    # optional, indentisch zu -v bei docker run
    ports:      # optional, indentisch zu -p bei docker run
  servicename2: 
    ...
  
volumes:      # optional, indentisch zu docker volume create

networks:     # optional, indentisch zu docker network create 
```

Eine genaue Beschreibung sämtlicher Keywords findet man in der offiziellen [Dokumentation](https://docs.docker.com/compose/).

## docker-compose CLI

Die meist genutzten Befehle sind `docker-compose up` und `docker-compose down`.

`docker-compose up` erstellt die volumes/networks und startet alle Container. Die flag -d kann wie bei container run angegeben werden um im dettach mode zu starten.

`docker-compose down` stoppt alle Container und entfernt die Container/Volumes/Networks.

`docker-compose stop` stoppt alle Container ohne diese zu entfernen. Vorallem nützlich wenn die Container im dettach mode laufen.

## Beispiel Konfiguration

```yaml
version: '3'

services:
  drupal:
    image: drupal
    ports:
      - '8080:80'
    volumes:
      - drupal-modules:/var/www/html/modules
      - durpal-profiles:/var/www/html/profiles
      - drupal-themes:/var/www/html/themes
      - drupal-sites:/var/www/html/sites

  database:
    image: postgres
    environment:
      - POSTGRES_PASSWORD=123456

volumes:
  drupal-modules:
  durpal-profiles:
  drupal-themes:
  drupal-sites:
```

*Hinweis:* für die Datenbank muss in diesem Fall kein Port angegeben werden, da nur der drupal Container mit dieser interagiert. Es kann also bei der Konfiguration von drupal einfach der DNS der Datenbank angegeben werden als Addresse und der Standart Postgres Port. Würde man jetzt vom Host System aus auch mit der Datenbank kommunizieren wollen, müsste man auch einen Port mitangeben.

## Build Images mit Compose

- Compose kann auch eigene Images bauen
- Werden automatisch gebaut wenn `docker-compose up` ausgeführt wird
- Können auch mit `docker-compose build` manuell gebaut werden

### Beispiel

```Dockerfile
# Dockerfile

FROM drupal:8.8.2

RUN apt-get update \
    && apt-get install -y git \ 
    && rm -rf /var/lib/apt/lists/*

WORKDIR /var/www/html/themes

RUN git clone --branch 8.x-3.x --single-branch --depth 1 https://git.drupalcode.org/project/bootstrap.git \
    && chown -R www-data:www-data bootstrap

WORKDIR /var/www/html
```

```yaml
# docker-compose.yml

version: '3'

services:
  drupal:
    build: .
    ports:
      - '8080:80'
    volumes:
      - drupal-modules:/var/www/html/modules
      - durpal-profiles:/var/www/html/profiles
      - drupal-themes:/var/www/html/themes
      - drupal-sites:/var/www/html/sites

  database:
    image: postgres:12.1
    environment:
      - POSTGRES_PASSWORD=123456
    volumes:
      - drupal-data:/var/lib/postgresql/data

volumes:
  drupal-modules:
  durpal-profiles:
  drupal-themes:
  drupal-sites:
  drupal-data:
```

# Docker Swarm

<!-- no toc -->
- [Übersicht Docker Swarm](#übersicht-docker-swarm)
- [Docker Swarm aktivieren](#docker-swarm-aktivieren)
- [Erstellen eines 3-Node Swarm](#erstellen-eines-3-node-swarm)
- [Overlay Networking](#overlay-networking)
- [Routing Mesh](#routing-mesh)
- [Example Voting App](#example-voting-app)
  - [Services](#services)
  - [Steps](#steps)
- [Swarm Stack](#swarm-stack)
  - [CLI Befehle](#cli-befehle)
  - [Beispiel eines Stacks](#beispiel-eines-stacks)
- [Swarm Secrets](#swarm-secrets)
  - [Secret Storage](#secrets-storage)
  - [Erstellen eines Secrets](#erstellen-eines-secrets)
    - [Mit einer Datei](#mit-einer-datei)
    - [Über die Konsole](#über-die-konsole)
  - [Verwenden von Secrets](#verwenden-von-secrets-am-beispiel-postgres)
  - [Verwenden von Secrets mit Stacks](#verwenden-von-secrets-mit-stacks)
- [Service Updates](#service-updates)
- [Docker Healthchecks](#--docker-healthchecks)
  - [Docker Run Beispiel](#healthcheck-docker-run-beispiel)
  - [Dockerfile Beispiel](#healthcheck-dockerfile-beispiel)
    - [Healthcheck Nginx Dockerfile](#healthcheck-in-nginx-dockerfile)
    - [Healthcheck Postgres Dockerfile](#healthcheck-in-postgres-dockerfile)
  - [Compose/Stack Beispiel](#composestack-beispiel)
  
-------

## Übersicht Docker Swarm

- Swarm Mode ist eine Cluster Lösung die in Docker mit eingebaut ist
- Swarm ist standartmäßig nicht aktiviert
- Wird es aktiviert bietet es Zugriff auf folgende Befehle:
  - docker swarm
  - docker node
  - docker service
  - docker stack
  - docker secret
  
![How nodes work](https://docs.docker.com/engine/swarm/images/swarm-diagram.png)

## Docker Swarm aktivieren

Swarm kann mit `docker swarm init` aktiviert werden.

## Erstellen eines 3-Node Swarm

Nachdem im ersten Node swarm initalisiert wurde erscheint folgende Ausgabe im Terminal:

![swarm 1](assets/img/swarm_1.PNG)

Der in der mitte stehende Befehl kann nun in den anderen beiden Nodes ausgeführt werden um diese zum Cluster hinzuzufügen. Mit `docker node promote node2 node3` werden die anderen beiden Nodes zum Manager. <br>

Mit `docker node ls` bekommen wir die Nodes des swarm clusters aufgelistet.

Als nächstes fügen wir einen Service mit 3 Tasks hinzu.

```console
$ docker service create --replicas 3 alpine ping 8.8.8.8
```

Mit `docker service ls` bekommen wir die Services aufgelistet

Wenn wir `docker node ps node2` ausführen, wird aufgelistet welche Tasks auf dem jeweils angegebenen Node läuft. `docker service ps SERVICE_NAME` zeigt alle Tasks des jeweiligen Service an und auf welchem Node sie laufen.

## Overlay Networking

- Bei erstellen eines Networks einfach `--driver overlay` angeben
- Verwendung für container-to-container traffic innerhalb eines Swarms
- Jeder Service kann sich mit keinem, einen oder mehreren Netzwerken verbinden

## Routing Mesh

- Routet eingehende Pakete für einen Service zum entsprechenden Task
- Spannt sich über alle Nodes im Swarm
- Nutzt IPVS des Linux Kernels ([Wikipedia](https://en.wikipedia.org/wiki/IP_Virtual_Server))
- Funktioniert auf zwei Arten:
  - Container zu Container innerhalb eines overlay networks
  - Externer eingehender traffic auf einen offenen Port

Einfach ausgedrückt: wenn wir uns mit einem Task verbinden wollen, müssen wir nur die IP von einem beliebigen Node angeben, es ist egal in welchem Node der Task läuft.

Damit kommt es jedoch auch dazu, dass der load balancer von Swarm uns bei mehreren Anfragen auf unterschiedliche Tasks routet (da es sich um einen OSi-Layer 3 load balancer handelt). Um dieses Problem zu beheben wird noch ein zusätzlicher proxy load balancer (OSi-Layer 4) benötigt. Dieses Problem tritt nur auf wenn es von einem Service mehrere Replicas gibt.

## Example Voting App

![architecture](assets/img/architecture.png)

### Services

- vote
  - bretfisher/examplevotingapp_vote
  - web front end for users to vote dog/cat
  - ideally published on TCP 80. Container listens on 80
  - on frontend network
  - 2+ replicas of this container

- redis
  - redis:3.2
  - key/value storage for incoming votes
  - no public ports
  - on frontend network
  - 1 replica

- worker
  - bretfisher/examplevotingapp_worker
  - backend processor of redis and storing results in postgres
  - no public ports
  - on frontend and backend networks
  - 1 replica

- db
  - postgres:9.4
  - one named volume needed, pointing to /var/lib/postgresql/data
  - on backend network
  - 1 replica
  - remember set env for password-less connections -e POSTGRES_HOST_AUTH_METHOD=trust

- result
  - bretfisher/examplevotingapp_result
  - web app that shows results
  - runs on high port since just for admins (lets imagine)
  - so run on a high port of your choosing (I choose 5001), container listens on 80
  - on backend network
  - 1 replica

### Steps

```console
$ docker swarm init
```

Danach alle Nodes zum Swarm hinzufügen, zum Manager promoten und anschließend die Netzwerke erstellen:

```console
$ docker network create --driver overlay fontend

$ docker network create --driver overlay backend
```

Als nächstes die Services erstellen:

```console
$ docker service create --replicas 2 --name vote -p 80:80 --network frontend bretfisher/examplevotingapp_vote

$ docker service create --name redis --network frontend redis:3.2

$ docker service create --name worker --network frontend --network backend bretfisher/examplevotingapp_worker

$ docker service create --name db --network backend --mount type=volume,source=db-data,target=/var/lib/postgresql/data -e POSTGRES_HOST_AUTH_METHOD=trust postgres:9.4

$ docker service create --name result -p 5001:80 --network backend bretfisher/examplevotingapp_result
```
Danach sollte die Voting App auf Port 80 verfügbar sein und unter Port 5001 das live Ergebniss.

## Swarm Stack

Swarm Stack ist im Prinzip Compose auf der Produktions Ebene.

- Service, Netzwerke und Volumes können in der Compose Datei für Swarm definiert werden
- Wird mit dem Befehl `docker stack deploy` verwendet
- build ist nicht verfügbar in den stack compose Dateien, dafür deploy
- Compose ignoriert deploy, Stack ignoriert build
- docker-compose CLI wird auf einem Swarm Server nicht benötigt

*build sollte niemals in Produktion stattfinden*

### CLI Befehle

Deployen/Updaten eines Stacks:

```console
$ docker stack deploy -c COMPOSE_FILE_NAME STACK_NAME
```

Auflisten aller Stacks:

```console
$ docker stack ls
```

Auflisten der Services eines Stacks:

```console
$ docker stack services STACK_NAME
```

Entfernen eines Stacks:

```console
$ docker stack rm STACK_NAME
```

### Beispiel eines Stacks

```yaml
version: "3"
services:

  redis:
    image: redis:alpine
    networks:
      - frontend
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    environment:
      - POSTGRES_HOST_AUTH_METHOD=trust
    deploy:
      placement:
        constraints: [node.role == manager]
  vote:
    image: bretfisher/examplevotingapp_vote
    ports:
      - 5000:5000
    networks:
      - frontend
    depends_on:
      - redis
    deploy:
      replicas: 5
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure

  result:
    image: bretfisher/examplevotingapp_result
    ports:
      - 5001:80
    networks:
      - backend
    depends_on:
      - db
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  worker:
    image: bretfisher/examplevotingapp_worker
    networks:
      - frontend
      - backend
    depends_on:
      - db
      - redis
    deploy:
      mode: replicated
      replicas: 1
      labels: [APP=VOTING]
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints: [node.role == manager]

  visualizer:
    image: bretfisher/visualizer
    ports:
      - 8080:8080
    stop_grace_period: 1m30s
    networks:
      - frontend
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  frontend:
  backend:

volumes:
  db-data:
```

## Swarm Secrets

### Secrets Storage

- Einfachste sichere Lösung um Secrets in Swarm zu speichern
- Was sind Secrets?
  - Benutzernamen und Passwörter
  - TLS Zertifikate und keys
  - API Keys
  - SSH Keys
  - Alle Daten auf die Nutzer nicht einsehen dürfen
- Supportet strings oder binäre Datein bis 500Kb
- Apps müssen nicht umgebaut werden um diese Funktion zu nutzen
- Secrets werden erst in Swarm gespeichert und dann dem Service zugewiesen
- /run/secrets/<secret_name> oder /run/secrets/<secret_alias>

Secrets können mit `docker service update --secret-rm` entfernt oder mit `docker service update --secret-add` hinzugefügt werden. *ACHTUNG*: Dabei wird der Container neu erstellt!

### Erstellen eines Secrets

#### Mit einer Datei

```console
$ docker secret create SECRET_NAME SECRET_FILE
```

#### Über die Konsole

```console
$ echo "SECRET" | docker secret create SECRET_NAME -
```

### Verwenden von Secrets (am Beispiel Postgres)

```console
docker service create --name psql \
  --secret psql_user --secret psql_pass -e \
  POSTGRES_PASSWORD_FILE=/run/secrets/psql_pass \
  POSTGRES_USER_FILE=/run/secrets/psql_user postgres
```

### Verwenden von Secrets mit Stacks

```yaml
version: '3.1' # min. für Secrets

services:
  drupal:
    image: drupal:8.2
    ports:
      - '8080:80'
    volumes:
      - drupal-modules:/var/www/html/modules
      - durpal-profiles:/var/www/html/profiles
      - drupal-themes:/var/www/html/themes
      - drupal-sites:/var/www/html/sites

  database:
    image: postgres:12.1
    secrets:    # gibt dem Service Zugriff auf die jweiligen Secrets
      - psql_user
      - psql_password
    environment:
      - POSTGRES_USER_FILE=/run/secrets/psql_user
      - POSTGRES_PASSWORD_FILE=/run/secrets/psql_password
    volumes:
      - drupal-data:/var/lib/postgresql/data

secrets:
  psql_user:
    external: true  # wenn das Secret im Swarm bereits vorhanden ist
                    # beispielsweise wenn es über die CLI erstellt wurde
  psql_password: 
    file: ./psql_password.txt   # wenn das Secret aus einer Datei kommt 

volumes:
  drupal-modules:
  durpal-profiles:
  drupal-themes:
  drupal-sites:
  drupal-data:
```

## Service Updates

- Ermöglich 'rolling replacement'* von Tasks/Containern in einem Service
- Bei den meisten Änderungen werden Container ersetzt
- rollback und healthcheck Optionen
- stack deploy bei einem bereits existierenden Stack triggert ein Update

**ein Container nach dem anderen wird geupdatet um downtime zu minimieren*

### Update Beispiele

- Ein Image auf eine neue Version updaten:
  - `docker service update --image myapp:1.2.1 <servicename>`
- Hinzufügen einer Umgebungsvariable und entfernen eines Ports:
  - `docker service update --env-add NODE_ENV=production --publish-rm 8080`
- Anzahlen der Replicas von zwei Servicen ändern:
  - `docker service scale web=8 api=6`

Beim updaten von Stacks sollte darauf verzichtet werden Updates direkt über die CLI durchzuführen, sondern man sollte es immer in der compose Datei ändern.

## Docker Healthchecks

- Unterstützt in Dockerfile, Compose YAML, docker run und Swarm Services
- Wird innerhalb des Containers ausgeführt
- Erwartet exit 0 (OK) oder exit 1 (error)
- 3 container states: starting, healthy, unhealthy
- Aktueller Healthcheck wird bei `docker container ls` angezeigt
- Die letzten 5 Healthchecks können mit `docker container inspect` eingesehen werden
- Services ersetzen den jeweiligen Task wenn der healthcheck fehlschlägt
- Service updates wartet auf den healthcheck bevor das Update fortgesetzt wird

### Healthcheck Docker Run Beispiel

```console
$ docker run \
  --health-cmd="curl -f localhost:9200/_cluster/health || false" \
  --health-interval=5s \
  --health-retries=2 \
  --health-timeout=2s \
  --health-start-period=15s \
  elasticsearch:2
```

### Healthcheck Dockerfile Beispiel

```dockerfile
HEALTHCHECK --interval=5m --timeout=3s \
  CMD curl -f http://localhost/ || false
```

Verfügbar Optionen:

- --interval=DURATION (default: 30s)
- --timeout=DURATION (default: 30s)
- --start-period=DURATION (default: 0s)
- --retries=N (default: 3)

#### Healthcheck in Nginx Dockerfile

```dockerfile
FROM nginx:1.13

HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost/ || exit 1
```

#### Healthcheck in Postgres Dockerfile

```dockerfile
FROM postgres:latest

# specify real user with -U to prevent errors in log

HEALTHCHECK --interval=5s --timeout=3s \
  CMD pg_isready -U postgres || exit 1
```

### Compose/Stack Beispiel

```yaml
version: '2.1'    # min. Version für healthchecks

services: 
  web:
    image: nginx:latest
    healthcheck:
      test: ["CMD","curl","-f","http://localhost"]
      interval: 1m30s
      timeout: 10s
      retries: 2
      start_period: 1m    # min. Version 3.4
```

# Container Registries

## Docker Hub

- Größte öffentliche image registry
- Erlaubt es Images zu bauen
- Verlinkung zu GitHub/BitBucket um automatisch Images zu bauen bei commit
- Verkettung von Image Builds
- Dockers standard registry

## Docker Registry

- Eine private registry für das eigene Netzwerk
- Im Vergleich zu Docker Hub nur sehr wenige Features
- Im Kern eine web API
- Kann in der Cloud deployed werden

[Link](https://hub.docker.com/_/registry) zum offiziellen Image.

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
$ docker volume prune
```

Mit --help können weitere Infos zu breinigen ausgegeben werden.

## Named Volumes in Docker Swarm

```bash
# pattern:
--mount type=volume,source=VOLUME_NAME,target=/path/to/volume

# example:
--mount type=volume,source=db-data,target=/var/lib/postgresql/data
```

## Rebalance Nodes Docker Swarm

```console
$ docker service update --force web
```

# Inhaltsverzeichnis

- [Docker Mastery](#docker-mastery)
  - [Content](#content)
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
    - [Bind Mounts](#bind-mounts)
- [Docker Compose](#docker-compose)
  - [Übersicht Docker Compose](#übersicht-docker-compose)
  - [docker-compose.yml](#docker-composeyml)
  - [docker-compose CLI](#docker-compose-cli)
  - [Beispiel Konfiguration](#beispiel-konfiguration)
  - [Build Images mit Compose](#build-images-mit-compose)
    - [Beispiel](#beispiel)
- [Docker Swarm](#docker-swarm)
  - [Übersicht Docker Swarm](#übersicht-docker-swarm)
  - [Docker Swarm aktivieren](#docker-swarm-aktivieren)
  - [Erstellen eines 3-Node Swarm](#erstellen-eines-3-node-swarm)
  - [Overlay Networking](#overlay-networking)
  - [Routing Mesh](#routing-mesh)
  - [Example Voting App](#example-voting-app)
    - [Services](#services)
    - [Steps](#steps)
  - [Swarm Stack](#swarm-stack)
    - [CLI Befehle](#cli-befehle)
    - [Beispiel eines Stacks](#beispiel-eines-stacks)
  - [Swarm Secrets](#swarm-secrets)
    - [Secrets Storage](#secrets-storage)
    - [Erstellen eines Secrets](#erstellen-eines-secrets)
      - [Mit einer Datei](#mit-einer-datei)
      - [Über die Konsole](#über-die-konsole)
    - [Verwenden von Secrets (am Beispiel Postgres)](#verwenden-von-secrets-am-beispiel-postgres)
    - [Verwenden von Secrets mit Stacks](#verwenden-von-secrets-mit-stacks)
  - [Service Updates](#service-updates)
    - [Update Beispiele](#update-beispiele)
  - [Docker Healthchecks](#docker-healthchecks)
    - [Healthcheck Docker Run Beispiel](#healthcheck-docker-run-beispiel)
    - [Healthcheck Dockerfile Beispiel](#healthcheck-dockerfile-beispiel)
      - [Healthcheck in Nginx Dockerfile](#healthcheck-in-nginx-dockerfile)
      - [Healthcheck in Postgres Dockerfile](#healthcheck-in-postgres-dockerfile)
    - [Compose/Stack Beispiel](#composestack-beispiel)
- [Container Registries](#container-registries)
  - [Docker Hub](#docker-hub)
  - [Docker Registry](#docker-registry)
- [Kubernetes](#kubernetes)
- [Notizen](#notizen)
  - [Port vergabe](#port-vergabe)
  - [System bereinigen](#system-bereinigen)
  - [Named Volumes in Docker Swarm](#named-volumes-in-docker-swarm)
  - [Rebalance Nodes Docker Swarm](#rebalance-nodes-docker-swarm)
- [Inhaltsverzeichnis](#inhaltsverzeichnis)