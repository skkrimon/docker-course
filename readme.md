# Docker Mastery

[Container](#container)<br>
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

## Notizen

### Port vergabe

```console
-p 8080:80
```

linke Seite: Port der Host Maschine <br>
rechte Seite: Port des Containers <br>

<br>

Der Container wäre also über 127.0.0.1:8080 erreichbar.
