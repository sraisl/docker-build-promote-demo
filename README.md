# docker-build-promote

Beispielprojekt: content-hash-basierte Docker-Builds mit Snapshot/Release-Promotion über GitHub Actions und GHCR — fast ausschließlich mit Marketplace-Actions, ohne eigene Shell-Skripte.

## Funktionsweise

### CI ([.github/workflows/ci.yml](.github/workflows/ci.yml))

Läuft bei Pushes auf `main` und bei Pull Requests:

1. **Hadolint** ([hadolint/hadolint-action](https://github.com/hadolint/hadolint-action)) prüft das `Dockerfile`. Schlägt der Lint fehl, passiert nichts weiter.
2. **Content-Hash**: die eingebaute Expression `hashFiles('Dockerfile', '.dockerignore', 'app/**')` berechnet einen SHA-256 über alle Dateien, die den Image-Inhalt beeinflussen — kein Skript nötig.
3. [tyriis/docker-image-tag-exists](https://github.com/tyriis/docker-image-tag-exists) prüft, ob `foo-image:sha-<hash>` bereits in GHCR existiert.
   - **Nein** → [docker/build-push-action](https://github.com/docker/build-push-action) baut und pusht mit den Tags aus [docker/metadata-action](https://github.com/docker/metadata-action):
     - `foo-image:sha-<hash>` — unveränderlicher, inhaltsadressierter Tag
     - `foo-image:snapshot` — beweglicher Alias auf den letzten Stand
   - **Ja** → Build wird übersprungen, [akhilerm/tag-push-action](https://github.com/akhilerm/tag-push-action) zeigt nur den `snapshot`-Tag registry-seitig auf das vorhandene Image.

   Bei Pull Requests wird nur gelintet und probegebaut, nicht gepusht.

### Release ([.github/workflows/release.yml](.github/workflows/release.yml))

Läuft bei Git-Tag-Pushes, z.B. via GitHub Release. Die CalVer-Validierung (`vYYYY.MM.PATCH`, z.B. `v2026.06.0`) passiert direkt über die Glob-Filter des Triggers — Tags, die dem Schema nicht entsprechen, starten den Workflow gar nicht erst.

1. Der Content-Hash des getaggten Commits wird per `hashFiles()` berechnet.
2. [tyriis/docker-image-tag-exists](https://github.com/tyriis/docker-image-tag-exists) prüft, ob `foo-image:sha-<hash>` in GHCR existiert. Wenn nicht, bricht der Workflow per [actions/github-script](https://github.com/actions/github-script) mit Fehler ab — der CI-Workflow muss für diesen Stand zuerst gelaufen sein.
3. Existiert es, retaggt [akhilerm/tag-push-action](https://github.com/akhilerm/tag-push-action) das Image **registry-seitig** (crane, kein Pull/Push der Layer) auf:
   - `foo-image:v2026.06.0` (der CalVer-Tag)
   - `foo-image:latest`

### Warum ein `sha-<hash>`-Tag und nicht nur `snapshot`?

`snapshot` ist ein beweglicher Tag und zeigt immer nur auf den zuletzt gebauten Stand. Um beim Release prüfen zu können, ob ein Image *zu genau diesem Quellstand* existiert, braucht jedes Image einen unveränderlichen, inhaltsadressierten Tag. `snapshot` wird als komfortabler Alias zusätzlich gepflegt.

## Release erstellen

```sh
git tag v2026.06.0
git push origin v2026.06.0
```

oder über die GitHub-UI ein Release mit Tag `v2026.06.0` anlegen.

## Lokal bauen

```sh
docker build -t foo-image:dev .
docker run --rm -p 8080:80 foo-image:dev
# http://localhost:8080
```

## Hinweise

- Die Workflows nutzen das eingebaute `GITHUB_TOKEN` (Permission `packages: write`) — keine zusätzlichen Secrets nötig.
- Einziger verbliebener Shell-Schritt (eine Zeile pro Workflow): das Kleinschreiben des Repository-Owners, da GHCR kleingeschriebene Namen verlangt und GitHub-Expressions keine lowercase-Funktion bieten.
- Beim allerersten Push legt GHCR das Package privat an; Sichtbarkeit ggf. in den Package-Einstellungen anpassen.
# docker-build-promote-demo
