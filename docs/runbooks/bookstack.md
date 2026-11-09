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

## Change the base URL

Use this one-time procedure when deploying the migrated library from
`https://bookstack.localhost` to `https://docker.bookstack`. Existing absolute
image URLs and content links need a database update as well as `APP_URL`.
The [LinuxServer image instructions](https://docs.linuxserver.io/images/docker-bookstack/#changing-app_url)
provide the `bookstack:update-url` command below. This is an operator procedure,
not a record of a completed live cutover.

1. On CT 105, confirm the live `wiki` project and directory using the container
   labels described in [common precautions](common.md#establish-the-target).
   The recorded source is `/root/services/optiplex/wiki/compose.yml`, outside this
   checkout. Confirm both selected migrated volumes listed above, current image
   IDs, and the actual old URL before proceeding. If a prior cutover already
   migrated the content, verify it instead of repeating the mutation.
2. Arrange downtime and block client access to BookStack through both the proxy
   and direct port 6875, keeping it blocked through verification. Stop BookStack,
   then MariaDB. Follow the [consistent cold backup procedure](common.md#before-a-restart-upgrade-or-restore)
   for both `/config` volumes, the original `APP_KEY`, credentials, and live
   deployment configuration. Verify archive checksums and record the backup
   location privately before changing anything; the October backup does not
   cover subsequent edits.
3. Set `APP_URL: https://docker.bookstack` in the live Compose source, preserving
   the image references, encryption key, credentials, and volume selections.
   Start `bookstack-db` and wait for database readiness, then apply the changed
   application environment without pulling an image:

   ```sh
   docker compose up -d --no-deps --pull never --force-recreate bookstack
   ```

   Confirm the recreated application's mounts and readiness before continuing.
   A first-run screen means stop and investigate the selected data.
4. With the application and database running and client access still blocked,
   run from the confirmed live stack directory:

   ```sh
   docker compose exec bookstack php /app/www/artisan bookstack:update-url 'https://bookstack.localhost' 'https://docker.bookstack'
   ```

   Require successful completion; preserve the result privately. If the actual
   old URL differs, use that exact value. Do not substitute a different hostname
   merely to make the command succeed.
5. From a LAN client using Pi-hole DNS and the trusted Caddy CA, sign in at
   `https://docker.bookstack`. Open existing pages, embedded uploaded images,
   attachments, and internal content links; inspect their destinations for the
   old hostname. Check permissions and expected library contents. Verify again
   after restarting BookStack, then reopen client access and record the cutover.

If the command or acceptance checks fail, keep access blocked, preserve the failed
state, and restore the saved live configuration and consistent pre-change volume
pair using the recovery procedure below. Changing `APP_URL` back alone does not
undo database content changes. Avoid a blanket reverse URL replacement, which
could also change content that already used the new hostname before this cutover.

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
