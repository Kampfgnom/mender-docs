---
title: Mender Server
taxonomy:
    category: docs
---

This document details troubleshooting steps for the most common problems with the Mender server.
The first part applies to all installations, while the section below on Production installations
only applies when the Mender server is [installed for production](../../administration/production-installation).

## Cleaning up the deviceauth database after device decommissioning
It is possible that after the device decommissioning operation there will be some unaccessible and unnecessary data in the deviceauth database.
In this case, you should clean your deviceauth database.

Is is recommended to backup your data before performing the clean up operation.
The [Backup and restore](../../administration/backup-and-restore) chapter provides examples and
introduces example tools provided in Mender integration repository.

To clean up the deviceauth database, run the following from within the integration repository:
```
docker-compose exec mender-device-auth /usr/bin/deviceauth maintenance --decommissioning-cleanup
```

## The virtual QEMU device is not showing up in demo mode

When running the Mender server in demo mode, as described in the [getting started tutorial](../../getting-started/deploy-to-virtual-devices),
a virtual `qemux86-64` device should connect to and ask to join the server.

If this does not happen, please make sure your environment meet the resource requirements
to run the Mender Server. In particular, it is known that the virtual device will not
start if you do not have enough memory.


## A device shows up as pending after preauthorizing it
If you see your device gets the `pending` status after [preauthorizing it](../../server-integration/preauthorizing-devices), something went wrong. Most likely there is a mismatch between the identity and public key [you preauthorized](../../server-integration/preauthorizing-devices#call-the-preauthorize-api) and what your Mender client is actually using. Note that the Mender server does an exact string match against the device identity and public key that is preauthorized and the data your device reports, so any whitespace or string mismatch (in either the identity or public key) will lead to the Mender server not matching your device with the preauthorized data.

To diagnose this, look for the device identity in the `admission` service, for example:

```bash
curl -H "Authorization: Bearer $JWT" $MENDER_SERVER_URI/api/management/v1/admission/devices | python -m json.tool
```

```json

[
    {
        "attributes": {
            "mac": "52:54:00:50:9b:84"
        },
        "device_id": "5afdcccf8b89f00001fc40d7",
        "device_identity": "{\"mac\":\"52:54:00:50:9b:84\"}",
        "id": "5afdcccf8b89f00001fc40d6",
        "key": "-----BEGIN PUBLIC KEY-----MIIBojANBgkqhkiG9w0BAQEFAAOCAY8AMIIBigKCAYEAvOsee2ivRTkpA2GNWsjd6fH4OgAYwheHkY1U9i2GaPdYbhQb4hBUFoOLhdPFx5wwEqxJ8LnJOJBYywUthv59iJy01w4RTPiTEs3A6eXGdiLO0/RqsWqK5z2KeYiCrI52oE63pY6Y0JEZBpqzs2V9WsLOn6cnQU6HzHltYIuRpzwZWTWFxAFuU+FvDvj9QmD/Y6tos0yaMhfhpgJj3Iw9uARkFAv4DVn+HeA14PPVzHD4xJPUHL6H8FMfeIylejzaOnNHn6vkrvpuMQSvvZjlkH+uV7N93kj3JxSJ2LL9oMY9EargUkT0covZPdAE0G3wwNYCAIYRclzvI1w3DZ03oK2HCveVzFkBPbCwt4/pDReVzlRbQJ6CHkZqCbipoEH0/Ucetzp9fJ3mW3jE2yH1JK8nnpprbNYOCA988s6q3ifxbR6nWkPTbG3JyZL3ythV1o7FgOcwyKh8bneHoZaOa9BnNrHkDz9uG1Xwbe6As62QyZjk2pjQswswQsh/6AvrAgMBAAE=-----END PUBLIC KEY-----",
        "request_time": "2018-05-17T18:41:19.546Z",
        "status": "preauthorized"
    },
    {
        "attributes": {
            "mac": "52:54:00:50:9b:84"
        },
        "device_id": "5afdcccf8b89f00001fc40d7",
        "device_identity": "{\"mac\":\"52:54:00:50:9b:84\"}",
        "id": "5afdccd1226abc0001ac0c16",
        "key": "-----BEGIN PUBLIC KEY-----\nMIIBojANBgkqhkiG9w0BAQEFAAOCAY8AMIIBigKCAYEAvOsee2ivRTkpA2GNWsjd\n6fH4OgAYwheHkY1U9i2GaPdYbhQb4hBUFoOLhdPFx5wwEqxJ8LnJOJBYywUthv59\niJy01w4RTPiTEs3A6eXGdiLO0/RqsWqK5z2KeYiCrI52oE63pY6Y0JEZBpqzs2V9\nWsLOn6cnQU6HzHltYIuRpzwZWTWFxAFuU+FvDvj9QmD/Y6tos0yaMhfhpgJj3Iw9\nuARkFAv4DVn+HeA14PPVzHD4xJPUHL6H8FMfeIylejzaOnNHn6vkrvpuMQSvvZjl\nkH+uV7N93kj3JxSJ2LL9oMY9EargUkT0covZPdAE0G3wwNYCAIYRclzvI1w3DZ03\noK2HCveVzFkBPbCwt4/pDReVzlRbQJ6CHkZqCbipoEH0/Ucetzp9fJ3mW3jE2yH1\nJK8nnpprbNYOCA988s6q3ifxbR6nWkPTbG3JyZL3ythV1o7FgOcwyKh8bneHoZaO\na9BnNrHkDz9uG1Xwbe6As62QyZjk2pjQswswQsh/6AvrAgMBAAE=\n-----END PUBLIC KEY-----\n",
        "request_time": "2018-05-17T18:41:21.483Z",
        "status": "pending"
    }
]
```

In this case you can see that there are two authentication sets with the exact same device identity string: `"{\"mac\":\"52:54:00:50:9b:84\"}"`, one `preauthorized` and one `pending`. So the device reported (see the `pending` set) the exact same identity as we preauthorized. However, there is a mismatch in the format of the public key: the pending set, which the device reported, contains additional `\n` characters.

The solution is to decommission the device and [remove all authentication sets](../../server-integration/preauthorizing-devices#make-sure-there-are-no-existing-authentication-sets-for-your-dev) (from both `admission` and `devauth`) and adjust the public key string used in the [preauthorize API call](../../server-integration/preauthorizing-devices#call-the-preauthorize-api) to match exactly the one reported by the device, as seen in the `pending` data above.


## Install a Mender demo server on Windows

The Mender [getting started guide](../../getting-started) assumes you are using Ubuntu Linux to install the Mender server.
Docker images used should be compatible with a Windows host as well, however there are some server maintenance scripts,
like `up`, that are Unix-specific.

So currently the easiest way to use a Windows workstation to run the Mender demo server is to get a virtual
machine with Ubuntu Linux and start the Mender server in there.


## mender-api-gateway exits with code 132

When starting the Mender server, `mender-api-gateway` exits immediately with a message
similar to this:

```bash
mender-api-gateway_1 exited with code 132
```

The logs from docker-compose also show that `mender-api-gateway` exits with code 132:

```bash
2017-07-03T11:14:17.223223822+02:00 container die c2a1cf1c19651950e804c7fe53cb9c0ffeb8b71de744500ab4844b370cf0480d (com.docker.compose.config-hash=5b9dfb050a59b9cbd67082037478562ff44c78cf4346d9a83225d1a74d557271, com.docker.compose.container-number=1, com.docker.compose.oneoff=False, com.docker.compose.project=menderproduction, com.docker.compose.service=mender-api-gateway, com.docker.compose.version=1.14.0, exitCode=132, image=mendersoftware/api-gateway:1.1.x, name=menderproduction_mender-api-gateway_1)
```

The problem here is most likely that your server CPU does not support the SSE 4.2
instruction set, while [openresty is compiled assuming SSE4.2 support](https://github.com/openresty/openresty/issues/267?target=_blank).
Try again with a more modern CPU (from 2009 or later), or consider building
openresty from source for your architecture.


# Production installations

For the rest of this document, it is assumed that commands are run through production
helper script `run` as detailed in the [production installation documentation](../../administration/production-installation).

## Listing active containers

Listing containers and their statuses is a first step in any troubleshooting
scenario. Assuming current working directory is `production` we can call
production helper script like this:


```
user@local$ ./run ps
                   Name                                  Command               State           Ports
-------------------------------------------------------------------------------------------------------------
menderproduction_mender-api-gateway_1         /entrypoint.sh                   Up      0.0.0.0:443->443/tcp
menderproduction_mender-deployments_1         /entrypoint.sh                   Up      8080/tcp
menderproduction_mender-device-adm_1          /usr/bin/deviceadm -config ...   Up      8080/tcp
menderproduction_mender-device-auth_1         /usr/bin/deviceauth -confi ...   Up      8080/tcp
menderproduction_mender-gui_1                 /entrypoint.sh                   Up
menderproduction_mender-inventory_1           /usr/bin/inventory -config ...   Up      8080/tcp
menderproduction_mender-mongo_1               /entrypoint.sh mongod            Up      27017/tcp
menderproduction_mender-useradm_1             /usr/bin/useradm -config / ...   Up      8080/tcp
menderproduction_minio_1                      minio server /export             Up      9000/tcp
menderproduction_storage-proxy_1              /usr/local/openresty/bin/o ...   Up      0.0.0.0:9000->9000/tcp
```

Alternatively, the same information can be obtained by running `docker ps`
directly with a label filter, like this:

```
user@local$ docker ps --filter label=com.docker.compose.project=menderproduction
CONTAINER ID        IMAGE                                               COMMAND                  CREATED             STATUS              PORTS                    NAMES
c668a91617ea        mendersoftware/api-gateway:latest                   "/entrypoint.sh"         39 minutes ago      Up 39 minutes       0.0.0.0:443->443/tcp     menderproduction_mender-api-gateway_1
9ebb2fc86d0c        mendersoftware/useradm:latest                       "/usr/bin/useradm ..."   40 minutes ago      Up 39 minutes       8080/tcp                 menderproduction_mender-useradm_1
566a2a3c3773        mendersoftware/inventory:latest                     "/usr/bin/inventor..."   40 minutes ago      Up 39 minutes       8080/tcp                 menderproduction_mender-inventory_1
d33f8b4af1bd        mendersoftware/deployments:latest                   "/entrypoint.sh"         40 minutes ago      Up 39 minutes       8080/tcp                 menderproduction_mender-deployments_1
0bb97d64ee4f        mendersoftware/deviceadm:latest                     "/usr/bin/devicead..."   40 minutes ago      Up 39 minutes       8080/tcp                 menderproduction_mender-device-adm_1
1ddbad5520e9        mendersoftware/deviceauth:latest                    "/usr/bin/deviceau..."   40 minutes ago      Up 39 minutes       8080/tcp                 menderproduction_mender-device-auth_1
e7ad33929628        mendersoftware/openresty:1.11.2.2-alpine            "/usr/local/openre..."   40 minutes ago      Up 39 minutes       0.0.0.0:9000->9000/tcp   menderproduction_storage-proxy_1
cdaab7768ec7        mongo:3.4                                           "/entrypoint.sh mo..."   40 minutes ago      Up 40 minutes       27017/tcp                menderproduction_mender-mongo_1
867e76066fad        mendersoftware/gui:latest                           "/entrypoint.sh"         40 minutes ago      Up 40 minutes                                menderproduction_mender-gui_1
54ae287d24ac        mendersoftware/minio:RELEASE.2016-12-13T17-19-42Z   "minio server /export"   40 minutes ago      Up 40 minutes       9000/tcp                 menderproduction_minio_1
```
## Service keeps restarting

!!! Mender service containers are configured with restart policy `on-failure`.

`docker-compose ps` may be helpful with identification of containers that are
restarting:

```
user@local$ ./run ps
                   Name                                  Command                 State              Ports
------------------------------------------------------------------------------------------------------------------
menderproduction_mender-api-gateway_1         /entrypoint.sh                   Up           0.0.0.0:443->443/tcp
menderproduction_mender-deployments_1         /entrypoint.sh                   Restarting
menderproduction_mender-device-adm_1          /usr/bin/deviceadm -config ...   Up           8080/tcp
menderproduction_mender-device-auth_1         /usr/bin/deviceauth -confi ...   Up           8080/tcp
menderproduction_mender-gui_1                 /entrypoint.sh                   Up
menderproduction_mender-inventory_1           /usr/bin/inventory -config ...   Up           8080/tcp
menderproduction_mender-mongo_1               /entrypoint.sh mongod            Up           27017/tcp
menderproduction_mender-useradm_1             /usr/bin/useradm -config / ...   Up           8080/tcp
menderproduction_minio_1                      minio server /export             Up           9000/tcp
menderproduction_storage-proxy_1              /usr/local/openresty/bin/o ...   Up           0.0.0.0:9000->9000/tcp
```
In the case presented above, `mender-deployments` is restarting.

!! `docker-compose` may show `Restating` status for containers that are restarting in a quick succession, if containers restart after a longer while, they may appear as `Up`

In situations where some containers are restarting after running for a longer
while, `docker ps` will show the containers having a shorter lifetime than
others. In the listing show below, `mender-deployments` service uptime is
shorter than that of the other containers:

```
user@local$ docker ps --filter label=com.docker.compose.project=menderproduction
CONTAINER ID        IMAGE                                               COMMAND                  CREATED             STATUS              PORTS                    NAMES
2f4c700923f9        mendersoftware/api-gateway:latest                   "/entrypoint.sh"         4 minutes ago       Up 4 minutes        0.0.0.0:443->443/tcp     menderproduction_mender-api-gateway_1
8f46579aefa4        mendersoftware/deployments:latest                   "/entrypoint.sh"         5 minutes ago       Up 54 seconds       8080/tcp                 menderproduction_mender-deployments_1
7236def09c97        mendersoftware/useradm:latest                       "/usr/bin/useradm ..."   5 minutes ago       Up 5 minutes        8080/tcp                 menderproduction_mender-useradm_1
be9cc9ee74b2        mendersoftware/deviceauth:latest                    "/usr/bin/deviceau..."   5 minutes ago       Up 5 minutes        8080/tcp                 menderproduction_mender-device-auth_1
aa70b181d490        mendersoftware/deviceadm:latest                     "/usr/bin/devicead..."   5 minutes ago       Up 5 minutes        8080/tcp                 menderproduction_mender-device-adm_1
5d84b70d1187        mendersoftware/inventory:latest                     "/usr/bin/inventor..."   5 minutes ago       Up 5 minutes        8080/tcp                 menderproduction_mender-inventory_1
1dc4843ec4db        mendersoftware/openresty:1.11.2.2-alpine            "/usr/local/openre..."   5 minutes ago       Up 5 minutes        0.0.0.0:9000->9000/tcp   menderproduction_storage-proxy_1
fb216139bc60        mongo:3.4                                           "/entrypoint.sh mo..."   5 minutes ago       Up 5 minutes        27017/tcp                menderproduction_mender-mongo_1
9d6eacd2ae83        mendersoftware/gui:latest                           "/entrypoint.sh"         5 minutes ago       Up 5 minutes                                 menderproduction_mender-gui_1
cc0b330a3cd5        mendersoftware/minio:RELEASE.2016-12-13T17-19-42Z   "minio server /export"   5 minutes ago       Up 5 minutes        9000/tcp                 menderproduction_minio_1
```

### Docker event log

Docker event monitor provides a dynamic view of events. This feature can be
helpful, especially with filters applied. Example view of events registered when
a `mender-deployments` container was restarting (output edited for clarity):

```
user@local$ docker events --filter label=com.docker.compose.project=menderproduction
2017-01-31T09:14:13.291589609+01:00 container die 8f46579aefa47b717c79c4216131391fba7fc938b276d1469d0944691a740d37
   (com.docker.compose.config-hash=80ec90a07e4567c923a6f54126052f4e73a68373113b0dc460d7488b8be9761b,
    com.docker.compose.container-number=1, com.docker.compose.oneoff=False,
    com.docker.compose.project=menderproduction, com.docker.compose.service=mender-deployments,
    com.docker.compose.version=1.10.0, exitCode=1,
    image=mendersoftware/deployments:latest, name=menderproduction_mender-deployments_1)
2017-01-31T09:14:14.699717693+01:00 container start 8f46579aefa47b717c79c4216131391fba7fc938b276d1469d0944691a740d37
   (com.docker.compose.config-hash=80ec90a07e4567c923a6f54126052f4e73a68373113b0dc460d7488b8be9761b,
    com.docker.compose.container-number=1, com.docker.compose.oneoff=False,
    com.docker.compose.project=menderproduction, com.docker.compose.service=mender-deployments,
    com.docker.compose.version=1.10.0,
    image=mendersoftware/deployments:latest, name=menderproduction_mender-deployments_1)
```

## Container logs

Container logs provide access to logging output from the container. Example listing:

```
user@local$ ./run logs --tail 10 mender-deployments
Attaching to menderproduction_mender-deployments_1
mender-deployments_1        | WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
mender-deployments_1        | time="2017-01-31T08:18:26Z" level=fatal msg="NoCredentialProviders: no valid providers in chain. Deprecated. \n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors" file=proc.go func=runtime.main line=183
mender-deployments_1        | WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
mender-deployments_1        | time="2017-01-31T08:18:57Z" level=fatal msg="NoCredentialProviders: no valid providers in chain. Deprecated. \n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors" file=proc.go func=runtime.main line=183
mender-deployments_1        | WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
mender-deployments_1        | time="2017-01-31T08:19:56Z" level=fatal msg="NoCredentialProviders: no valid providers in chain. Deprecated. \n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors" file=proc.go func=runtime.main line=183
mender-deployments_1        | WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
mender-deployments_1        | time="2017-01-31T08:21:44Z" level=fatal msg="NoCredentialProviders: no valid providers in chain. Deprecated. \n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors" file=proc.go func=runtime.main line=183
mender-deployments_1        | WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
mender-deployments_1        | time="2017-01-31T08:25:15Z" level=fatal msg="NoCredentialProviders: no valid providers in chain. Deprecated. \n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors" file=proc.go func=runtime.main line=183
```

The same log can be obtained by running ``docker log`:

```
user@local$ docker logs --tail 10  menderproduction_mender-deployments_1
WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
time="2017-01-31T08:18:26Z" level=fatal msg="NoCredentialProviders: no valid providers in chain. Deprecated. \n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors" file=proc.go func=runtime.main line=183
WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
time="2017-01-31T08:18:57Z" level=fatal msg="NoCredentialProviders: no valid providers in chain. Deprecated. \n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors" file=proc.go func=runtime.main line=183
WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
time="2017-01-31T08:19:56Z" level=fatal msg="NoCredentialProviders: no valid providers in chain. Deprecated. \n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors" file=proc.go func=runtime.main line=183
WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
time="2017-01-31T08:21:44Z" level=fatal msg="NoCredentialProviders: no valid providers in chain. Deprecated. \n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors" file=proc.go func=runtime.main line=183
WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
time="2017-01-31T08:25:15Z" level=fatal msg="NoCredentialProviders: no valid providers in chain. Deprecated. \n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors" file=proc.go func=runtime.main line=183
```

## Inspecting containers

As seen in [container logs](#container-logs) section, `mender-deployments`
service is restarting. The logs suggest there might be missing credentials for
an AWS related service.
From the [production installation](../../administration/production-installation) guide, we can recall
that
`mender-deployments`
[service configuration](../../administration/production-installation#deployments-service) contains
credentials for artifact storage service.

Configuration of current instance of `mender-deployments` can be viewed using
`docker inspect` command. Looking for `Env` (container environment
configuration) in inspect output reveals that `DEPLOYMENTS_AWS_AUTH_KEY` and
`DEPLOYMENTS_AWS_AUTH_SECRET` are unset:

```
user@local$ docker inspect menderproduction_mender-deployments_1 |& less
...
            "StdinOnce": false,
            "Env": [
                "DEPLOYMENTS_AWS_AUTH_KEY",
                "DEPLOYMENTS_AWS_AUTH_SECRET",
                "DEPLOYMENTS_AWS_URI=https://s3.example.com:9000",
                "STORAGE_BACKEND_CERT=/etc/ssl/certs/storage-proxy.crt",
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
...
```

`docker inspect` output contains all information about container instance,
volumes, network, aliases etc.
