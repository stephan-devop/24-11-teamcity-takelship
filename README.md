### November 2024

# Das TeamCity takelship-Projekt

1. [Welche Vorteile bieten lokale Entwicklungsumgebungen?](#lokal)
2. [Was ist das takelship?](#takelship)
3. [Welches Problem löst das takelship?](#probleme)
4. [Welche Vorteile bietet das takelship?](#vorteile)
5. [Wie funktioniert das praktisch?](#praktisch)
6. [Welche Technologien werden verwendet?](#technologien)
7. [Wie ist das takelship aufgebaut?](#takelship)
8. [Wie sind takelship-Projekte aufgebaut?](#projekte)
9. [Welche Einschränkungen gibt es?](#limits)

<a id="lokal"></a>
## Welche Vorteile bieten lokale Entwicklungsumgebungen?

- Lokale Entwicklung ermöglicht lokale Tests
- Lokale Tests laufen in CI/CD-Pipelines
- Identischen Umgebungen ermöglichen reproduzierbare Ergebnisse
- Vollständige Kontrolle erweitert Debugging-Möglichkeiten
- Vollständige Isolation ermöglicht Experimente ohne Angst
- Deployment einer lokal entwickelten Pipeline: `git remote add` und `git push` 

<a id="takelship"></a>
## Was ist das takelship?

- Ein Generator von Docker Compose-Setups
- Eine Laufzeitumgebung für Docker Compose-Setups

Das [takelship](https://github.com/takelwerk/takelship/) ist ein Framework zum unkomplizierten Deployment komplexer Systeme. Ziel ist es, mit einem Befehl ([ship](https://github.com/takelwerk/takelage-cli)) ein vollständiges Setup nahezu beliebiger Software zu deployen.

<a id="probleme"></a>
## Welches Problem löst das takelship?

Ziel ist die Erstellung von TeamCity-Pipelines, die mit unterschiedlichen Parametrisierungen sowohl lokal als auch online laufen. Diese Pipelines sind selber Code und können dann mit der lokalen Parametrisierung in einer Pipeline getestet werden.

`ship start teamcity` erzeugt eine vollständige lokale Entwicklungsumgebung:

- TeamCity Server (Oberfläche wie teamcity.com)
- TeamCity Runner (Für die TeamCity-Pipelines)
- Forgejo Server (Simuliert GitHub)
- Forgejo Runner (Simulieren GitHub Runner)
- Registry Server (Simuliert DockerHub Registry)
- Registry UI (Simuliert DockerHub UI)
- Cache Server (Beispiel: Apt Proxy)

<a id="vorteile"></a>
## Welche Vorteile bietet das takelship?

- *zeitsparend:*
  Komplexe Software-Setups lassen sich in Minuten nutzen.

- *einfach:*
  Die Komplexität wird versteckt, aber dokumentiert und getestet.

- *rekonstruierbar:*
  Die Projekte können mit wenigen Befehlen gelöscht und neu erzeugt werden.

- *identisch:*
  Die Projekte sind auf allen Hostsystemen gleich.

- *redundant:*
  Die Projekte funktionieren auch ohne takelship auf dem Host.

- *portabel:*
  Die Projekte lassen sich per git verteilen und sichern.

- *unaufdringlich:*
  Das takelship erfordert minimale Eingriffe ins Hostsystem:\
  1 gem + 1 image / host\
  1 folder + 1 container / project

- *quelloffen und frei:*
  Das [takelship](https://github.com/takelwerk/takelship/) und [ship](https://github.com/takelwerk/takelage-cli) stehen unter der GNU General Public License auf GitHub.

<a id="praktisch"></a>
## Wie funktioniert das praktisch?

Als Frontend dient eine in Ruby geschriebene App namens `ship`, welche die git-Grammatik nachahmt. Die App wird per Behavior-driven development entwickelt und getestet.

```bash
mkdir teamcity
cd teamcity
ship start teamcity
```

Dieser Befehl hat einen takelship Docker-Container gestartet:

```bash
docker ps --format json | jq
```

Die `ship`-Konfiguration des Projekts ist in der `takelage.yml`. Sie besteht im Normalfall nur aus den lokalen Ports, denn ansonsten könnte wegen Portkonflikten auf jedem Rechner nur ein takelship gestartet werden. Die Ports können hier geändert oder abgeschaltet werden. Die neue Konfiguration wird bei einem `ship restart` in die Docker Compose-Konfiguration übertragen.

```bash
bat takelage.yml
```

Alle Dateien des Projekts befinden sich in einem Verzeichnis `takelship`.

```bash
tree takelship/compose
```

Die Logs des aktuellen Projekts zeigt `ship logs`, was lediglich ein `docker logs` auf das „richtige“ takelship ist:

```bash
ship logs -f
```

Die Images der Container liegen in einem Verzeichnis:

```bash
du -hs takelship/cache
```

Auf dem Host läuft nur ein Container:

```bash
docker ps
```

Aber im takelship laufen viele:

```bash
ship podman ps
```

Mittlerweile sollten die Services erreichbar sein. Wenn es bunt wird, ist der Bootvorgang fertig:

```bash
ship logs -f
```

So lassen wir die Ports und Passwörter der Diensta nochmal anzeigen, die wir nun ansurfen können:

```bash
ship start
```

Wenn wir nun zum Beispiel in Forgejo ein Repository anlegen, dann ist das auch nutzbar, wenn die Services *mit den gleichen Daten* und *unter den gleichen Ports* auf dem Host gestartet werden. Dafür stoppen wir das takelship, um die Ports freizugeben:

```bash
ship stop
```

Es läuft kein takelship-Container mehr:

```bash
docker ps
```

Jetzt können wir das takelship auf dem Host starten:

```bash
takelship/compose/projects/teamcity/docker-compose-up
```

Das Aufräumen nicht vergessen:

```bash
takelship/compose/projects/teamcity/docker-compose-down
```

<a id="technologien"></a>
## Welche Technologien werden verwendet?

### Verwendete Technologien

Debian, Docker, Podman, Packer, Molecule

### Verwendete Sprachen

Ansible, Jinja2, Python, Bash, Ruby

### Systemvoraussetzungen

#### `takelship` Hypervisor:

- Docker Desktop Mac (arm64)
- Docker Desktop Linux (amd64)
- Docker Engine Linux (amd64 & arm64)

#### `ship` Command Line Interface:

- Ruby

<a id="takelship"></a>
## Wie ist das takelship aufgebaut?

Das takelship ist ein Docker Container dessen Image auf `debian:stable-slim` basiert. Im takelship ist Podman als Hypervisor installiert.

Das takelship erzeugt Projektdateien in einem reingereichten Verzeichnis des Hosts. Anschließend führt es die Docker Compose-Dateien mit podman-compose im takelship aus.

Aus der Sicht des Hosts läuft nur ein Container, aber durch die Nested Virtualization laufen tatsächlich beliebig viele Container in diesem einen Host.

Die Ports der Services werden durch `podman` auf den localhost des takelships gemappt. Die Ports des takelships werden durch `docker` auf die Ports des localhost des Hosts gemappt.

Die `podman`-Portkonfiguration im takelship ist statisch, aber die die `docker`-Portkonfiguration auf dem Host wird durch `ship` dynamisch konfiguriert. Die `ship`.Konfiguration lässt sich über Konfigurationsdateien und/oder Umgebungsvariablen steuern. Siehe: `tau config`

Der Podman-Socket des takelships wird auf dem localhost des Hosts gemapped und ist via DOCKER_HOST ansprechbar. `ship podman` hingegen macht einen `docker exec`-Aufruf.

<a id="projekte"></a>
## Wie sind takelship-Projekte aufgebaut?

Ein Projekt besteht aus Daten und Konfigurationen.
Software und Laufzeitumgebung sind nicht Teil eines Projekts.

Ein takelship-Projekt ist ein Verzeichnis mit:
- einer `docker-compose.yml`, die Compose-Dateien von einem oder mehreren takelship-Services einbindet
- Startskripten (`run-podman` im takelship und `docker-compose-up` auf dem Host)

Ein takelship-Service ist eine Verzeichnis mit:
- einer `docker-compose.yml` mit genau einem Docker-Image
- Konfigurationsdateien des Services
- Konfigurationsskripten des Services
- Startskripten des Services

Komplexität wird versteckt. Beispiele:
- Der Service „teamcity-runner“ benötigt ein [Preinstall-Skript](https://github.com/takelwerk/takelship/blob/main/ansible/roles/takel_ship_teamcity/templates/run-preinstall.teamcity-runner.bash.j2), um die Architektur des Docker-Images anzupassen.
- Der Service „forgejo-server“ benötigt ein [Postinstall-Skript](https://github.com/takelwerk/takelship/blob/main/ansible/roles/takel_ship_forgejo/templates/run-postinstall.forgejo-server.bash.j2), um einen User anzulegen und ein Token für den Befehl „forgejo“ zu erzeugen
- Der Service „forgejo-runner“ benötigt ein [Preinstall-Skript](https://github.com/takelwerk/takelship/blob/main/ansible/roles/takel_ship_forgejo/templates/run-preinstall.forgejo-runner.bash.j2), um die Runner beim Forgejo-Server zu registrieren.

<a id="limits"></a>
## Welche Einschränkungen gibt es?

Das takelship funktioniert nur mit Docker auf dem Host, nicht mit Podman.

Docker Desktop für Linux hat (grundsätzlich) leichte Probleme mit Dateirechten, für die es aber einen [Workaround](https://docs.docker.com/desktop/faqs/linuxfaqs/#how-do-i-enable-file-sharing) gibt.
