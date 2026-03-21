# Data Backup
Some users prefer backing up the LWS database in case it gets trashed due to a
bug, etc. Or in case of hard-drive corruption you can quickly migrate
to a new machine if you have a backup of the `monerod` blockchain and LWS
database.

!!! info

    All of the data stored by `monerod` and LWS can be recovered from
    blockchain data (but this may change LWS side). And all LWS wallets powered
    by `lwsf` (`Skylight` and `lwcli`) automatically detect when there's been a
    change server side, and will request new account creation with import from
    original height. So off-site cloud backups are less critical, but many
    users report that local snapshots were useful for recovering from LWS bugs
    (which are becoming less frequent with higher deployment counts).

Backup of a running/live LWS and `monerod` blockchain can be done with the
`mdb_copy` command. The command creates a read snapshot of the DB, which is
then bit copied into a new file. Users running on "bare metal"  should install
`lmdb` from their package manager, and then run:

```bash
mdb_copy -c /home/monero-lws/.bitmonero/light_wallet_server /home/monero-lws/
```

Which creates a file safe to copy/move/delete called
`/monero/monero-lws/data.mdb`. Docker users have bit more work to do. Do not
touch the DB in `light_wallet_server`, while LWS is running, without lmdb
tooling - you could copy a partially updated file which will be in a bad state.

!!! warning

    You must match "endianess" with your backup. In practical terms, this means
    you cannot import to/from PowerPC, but can import to/from Intel/arm64/Risc-V
    interchangeably.

## Creating lmdb Container
There is no decent docker container image for lmdb, only a very stripped down
version provided by [StageX](https://stagex.tools). This is actually a benefit -
StageX is a reproducible build environment where the images are verified by
multiple parties.

You need to create a file called `Dockerfile` with the following contents:

```
FROM scratch
COPY --from=stagex/core-musl . /
COPY --from=stagex/core-lmdb . /
```

and then run the command

```bash
docker build -t lmdb .
```

which creates a local container image called `lmdb` with lmdb executables.

## Containerized Backup
Now that you have a container image for lmdb, you can issue the snapshot
command from above but with correct volumes specified:

```bash
docker run -it --rm -v lws-hidden-service_lws:/home/monero-lws lmdb mdb_copy -c /home/monero-lws/.bitmonero/light_wallet_server /home/monero-lws/
docker cp lws-hidden-service-lws-1:/home/monero-lws/data.mdb lws_snapshot.mdb
docker run -it --rm -v lws-hidden-service_lws:/home/monero-lws busybox rm /home/monero-lws/data.mdb
```

After running these commands, you should have a compacted LWS db
(`lws_snapshot.mdb`) in your local directory. You should backup this file using
whatever technique you desire, and considering doing so in snapshots.

!!! tip

    Create a `lws_backup.sh` file, put `#!/bin/sh` on the first line, then copy
    the above 3 commands into the file. Lookup how to run cronjobs on your
    Linux distro for nightly backups, and call `lws_backup.sh` from the job.

!!! tip

    A utility called `rsnapshot` is useful for storing multiple versions of
    files in a local directory. Off-site backups should likely use an encrypted
    snapshot system: `borg-backup`, `kopia`, or `restic`.


### monerod Backup
The monerod doesn't really need a backup, but this may be desireable if you
want faster recovery if your system fails. If you want to proceed with the
backup, make sure your total drive size is 2 TB as this will require lots of
temporary storage. The commands for backup are:

```bash
docker run -it --rm -v lws-hidden-service_monerod:/home/monero lmdb mdb_copy -c /home/monero/.bitmonero/lmdb /home/monero
docker cp lws-hidden-service-monerod-1:/home/monero/data.mdb monerod_snapshot.mdb
docker run -it --rm -v lws-hidden-service_monerod:/home/monero --entrypoint rm ghcr.io/vtnerd/monero-lws /home/monero/data.mdb
```

These commands need to be run less often, unless you need super fast recovery.
It is  not recommended that you `rsnapshot` or off-site backup as this will be
quite large. You'll probably just want a second copy on a second local
machine.
