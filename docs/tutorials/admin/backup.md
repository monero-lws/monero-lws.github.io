# Data Backup
Backup of a running/live LWS can be done with the `mdb_copy` command. Users
that did not follow our docker tutorials should install LMDB from their package
manager, and then run:

```bash
mdb_copy -c /home/monero-lws/.bitmonero/light_wallet_server /home/monero-lws/
```

Which creates a file safe to copy/move/delete called
`/monero/monero-lws/data.mdb`. Docker users have bit more work to do.

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
docker run -it --rm -v lws-hidden-service_lws:/home/monero-lws --entrypoint rm ghcr.io/vtnerd/monero-lws /home/monero-lws/data.mdb
```

After running these commands, you should have a compacted LWS db
(`lws_snapshot.mdb`) in your local directory. You should backup this file using
whatever technique you desire, and considering doing so in snapshots.

### monerod Backup
The monerod doesn't really need a backup, but this may be desireable if you
want faster recovery if your system fails. If you want to proceed with the
backup, make sure your total drive size is 1 TB as this will require lots of
temporary storage. The commands for backup are:

```bash
docker run -it --rm -v lws-hidden-service_monerod:/home/monero lmdb mdb_copy -c /home/monero/.bitmonero/lmdb /home/monero
docker cp lws-hidden-service-monerod-1:/home/monero/data.mdb monerod_snapshot.mdb
docker run -it --rm -v lws-hidden-service_monerod:/home/monero --entrypoint rm ghcr.io/vtnerd/monero-lws /home/monero/data.mdb
```

These commands need to be run less often, unless you need super fast recovery.
