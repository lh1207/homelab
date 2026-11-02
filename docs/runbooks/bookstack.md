# BookStack

## Scope and recorded state

Read [common precautions](common.md). [Reference Compose](../../reference/wiki/compose.yml)
describes stack `wiki`, services `bookstack` and `bookstack-db`, in CT 105. The live
source is recorded at `/root/services/optiplex/wiki/compose.yml`. HTTP is on 6875;
the recorded base URL is `https://bookstack.localhost`. CT 106 is stopped in Archive.

| Mount | Selected migrated volume |
| --- | --- |
| BookStack `/config` | `wiki_bookstack_migrated_20261018` |
| MariaDB `/config` | `wiki_bookstack_db_migrated_20261018` |

Overrides are `BOOKSTACK_DATA_VOLUME` and `BOOKSTACK_DB_VOLUME`. The migration
record reports BookStack v26.03.4, original-account sign-in, seven books, three
shelves, ten active pages plus one deleted page, and HTTPS after restart. These
are historical baseline counts; compare with the chosen backup's actual state.

## Diagnose

In the confirmed live stack directory:

```sh
docker compose ps bookstack bookstack-db
docker compose logs --tail=100 bookstack-db bookstack
```

Check CT 105 disk space and mounts if database startup fails. Test the direct
`http://DOCKER_LXC_IP:6875` backend from an authorized client, then the HTTPS route.
A redirect to the configured URL can be expected. A proxy 502 points to upstream
reachability; a database connection error requires checking the DB container,
shared `wiki` network, and private configuration.

Confirm `DB_HOST=bookstack-db`, port 3306, database `bookstack`, and the matching
application/database credentials privately. BookStack uses `DB_USERNAME` and
`DB_PASSWORD`, not `DB_USER`/`DB_PASS`. Changing MariaDB initialization environment
variables does not reset users in an existing database. Do not reset database
passwords or initialize an empty database as a diagnostic shortcut.

For redirects or mixed content, verify deployed `APP_URL` matches the route and
the proxy/client CA trust. Do not generate a new `APP_KEY` to fix authentication
or decryption errors; the original encryption key is part of the recovery set.

## Restart or recover

For an application-only transient failure with a healthy DB:

```sh
docker compose restart bookstack
```

For a planned full restart, stop the application before the DB, start the DB,
wait for successful readiness in its logs, then start the application:

```sh
docker compose stop bookstack
docker compose stop bookstack-db
docker compose start bookstack-db
# Wait for database readiness before the next command.
docker compose start bookstack
```

Compose `depends_on` alone does not establish database readiness. Back up both
stopped `/config` volumes as a consistent pair, with the original `APP_KEY`,
custom environment settings, uploaded files, DB credentials, and deployment
sources. A database-only backup omits uploads; Git omits private keys and data.

Follow the common isolated restore procedure, using fresh volumes and both volume
overrides. Start a compatible MariaDB version before the matching BookStack image.
The references use `latest`; record exact live images and avoid accidental upgrade
or automatic schema migration during recovery. Do not regenerate the key. Verify
the restored application and DB mounts before permitting writes.

## Acceptance and rollback

Sign in with an original account, inspect expected shelves/books/pages and access
permissions, open existing uploaded images and attachments, and verify HTTPS links.
Create, edit, and remove a designated test page if normal writes are in scope.
Check logs and repeat sign-in and a content read after restart. A login screen or
HTTP 200 alone does not prove the library and encrypted data survived.

If acceptance fails, stop the app before the DB, preserve failed data, then restore
the saved configuration and compatible application/DB backup pair. Do not run an
older image against an upgraded schema. The untouched May 4 volumes are not the
migrated library; their saved root password previously failed. CT 106 is also a
dated fallback: stop the Docker pair before switching and account for newer data.
