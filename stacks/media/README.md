media
=====

Finds, downloads, renames and subtitles films and series into the library a
media server already reads, and lets the household ask for something without
touching any of that.

| Service | Does |
|---|---|
| [Prowlarr](https://prowlarr.com) | Holds the indexers once and hands them to Radarr and Sonarr |
| [Radarr](https://radarr.video) | Films: search, download, rename into the library |
| [Sonarr](https://sonarr.tv) | Series, the same way |
| [Bazarr](https://www.bazarr.media) | Subtitles next to the files Radarr and Sonarr placed |
| [Transmission](https://transmissionbt.com) | The torrent client |
| [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr) | Answers the bot checks some indexers put in front of their search |
| [Seerr](https://seerr.dev) | Requests from the household, signed in with their media server account |
| [Kometa](https://kometa.wiki) | Collections in Plex |
| curator | Stops looking for a better release once a film or episode has settled |

Seerr is what Overseerr and Jellyseerr became; both older projects point here.

## Two directories, on purpose not one

`MEDIA_DOWNLOADS_DIR` is where Transmission writes. `MEDIA_LIBRARY_DIR` holds
the libraries the media server reads, one subfolder per library. Radarr and
Sonarr take a finished download and copy or move it across under a clean name.

The usual advice is one directory for both so imports become instant hardlinks.
That only works on one filesystem, and it means the library disk carries
unfinished files. Keeping them apart costs a copy per download and buys a
library disk that only ever holds finished, renamed files. Keep the downloads
outside `BASE_DIR` as well, or the backup copies every half-finished download.

Radarr, Sonarr and Transmission mount the downloads at the same container path,
`/downloads`, and Radarr, Sonarr and Bazarr mount the library at `/media`. That
is what makes a path one of them reports valid in the others, without remote
path mappings.

**A copy to another disk writes straight to the final name.** A media server
that scans on file changes can pick up a file halfway through. Turn that
automatic scan off, scan on a schedule, and let Radarr and Sonarr notify the
server when an import finishes (Settings -> Connect).

## No web interface on a host port

Only the torrent peer port is published. The reverse proxy reaches every web
interface by container name over the shared network, with forward
authentication in front, so nobody on the network can walk around the login by
using a port. When authentik is down, turn forward authentication off for that
host in the proxy rather than publishing a port.

That also means a web interface is unreachable until its proxy host exists.

Suggested groups: everything is administration except Seerr, which is for the
household. With forward authentication in front, switch the built-in login of
Radarr, Sonarr, Prowlarr and Bazarr to disabled, or people sign in twice.

## First run

Order matters, because each step needs an API key from the one before.

1. **Transmission**: nothing to do; the image already writes to
   `/downloads/incomplete` and `/downloads/complete`
2. **Radarr and Sonarr**
   - Settings -> Media Management: root folder `/media/<library>`
   - Turn on renaming; it is off by default, and so is the clean name you are
     here for
   - Settings -> Download Clients: Transmission at `transmission:9091`
   - Existing libraries: Library -> Import. It expects one folder per film with
     the year in the folder name, and one folder per series
3. **Prowlarr**: add the indexers, then Settings -> Apps: Radarr at
   `http://radarr:7878` and Sonarr at `http://sonarr:8989` with their API keys
4. **Bazarr**: connect Radarr and Sonarr the same way, then choose languages
5. **Prowlarr, for indexers that check for bots**: Settings -> Indexer Proxies -> add
   FlareSolverr at `http://flaresolverr:8191`, give it a tag, and put that tag on the
   indexers that need it. An indexer without the tag ignores it
6. **Seerr**: sign in with the media server account, then add Radarr and Sonarr
7. **Kometa**: create `config.yml` below; the container waits until it exists

## When to stop upgrading

Radarr and Sonarr keep hunting for a better release forever: the only built-in
stop is a quality and score you may never reach. So a film that exists online
only as a 1080p rip is searched for every feed cycle, for years.

The `curator` service closes that off. Every hour it unmonitors anything that
has a file and has passed **both** windows: `CURATOR_DAYS_AFTER_ADD` since it
landed in the library, and `CURATOR_DAYS_AFTER_RELEASE` since it came out. The
later of the two wins, so something added the week it appears still gets its
full release window, while an old film is left alone shortly after it lands.

A file that already is what its profile is after is let go at once, without
waiting for either window. That is judged on what ffprobe found in the file,
because custom formats only ever see the release name, and names lie in both
directions: a "5.1 BluRay" can hold stereo Opus, and a plain "BluRay x264" can
hold perfectly good 5.1. What counts as done follows the profile's cutoff:

| Cutoff | Done when the file has |
|---|---|
| 1080p | at least 1080p, 5.1 or more, AC3 or EAC3 |
| 4K | 2160p in HDR or Dolby Vision, and EAC3 Atmos |
| 4K with remuxes | 2160p in HDR or Dolby Vision, and Atmos in any codec |

In every case the audio has to be in the original language when the file says
which language it is. AAC 5.1 plays fine but does not count as done, so a
Dolby release can still replace it within the window.

Unmonitoring stops searching, nothing else: the file stays, and you can still
ask for a better one yourself through Interactive Search.

It reads the API keys straight from the two config files, so it carries no
secrets, and it is a loop with a sleep rather than a cron daemon — one fewer
moving part to reason about.

### Moving versus copying

Radarr and Sonarr only *move* a download when "Remove Completed" is on for the
client and the torrent has stopped after its seeding limit. A torrent that is
still seeding is copied, and the copy on the server is deleted once the limit
is reached. Set the limit to what you want to give back; the import follows
from it.

## Kometa

It will not start without `${BASE_DIR}/kometa/config/config.yml`. Secrets stay
out of the file: Kometa replaces `<<name>>` with the variable `KOMETA_NAME`, and
the name after the prefix may not contain an underscore.

    libraries:
      Films:
        collection_files:
          - default: basic
          - default: imdb
    plex:
      url: http://host.docker.internal:32400
      token: <<plextoken>>
      timeout: 60
    tmdb:
      apikey: <<tmdbkey>>
      language: nl
      region: BE

Library names must match the media server exactly. Plain HTTP is deliberate
when the server runs on the same host: the traffic never leaves the machine,
and the server's certificate is issued for a `plex.direct` name, so HTTPS on
this address would only work with verification switched off.

Overlays rewrite the posters in Plex itself. Leave them out until you want
that; `remove_overlays: true` on a library undoes them.
