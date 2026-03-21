# Hidden Service
The easiest setup for novice self-hosters is Tor+monerod+LWS running in a
[single "docker compose" setup](#docker-compose) on a spare machine at home.
The Tor daemon provides end-to-end authenticated encryption from light-wallets
to your home machine, without complex firewall rules, TLS certificates, or
cloud computing costs. [Skylight wallet](https://skylight.magicgrants.com) has
built-in Tor support on desktop and mobile, and
[lwcli](https://github.com/cifro-codes/lwcli) has socks proxy for experienced
desktop users that can run their own Tor client.

!!! info

    Podman can be used as a replacement for docker - all of the provided files
    and guidance should work.

## Expectations
### ... for Users
Your typical user will have to install `Skylight` from Android/iOS app store,
select the option to use builtin Tor, and then copy+paste a long "\*.onion"
address into the server option. The onion address is automatically stored
encrypted on all LWS wallets so it only has to be entered once.

More advanced users on `lwcli` will have to run their own Tor client, and give
`lwcli` the socks proxy address. `lwcli` automatically stores the proxy and
onion address encrypted in the metadata file.

!!! warning

    One small "cheat" in the above explanation. You have to either allow _all_
    users that request an account on your LWS service, or manually approve
    addresses via the [command-line](#list-requests) or
    [admin API](https://docs.monerolws.com/api/admin). Tutorial+code for sending
    accept/reject links via Pushover is in the works. Accepting all requests is
    acceptable for now - your hidden address has to be leaked as a LWS server
    for it to be problematic.

### ... for Self-Hoster (you)
Install Linux+docker on a spare machine with at least 512GiB of SSD/nvme disk
space, and 8 GiB of RAM. Configure Tor and sync `monerod` as instructed by this
guide. Get a copy of the ["docker-compose"](#docker-compose) file provided in this
guide. Instruct docker to run the docker-compose file. Ensure docker starts
at boot.

Maintenance is performed by running two commands: (1) a "pull" command to fetch
latest versions of each container/project, and (2) a "restart" command that
will forcibly restart all 3 projects together.

Data backup isn't strictly necessary (right now!), as monerod and LWS only
store data that can be recovered from the Monero blockchain. That said, it
still may prudent to copy the DB locally in case of failures or quicker
recovery. A separate [tutorial](../admin/backup.md) is provided for this
- the process involves running another Docker command that safely copies the DB
of monerod and/or LWS, _even while they are running_. The result will be a file
for monerod and a file for LWS.

## Security
The provided docker compose file runs Tor, monerod, and LWS in separate
containers, such that they cannot read/write to the files of the other project.
In fact, container isolation prevents them from accessing any files on the
"host" filesystem, where "host" refers to the OS that is running Docker. This
is beneficial to your users because LWS will know the address+viewkeys of every
wallet; docker/container isolation makes it significantly more difficult for an
issue in Tor or monerod to affect the privacy of your users.

## Setup
The first step is setting up Linux and docker. These steps are out of scope for
this guide; internet search engines are your friend here.

!!! tip

    Limit accounts that are in the OS group `docker`. Any account in this group
    can read your Tor private key, so you'll want to restrict access.

### Create a directory to store config files
The recommended directory name is `lws-hidden-service`, but create one of your
choosing and `cd` into via bash.

### Create Tor Onion Address
Before any of the services can start, a persistent onion address must be
created. This address can be changed later to silently "revoke" some/all users,
but this can also be achieved with an admin DB command.

The first stage of the setup:

```bash
docker container create --name lws_tor_config docker.io/osminogin/tor-simple
docker cp lws_tor_config:/etc/tor/torrc ./
docker container rm lws_tor_config
```

Will create a file `torrc` in the current working directory which is the most
recent default configuration for Tor. You must edit this file and add the
following to the section labeled "for location-hidden services":

```
HiddenServiceDir /var/lib/tor/lws_hidden_service
HiddenServicePort 80 lws:8080
```

which instructs Tor to create/use a randmized hidden service address and
forward incoming connections to the LWS. 

### Initializing monerod
LWS will complain if `monerod` is not synced past the most recent checkpoint.
This unfortunately means we must run monerod separately initially, until it
synchronizes with the most recent block on the network. The command to start
this process is:

```bash
docker run -d --rm --name lws_monerod_sync -v lws-hidden-service_monerod:/home/monero ghcr.io/sethforprivacy/simple-monerod
```

The length of time required to sync will depend on your CPU and hard-drive
capabilities. Expect at least 24 hours. You can view the progress at anytime
by running:

```bash
docker logs lws_monerod_sync | tail
```

which will dump the last status updates. Remove `| tail` if you canot see the
synchronization progress. After syncing, simply run:

```bash
docker stop lws_monerod_sync
```

which will cleanup the container (due to original `--rm` flag) but save the
blockchain data in the volume.

### Docker Compose
Finally, after configuring Tor and syncing the blockchain, you simply need to
run a docker compose file which starts all of the processes and keeps them
running. Copy the below to a "compose.yml" file:

```yaml
name: lws-hidden-service
services:
  tor:
    image: docker.io/osminogin/tor-simple
    command: tor
    restart: always
    volumes:
      - ./torrc:/etc/tor/torrc:ro
      - tor_secrets:/var/lib/tor
  monerod:
    image: ghcr.io/sethforprivacy/simple-monerod
    command: --rpc-restricted-bind-ip=0.0.0.0 --rpc-restricted-bind-port 18089 --no-igd --zmq-pub tcp://0.0.0.0:18083 --zmq-rpc-bind-ip 0.0.0.0 --confirm-external-bind --enable-dns-blacklist --ban-list /home/monero/ban_list.txt
    restart: always
    ports:
      - 18080:18080
      - 18089:18089
    expose:
      - 18082
      - 18083
    volumes:
      - monerod:/home/monero
  lws:
    image: ghcr.io/vtnerd/monero-lws
    command: --rest-server http://0.0.0.0:8080 --confirm-external-bind --daemon tcp://monerod:18082 --sub tcp://monerod:18083 --auto-accept-creation
    restart: always
    ports:
      - 8080:8080
    depends_on:
      - monerod
    volumes:
      - lws:/home/monero-lws
volumes:
  tor_secrets:
  monerod:
  lws:
```

then run the command (always from this directory that your created for
`lws-hidden-services`):

```bash
docker compose up -d
```

Which should bring up Tor, monerod, and LWS into a system that is used
together. Keep this directory available, as you will need it after you restart
from updating the container images.

### Accessing my LWS
Accessing your LWS instance can be done either by connecting to the local IP
address on port 8080 OR by the new onion address. Finding your onion address
is accomplished by running this command:

```bash
docker run --rm -v lws-hidden-service_tor_secrets:/var/lib/tor docker.io/osminogin/tor-simple cat /var/lib/tor/lws_hidden_service/hostname
```

which will NOT print out the secret keys for the server, just the public
onion address. `Skylight` users can use this address directly when Tor is
enabled, and `lwcli` (and similar) users will need to install Tor and set proxy
settings to resolve the onion address.

### Port Forwarding
If you know how to forward ports of your router, it is preferrable for the
health of the Monero p2p network to set this up. You should forward external
port 18080 to the local ip of your host box on port 18080.

## Basic Administration
Your LWS server will automatically create new accounts when people request it.
If your onion address leaks publicly, people could fill up your system with
bogus addresses. You can prevent this by removing `--auto-accept-creation`
from your `compose.yml` file, and then running:

```bash
docker compose stop
docker compose run -d
```

which will restart all of the services with the new configuration.

### List Requests
If you prevented new account creation, or have users trying to "import" their
existing wallets which requires a rescan, you can view these requests
by running:

```bash
docker run --rm -v lws-hidden-service_lws:/home/monero-lws --entrypoint monero-lws-admin ghcr.io/vtnerd/monero-lws list_requests
```

which will list every request to create and import wallets. Accepting/rejecting
is accomplished by running one of the commands below:

```bash
docker run --rm -v lws-hidden-service_lws:/home/monero-lws --entrypoint monero-lws-admin ghcr.io/vtnerd/monero-lws accept_requests create <address>
docker run --rm -v lws-hidden-service_lws:/home/monero-lws --entrypoint monero-lws-admin ghcr.io/vtnerd/monero-lws accept_requests import <address>
docker run --rm -v lws-hidden-service_lws:/home/monero-lws --entrypoint monero-lws-admin ghcr.io/vtnerd/monero-lws reject_requests create <address>
docker run --rm -v lws-hidden-service_lws:/home/monero-lws --entrypoint monero-lws-admin ghcr.io/vtnerd/monero-lws reject_requests import <address>
```

and replacing `<address>` with their Monero wallet address.

### Disabling/Hiding Address
If there is a user that you which to disable or hide, you can achieve either
with one of the commands:

```bash
docker run --rm -v lws-hidden-service_lws:/home/monero-lws --entrypoint monero-lws-admin ghcr.io/vtnerd/monero-lws modify_account_status active <address>
docker run --rm -v lws-hidden-service_lws:/home/monero-lws --entrypoint monero-lws-admin ghcr.io/vtnerd/monero-lws modify_account_status inactive <address>
docker run --rm -v lws-hidden-service_lws:/home/monero-lws --entrypoint monero-lws-admin ghcr.io/vtnerd/monero-lws modify_account_status hidden <address>
```

`active` is the default state, `inactive` stops scanning the address but keeps
the address accessible via API, and `hidden` stops scanning and prevents access
to the historically scanned data entirely.

### Manual Rescan
A manual rescan is done by running:

```bash
docker run --rm -v lws-hidden-service_lws:/home/monero-lws --entrypoint monero-lws-admin ghcr.io/vtnerd/monero-lws rescan <height> <address>
```

where `<height>` and `<address>` must be replaced to their desired values.

## Updating Containers
Updating to the latest version of Tor, monerod, and LWS is achieved by running:

```bash
docker pull ghcr.io/sethforprivacy/simple-monerod
docker pull osminogin/tor-simple
docker pull ghcr.io/vtnerd/monero-lws
```

and then inspecting the output of the line that starts with `Status:`. If all
3 indicate they are up-to-date, then there is nothing left to do. If any of the
3 updated, then run the command:

```bash
docker compose down
docker compose up -d
```

from the directory where `compose.yml` was saved.

