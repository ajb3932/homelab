# Homelab Backup & Recovery Plan

Nov 3, 2026 · @Alex Jarvis-Blanks

## How backups work

Every night at 03:00 homelab backs up everything to two independent borg repositories: the backup disk in homelab and the Raspberry Pi off-site. Any stack can be restored to any kept point with `sudo /lab/scripts/restore-stack.sh`.

| Destination | Where | Access |
| --- | --- | --- |
| local | backup disk `sdd`, mounted at `/lab/archive` only during the backup | `/lab/archive/borg/data` |
| remote | Pi `srvrbckp01`, off-site over Tailscale, `ssh://alex@10.0.0.14/backup` | homelab's key can only add backups (append-only), never delete |

### Nightly job (`/lab/scripts/borgmatic.sh`, root crontab)

1. Mount the backup disk.
2. Nextcloud AIO backup into `/lab/archive/nextcloud` (Nextcloud is offline about 2–3 minutes; AIO container updates run too).
3. Stop Portainer for about 2 seconds and copy `portainer.db` to `portainer.db.snapshot`.
4. borgmatic: dump databases, back up to the disk and the Pi, prune, compact, and run checks when due.
5. Unmount and spin down the disk.

ntfy (`ajb-monitor`) gets a message on success and an urgent one on any failure. Log: `/lab/scripts/borgmatic.log`.

### What is in each archive

| Stack | Files | Database dump |
| --- | --- | --- |
| Immich | `/lab/immich/upload` (photos, videos, thumbnails, Immich's own nightly DB dumps) | `immich` (Postgres 14) |
| Portainer | `/lab/appdata/portainer/data`: stacks, compose files, stack variables, `portainer.db.snapshot` | — |
| Home Assistant | `/lab/appdata/homeassistant`, including its 7 encrypted backups in `backups/` | — |
| Nextcloud | `/lab/archive/nextcloud`: the AIO borg repo holding data, config and database | inside AIO repo |
| Authelia, BookStack, Grimmory, Reitti | their `/lab/appdata` folders | `authelia`, `bookstack`, `grimmory`, `reittidb` |
| Everything else | all of `/lab/appdata` and `/lab/data` | — |

Not backed up: `/pool/media` (replaceable media), `/lab/machines`, `/lab/scripts`, `/lab/projects`.

### Retention

One archive per night is kept for 14 days, then 1 per week for 8 weeks, 1 per month for 12 months and 1 per year for 3 years. Older archives from before November 2026 are kept under the same rules; the oldest is September 2025.

### Checks

Sundays: repository check weekly, archive check every 2 weeks, full test extract monthly. All five databases were test-restored into throwaway copies on 3 November 2026 and matched the live data.

## Keys and passwords

Keep a copy of each of these off homelab (Vaultwarden runs on homelab, so also keep a printed or offline copy). Without them the backups cannot be opened.

| Secret | Needed for | Where to find it |
| --- | --- | --- |
| Borg passphrase | both borg repos | `BORG_PASSPHRASE` in the borgmatic stack (Portainer) |
| Borg key, local repo | backup disk repo | export (below) |
| Borg key, remote repo | Pi repo | export (below) |
| Nextcloud AIO backup password | Nextcloud restore | AIO interface, https://homelab:8080 → Backup and restore |
| Home Assistant backup encryption key | HA `.tar` backups | HA → Settings → System → Backups → ⋮ → Encryption key |
| Pi login | rebuild from the Pi | your desktop's SSH key, or the Pi's own console |

The borg keys are encrypted with the passphrase, so you need both. To export them (the disk must be mounted for the borgmatic container to start):

```
sudo mount /lab/archive && docker start borgmatic
docker exec borgmatic borg key export --paper /mnt/borg-repository
docker exec borgmatic borg key export --paper ssh://alex@10.0.0.14/backup
docker stop borgmatic && sudo umount /lab/archive
```

To restore a key into a new setup: `borg key import --paper <repo>` and type it back in.

## Restore a stack with restore-stack.sh

While homelab is running, this is the normal way to restore anything:

```
sudo /lab/scripts/restore-stack.sh            # restore
sudo /lab/scripts/restore-stack.sh --dry-run  # show the plan and verify the archive, change nothing
```

1. Choose the stack.
2. Choose **local** (backup disk, fast) or **remote** (Pi, use if the disk has failed).
3. Choose the archive, newest first.
4. Read the plan and type `yes`.

The script then mounts the disk, stops the stack's containers, restores its files, restores its database into its database container, starts everything, and unmounts the disk. A nightly backup cannot start while a restore is running.

| Stack | Files restored | Database restored |
| --- | --- | --- |
| immich | `/lab/immich/upload`, merged: missing or changed files come back, newer files stay | immich |
| homeassistant | `/lab/appdata/homeassistant` | — |
| portainer | `/lab/appdata/portainer/data`, then `portainer.db.snapshot` is put in place | — |
| authelia | `/lab/appdata/authelia/app` and `secrets` | authelia |
| bookstack | `/lab/appdata/bookstack/config` | bookstack |
| grimmory | `/lab/appdata/media/grimmory/data` | grimmory |
| reitti | `/lab/appdata/reitti/reitti-data` | reittidb |
| nextcloud | mounts the disk and waits while you restore in the AIO interface; can first fetch the AIO repo from the Pi | AIO |
| other | any folder in `/lab/appdata`; containers using it are found and stopped automatically | raw files only |

Replaced folders are kept beside the original as `<folder>.pre-restore-<date-time>`. The script prints the `rm -rf` command to delete them once you're happy. If a step fails, the containers are left stopped and the previous folder is untouched, so you can move it back.

## Manual restore, stack by stack

Use these steps when the script can't run, for example on a rebuilt server. Start with the setup, then follow the stack's steps.

### Setup

```
sudo mount /lab/archive && docker start borgmatic
docker exec borgmatic borgmatic list --repository local --match-archives '*'   # pick <archive>
```

Use `--repository remote` instead of `local` to restore from the Pi. Extract with `--destination /` and files land in their live place:

```
docker exec borgmatic borgmatic extract --repository local --archive <archive> --path <archive path> --destination / --progress
```

| Archive path | Live path |
| --- | --- |
| `src/photos` | `/lab/immich` |
| `src/appdata` | `/lab/appdata` |
| `src/data` | `/lab/data` |
| `src/nextcloud` | `/lab/archive/nextcloud` |

When finished: `docker stop borgmatic && sudo umount /lab/archive`.

### Immich

1. `docker stop immich immich_ml` (leave `immich_db` running).
2. Extract `src/photos/upload`.
3. Restore the database: `docker exec borgmatic borgmatic restore --repository local --archive <archive> --database immich`
4. For archives made before 3 November 2026, step 3 fails. Use Immich's own dump from step 2 instead, picking the newest `immich-db-backup-*.sql.gz` in `/lab/immich/upload/backups` that is older than the archive:
   - `gunzip --stdout <dump> | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" | docker exec -i immich_db psql --dbname=immich --username=postgres`
5. `docker start immich immich_ml`. Missing thumbnails: Administration → Jobs → Generate thumbnails → Missing.

### Portainer

Restore Portainer first on a rebuilt server: it brings back every stack's compose file and variables.

1. `docker stop portainer`, then `sudo mv /lab/appdata/portainer/data /lab/appdata/portainer/data.old`
2. Extract `src/appdata/portainer/data`.
3. `sudo cp -a /lab/appdata/portainer/data/portainer.db.snapshot /lab/appdata/portainer/data/portainer.db`
4. `docker start portainer`. On a new server, start it from the compose file in `/lab/appdata/portainer`.

### Home Assistant

- **From an HA backup (preferred):** extract `src/appdata/homeassistant/backups`. On a fresh install choose **Restore from backup** during onboarding, upload the `.tar` and enter the encryption key. On a running install use Settings → System → Backups → choose one → Restore.
- **Whole folder:** `docker stop homeassistant`, move `/lab/appdata/homeassistant` aside, extract `src/appdata/homeassistant`, `docker start homeassistant`.

### Nextcloud

1. If the backup disk is lost, first fetch the AIO repo from the Pi: extract `src/nextcloud` from `--repository remote`.
2. Open https://homelab:8080. A running AIO: Backup and restore → choose a backup → Restore. A fresh AIO: **Restore former AIO instance**, location `/lab/archive/nextcloud`, then the AIO backup password.
3. Keep the disk mounted until AIO says the restore is finished. Nextcloud's data folder is `/lab/nextcloud`.

### Authelia, BookStack, Grimmory, Reitti

1. Stop the app container (`authelia`, `bookstack`, `grimmory` or `reitti`) and leave its database container running.
2. Move its folder aside and extract it (paths in the table in the previous section).
3. `docker exec borgmatic borgmatic restore --repository local --archive <archive> --database <authelia|bookstack|grimmory|reittidb>`
4. Start the app container.

### Any other stack

Stop its containers, move `/lab/appdata/<folder>` aside, extract `src/appdata/<folder>`, and start the containers. Any database files in that folder come back as they were at backup time.

## Full server rebuild

If homelab is lost, rebuild it with plain borg first; the borgmatic container comes back with the restored files. Bringing back about 1 TB takes 1–2 hours from the backup disk, or 8 hours or more from the Pi.

1. Install Ubuntu, Docker and `borgbackup` (1.2 or newer). Recreate `/lab`, the NVMe mount for `/lab/immich`, and the backup disk's `/etc/fstab` line with `noauto,nofail`.
2. Choose the source and set the passphrase:
   - Backup disk: `sudo mount /lab/archive`, then `REPO=/lab/archive/borg/data`
   - Pi: `REPO=ssh://alex@10.0.0.14/backup` (log in with your desktop's SSH key)
   - `export BORG_PASSPHRASE='<passphrase>'`; if the repo key is damaged, `borg key import --paper $REPO` first.
3. List archives: `borg list $REPO`, and choose `<archive>`.
4. Restore the files:
   - `cd /lab/appdata && sudo -E borg extract --strip-components 2 $REPO::<archive> src/appdata`
   - `cd /lab/data && sudo -E borg extract --strip-components 2 $REPO::<archive> src/data`
   - `cd /lab/immich && sudo -E borg extract --strip-components 2 $REPO::<archive> src/photos`
   - Backup disk lost: `cd /lab/archive/nextcloud && sudo -E borg extract --strip-components 2 $REPO::<archive> src/nextcloud`
5. Start Portainer from `/lab/appdata/portainer` and redeploy the stacks. They come back with their variables.
6. Restore the databases with borgmatic, now that its stack is back (see Manual restore): immich, authelia, bookstack, grimmory, reittidb.
7. Restore Nextcloud in the AIO interface (see Manual restore).
8. Recreate the scripts in `/lab/scripts` and the root cron line `0 3 * * * /lab/scripts/borgmatic.sh >/dev/null 2>&1`. Then lock the empty mountpoint: `sudo umount /lab/archive && sudo chattr +i /lab/archive`.

The borgmatic config, its Pi key and Home Assistant's backups all come back with `/lab/appdata` in step 4.

## Routine tasks

- **Daily:** check for the ntfy success message. No message at all means the job didn't run; check `/lab/scripts/borgmatic.log`.
- **Look inside the backup disk:** `sudo mount /lab/archive`, and `sudo umount /lab/archive` when done. Unmounted, `/lab/archive` is an empty, locked folder; this is normal.
- **Run a backup now:** `sudo /lab/scripts/borgmatic.sh`
- **Free space on the Pi (twice a year):** homelab can't delete on the Pi, so pruned archives still take space there. Log into the Pi from your desktop, check `borg list /backup` looks normal, then run `borg compact /backup`.

| How often | Restore test |
| --- | --- |
| Every 3 months | `sudo /lab/scripts/restore-stack.sh --dry-run` for Immich and Home Assistant |
| Every year | Restore one stack from the **Pi** onto a scratch copy and open it |

## Still to do

- [ ] Store the borg passphrase, both exported borg keys, the AIO backup password and the HA encryption key off homelab.
- [ ] Add `/lab/scripts:/src/lab-extra/scripts` and `/lab/projects:/src/lab-extra/projects` to the borgmatic compose (git repo), redeploy, then uncomment `/src/lab-extra` in `config.yaml`. Until then the backup and restore scripts are not themselves backed up.
- [ ] Move `BORG_PASSPHRASE` out of the committed compose file into a Portainer stack variable.
- [ ] Add a healthchecks.io ping (`healthchecks:` in `config.yaml`) so a missed night raises an alert.
- [ ] Remove the `alex@homelab` key from the Pi's `~/.ssh/authorized_keys` if you don't need to log in from homelab. It has full access, unlike borgmatic's append-only key.
