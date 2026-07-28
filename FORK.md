# Fork-Hinweise

Dieser Fork weicht in drei Punkten vom Upstream
([rosskouk/asknavidrome](https://github.com/rosskouk/asknavidrome)) ab.

## 1. Playlist-Namen über die Slot-Resolution auflösen

**Datei:** `skill/app.py`
**Branch:** `playlist-resolution` (Kandidat für einen Upstream-PR)

### Problem

Der Handler `NaviSonicPlayPlaylist` las bisher das rohe `value` des Slots aus:

```python
playlist = get_slot_value_v2(handler_input, 'playlist')
playlist_id = connection.search_playlist(playlist.value)
```

Das rohe `value` ist das, was Alexa akustisch verstanden hat — nicht der Wert
aus dem Slot Type. Bei nicht-englischen Playlist-Namen weicht beides
regelmäßig voneinander ab:

| Gesprochen              | `slot.value` | Resolution     |
|-------------------------|--------------|----------------|
| „spiele Playlist Charts" | `charts`     | `Hörercharts`  |
| (zweiter Versuch)        | `werner charts` | `Hörercharts` |

Die Suche in Navidrome lief also mit `charts` und lieferte
`No playlist matching the name charts was found!`, obwohl der Slot Type
sauber mit `ER_SUCCESS_MATCH` aufgelöst hatte.

Synonyme im Slot Type helfen nicht, weil der Code die Resolution nie liest.

### Lösung

Neuer Helper in `skill/app.py`:

```python
def resolve_slot_value(slot_value):
    """Return the slot type resolution if one matched, else the raw spoken value."""
    try:
        for authority in slot_value.resolutions.resolutions_per_authority:
            if 'ER_SUCCESS_MATCH' in str(authority.status.code):
                return authority.values[0].value.name
    except (AttributeError, IndexError, TypeError):
        pass

    return slot_value.value
```

Im Handler wird `playlist.value` an allen drei Stellen durch `playlist_name`
ersetzt (Suche, Fehlermeldung, Sprachausgabe).

Der Vergleich läuft bewusst über `'ER_SUCCESS_MATCH' in str(...)` statt über
Gleichheit: Die ASK-SDK liefert ein Enum, dessen String-Repräsentation sich
zwischen Versionen unterscheidet.

### Wirkung

Playlist-Namen sind frei wählbar — Umlaute, Komposita, beliebige
Schreibweisen. Transkriptionsvarianten fängt man über `synonyms` im Slot Type
ab, ohne die Playlist in Navidrome umbenennen zu müssen.

Nur der Playlist-Slot ist betroffen. `AMAZON.Artist`, `AMAZON.MusicAlbum`,
`AMAZON.MusicRecording` und `AMAZON.Genre` sind offene Typen ohne Resolution;
dort fällt der Helper automatisch auf `slot_value.value` zurück.

## 2. Dockerfile baut den lokalen Stand

**Datei:** `Dockerfile`
**Nur lokal — gehört nicht in einen Upstream-PR.**

Der Upstream-Build holt sich den Quellcode während des Builds von GitHub:

```dockerfile
RUN git clone https://github.com/rosskouk/asknavidrome.git
```

Damit ignoriert `docker build` das Arbeitsverzeichnis vollständig. Lokale
Änderungen landen nie im Image — das Image ist frisch gebaut, der Inhalt
stammt aber unverändert vom Upstream. Ersetzt durch:

```dockerfile
COPY . /opt/asknavidrome
```

`git` bleibt in der `apk add`-Zeile, weil einzelne pip-Abhängigkeiten es
benötigen.

## 3. Deutsches Interaction Model

**Datei:** `alexa.de-DE.json`

Übersetzung von `alexa.json` für die Locale `de-DE`. Unverändert bleiben
Intent-Namen, Slot-Namen und Slot-Typen — nur `invocationName`, die
`samples` und die Werte unter `playlist_names` sind angepasst.

Die Sprachausgabe des Skills bleibt englisch; die Strings sind fest in
`app.py` und `controller.py` hinterlegt und wurden bewusst nicht übersetzt,
um Rebases klein zu halten.

## Build

```bash
cd ~/docker/asknavidrome
docker compose up -d --build
```

Die Compose-Datei zeigt per `build.context` auf dieses Repo:

```yaml
services:
  asknavidrome:
    build:
      context: /home/sven/src/asknavidrome
    image: asknavidrome:local
```

Prüfen, ob der Patch im laufenden Container ist:

```bash
docker compose exec asknavidrome grep -c resolve_slot_value app.py   # erwartet: 2
```

## Upstream nachziehen

```bash
git fetch upstream
git stash                      # Dockerfile-Änderung parken
git rebase upstream/main
git stash pop
docker compose up -d --build
```

Die Änderung an `app.py` bleibt bewusst ein einzelner kleiner Commit, damit
Rebases konfliktfrei durchlaufen.
