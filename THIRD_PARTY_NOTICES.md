# Third-party notices

The Bridge Music Server container image (`public.ecr.aws/r5n8t3v1/bridge-server`)
bundles the Bridge server, which is proprietary (see `LICENSE`), with
the independent third-party programs listed below. Each one keeps its
own licence. The Bridge server talks to Navidrome only over HTTP, as a
separate process; the two are aggregated in one image, not combined into
one program.

In the image, this file, `LICENSE` and the `licenses/` directory are
installed at `/usr/share/doc/bridge/`. A running server also serves this
file at `GET /api/licenses` (Settings → About → Open-source licences).

---

## Navidrome 0.64.0

- **Licence:** GNU General Public License v3.0. The full text is in
  `licenses/GPL-3.0.txt`.
- **Copyright:** Deluan Quintão and the Navidrome contributors.
- **What ships:** the unmodified `navidrome` binary from the official
  `deluan/navidrome:0.64.0` image, installed at `/app/navidrome`.
- **Corresponding source:** https://github.com/navidrome/navidrome/tree/v0.64.0.
  A copy of the source archive for this exact version is also attached
  to the matching release of https://github.com/k-pet/bridge-server-selfhost.

### Written offer for source code

For at least three years after we last distribute a given image version,
Bridge Music will provide anyone who asks with a complete copy of the
Corresponding Source of the GPL-licensed software in that image, on a
medium customarily used for software interchange, for no more than our
cost of physically performing the distribution (a download link is free).

Request source: open an issue at https://github.com/k-pet/bridge-server-selfhost/issues

Say which image tag or digest you are running.

---

## s6-overlay 3.2.3.2

- **Licence:** ISC. The full text is in `licenses/ISC-s6-overlay.txt`.
- **Copyright:** (c) 2021-2026 Laurent Bercot, John Regan.
- **Source:** https://github.com/just-containers/s6-overlay/tree/v3.2.3.2
- **What ships:** the release tarballs, unpacked under `/` (process
  supervisor, `/init`). These include the skarnet.org utilities (skalibs,
  execline, s6, s6-rc and others), which are also ISC-licensed.

---

## Alpine Linux base and packages

The runtime is `alpine:3.24` plus these packages from the Alpine
repositories, with their shared-library dependencies:

| Package | Licence (from Alpine package metadata) |
|---|---|
| ffmpeg | GPL-2.0-or-later AND LGPL-2.1-or-later |
| wget | GPL-3.0-or-later WITH OpenSSL-Exception |
| ca-certificates | MPL-2.0 AND MIT |
| tzdata | Public-Domain |

The base system and the codec libraries FFmpeg links against carry
further licences (GPL-2.0, LGPL-2.1, MIT, BSD and others). The GPL-2.0,
LGPL-2.1 and GPL-3.0 texts are in `licenses/`.

Alpine records each package's licence in its package metadata rather
than installing licence files. To list every package in the image with
its version and licence:

```sh
docker run --rm --entrypoint apk public.ecr.aws/r5n8t3v1/bridge-server:latest list --installed

# One package's licence, description and files
docker run --rm --entrypoint sh public.ecr.aws/r5n8t3v1/bridge-server:latest \
  -c 'apk info --license ffmpeg && apk info -d ffmpeg && apk info -L ffmpeg'
```

The source of every Alpine package, including its full licence files, is
in Alpine's `aports` repository
(https://gitlab.alpinelinux.org/alpine/aports, branch `3.24-stable`)
and its source mirrors. The written offer above covers these packages
too.

---

## Go modules linked into the Bridge server

Listed from `go version -m /app/bridge-server`. Their licence texts,
including the Go standard library's, are in `licenses/go-modules.txt`.

| Module | Version | Licence |
|---|---|---|
| Go standard library and runtime | 1.27 | BSD-3-Clause |
| github.com/bogem/id3v2/v2 | v2.1.4 | MIT |
| github.com/dhowden/tag | v0.0.0-20240417053706-3d75831295e8 | BSD-2-Clause |
| github.com/dustin/go-humanize | v1.0.1 | MIT |
| github.com/go-flac/flacvorbis/v2 | v2.0.2 | Apache-2.0 |
| github.com/go-flac/go-flac/v2 | v2.0.4 | Apache-2.0 |
| github.com/google/uuid | v1.6.0 | BSD-3-Clause |
| github.com/llehouerou/go-mp4tag | v0.1.0 | MIT |
| github.com/mattn/go-isatty | v0.0.24 | MIT |
| github.com/ncruces/go-strftime | v1.0.0 | MIT |
| github.com/remyoudompheng/bigfft | v0.0.0-20230129092748-24d4a6f8daec | BSD-3-Clause |
| golang.org/x/sys | v0.48.0 | BSD-3-Clause |
| golang.org/x/text | v0.42.0 | BSD-3-Clause |
| modernc.org/libc | v1.75.7 | BSD-3-Clause (and bundled third-party notices) |
| modernc.org/mathutil | v1.7.1 | BSD-3-Clause |
| modernc.org/memory | v1.12.1 | BSD-3-Clause |
| modernc.org/sqlite | v1.58.0 | BSD-3-Clause; SQLite itself is public domain |

## npm packages bundled into the web UI

The production dependencies in `frontend/package.json`, compiled into
the web UI served by the Bridge server. Their licence texts are in
`licenses/npm-packages.txt`.

| Package | Version | Licence |
|---|---|---|
| @hcaptcha/react-hcaptcha | 2.2.0 | MIT |
| @supabase/supabase-js | 2.116.0 | MIT |
| @tanstack/react-query | 5.102.8 | MIT |
| @tanstack/react-virtual | 3.14.11 | MIT |
| react | 19.3.0 | MIT |
| react-dom | 19.3.0 | MIT |
| react-router-dom | 7.18.3 | MIT |

These packages pull in further transitive dependencies. Run
`npm ls --omit=dev --all` in `frontend/` for the full tree.

---

## Data sources

The server does not bundle these, but it fetches from them at run time
when you identify music.

- **MusicBrainz** (https://musicbrainz.org). Recording, release and
  artist data comes from the MusicBrainz database. Its core data is
  released under CC0 (public domain dedication); see
  https://musicbrainz.org/doc/About/Data_License. Only core data is
  used. Every request identifies the application in its User-Agent
  (`bridge-server/<version> ( https://bridgemusic.app )`), as the
  MusicBrainz API rules ask, and requests are rate-limited to at most
  one per second.
- **Cover Art Archive** (https://coverartarchive.org), a joint project of
  the MetaBrainz Foundation and the Internet Archive, supplies front
  cover images. Each image stays under the rights of its uploader or
  copyright holder. The server stores a fetched cover only next to the
  music it belongs to, in your own library.
- **Wikidata, Wikimedia Commons and Wikipedia** (https://www.wikidata.org,
  https://commons.wikimedia.org, https://en.wikipedia.org), projects of
  the Wikimedia Foundation, supply artist photographs. Starting from an
  artist's MusicBrainz id, the server reads the artist's Wikidata item
  (its "image" claim) or, failing that, the lead image of the artist's
  English Wikipedia article, and accepts only files hosted on Wikimedia
  Commons, whose licence is machine-readable. For each photograph it
  records the file page, author and licence (CC0, public domain,
  CC BY or CC BY-SA in nearly every case) and shows them as a credit
  line on the artist page to everyone who can see the photo, as those
  licences require. The image is copied byte for byte at the size
  Commons renders it — never cropped, re-encoded or otherwise adapted —
  so no derivative work is created. Wikidata's structured data is CC0;
  Wikipedia text is CC BY-SA but none of it is stored. Requests carry the
  User-Agent Wikimedia's policy asks for
  (`bridge-server/<version> ( https://bridgemusic.app )`) and are paced
  to about one per second. The texts of the two attribution licences are
  in `licenses/CC-BY-4.0.txt` and `licenses/CC-BY-SA-4.0.txt`; each
  photo's own licence, which may be an earlier version, is linked from
  its credit line.
