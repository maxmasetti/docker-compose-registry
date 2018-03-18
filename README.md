# Docker registry cache

## Concept

- Create a Docker registry cache for local network to speed up docker image download and keep a lower bandwidth.
- Use docker-compose.

## Server setup

Clone this repo:

```bash
$ git clone git@github.com:maxmasetti/docker-compose-registry.git registry
```
or 

```bash
$ git clone https://github.com/maxmasetti/docker-compose-registry.git registry
```
and start it up:

```bash
$ cd registry
$ docker-compose up
```

and it's done.

## Client setup

Modify o create the file `/etc/docker/daemon.json` adding the local mirror pointing to your server on port 5000:

```json
{
  "registry-mirrors": ["https://<my-docker-mirror-host-ip>:5000"]
}
```

Then restart docker daemon:

```bash
$ service docker stop
$ service docker sart
```

## Test
Verify that everything is working correctly running a new image on the client while watching server's log:

```bash
$ docker image ls -a
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
hello-world                   latest              48b5124b2768        14 months ago       1.84kB
```
remove hello-world if exists:

```bash
$ docker image rm 48b5124b2768
```
Verify your registry cache is empty:

```bash
$ curl http://<my-docker-mirror-host-ip>:5000/v2/_catalog
# outputs -> {"repositories":[]}
```

Run hello-world:

```bash
$ docker run hello-world
```
and verify it is present on your registry cache:

```bash
$ curl http://<my-docker-mirror-host-ip>:5000/v2/_catalog
# outputs -> {"repositories":["library/busybox"]}
```
In the server log some line should pass showing desired activity.
The images downloaded should be cached in the `data` folder and listed in the nested folder `data/docker/registry/v2/repositories/library/`.


## Description

### Docker-compose file

`docker-compose.yml`:

```yaml
version: '3.3'

services:
  registry:
    restart: always
    image: registry:latest
    ports:
      - 5000:5000
    volumes:
      - ./config/default/config.yml:/etc/docker/registry/config.yml:ro
      - ./data:/var/lib/registry:rw
  #environment:
    #- "STANDALONE=true"
    #- "MIRROR_SOURCE=https://registry-1.docker.io"
    #- "MIRROR_SOURCE_INDEX=https://index.docker.io"
```
 
### Registry configuration (server)

`/etc/docker/registry/config.yml`:

```yaml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
proxy:
  remoteurl: https://registry-1.docker.io
  # username: [username]
  # password: [password]
```

## Client setup

`/etc/docker/daemon.json`:

```json
{
  "registry-mirrors": ["https://<my-docker-mirror-host-ip>:5000"]
}
```

### Documentation
- [Registry configuration](https://docs.docker.com/registry/configuration/)
- [Registry as a pull through cache](https://docs.docker.com/registry/recipes/mirror/)

`;)`