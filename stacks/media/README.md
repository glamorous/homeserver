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
