# Docker registry cache

![License](https://img.shields.io/packagist/l/cakephp/app.svg?style=flat-square)

![Oyster registry](https://www.docker.com/sites/default/files/oyster-registry-3.png)

## Concept

- Create a Docker registry cache for local network to speed up docker image download and keep a lower bandwidth.
- Use docker-compose.

## Cache (server) setup

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
It's done.

### Optionally
Map the data folder to a better place: edit `docker-compose.yml` row

```yaml
    volumes:
      - ./data:/var/lib/registry:rw
```
and modify `./data` to fit your needs.


## Client setup

Modify or create the file `/etc/docker/daemon.json` and add the local mirror setting pointing to your server on port 5000:

```json
{
  "registry-mirrors": ["http://<my-docker-mirror-host-ip>:5000"]
}
```

Then restart docker daemon:

```bash
$ service docker stop
$ service docker sart
```

## Test
Verify that everything is working correctly running a new image on the client while watching server's log. First check if the required image is locally available,

```bash
$ docker image ls -a
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
...
hello-world                   latest              48b5124b2768        14 months ago       1.84kB
...
```
and remove it (hello-world in this case) if exists:

```bash
$ docker image rm 48b5124b2768
```
Verify your registry cache is empty. From the client query the server trhough the exposed API:

```bash
$ curl http://<my-docker-mirror-host-ip>:5000/v2/_catalog
# outputs -> {"repositories":[]}
```

In the client, run hello-world:

```bash
$ docker run hello-world
```
and verify it is present on your registry cache (server):

```bash
$ curl http://<my-docker-mirror-host-ip>:5000/v2/_catalog
# outputs -> {"repositories":["library/busybox"]}
```

In the server log some line should pass showing desired activity too.
The images downloaded will be stored in the server `data` folder and listed in the nested folder `data/docker/registry/v2/repositories/library/`.

## TBD
Secure the registry implementing ssl (https) on port 5000.

## Some description

### Docker-compose file (server)

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

### Client side

`/etc/docker/daemon.json`:

```json
{
  "registry-mirrors": ["http://<my-docker-mirror-host-ip>:5000/"]
}
```
On macOS, open preferences of Docker application from the menu bar, then go to  Daemon tab -> Advanced and insert the previous row. Apply and restart. Test if it works as expected.

![macOS registry setup](https://dl.dropboxusercontent.com/s/lietx1gxcbg621o/macOS-docker-registry-setup.png)


## Documentation
- [Registry configuration](https://docs.docker.com/registry/configuration/)
- [Registry as a pull through cache](https://docs.docker.com/registry/recipes/mirror/)
- [docker-compose](https://docs.docker.com/compose/compose-file/)

### Works this one is based on
- [laurent-malvert](https://github.com/laurent-malvert/docker-registry-proxy-cache)
- [docker-registry-setup](https://github.com/kwk/docker-registry-setup)
- [run-a-registry-as-a-service](https://github.com/docker/docker.github.io/blob/master/registry/deploying.md#run-a-registry-as-a-service)
- [mirror-cache-dockerhub-locally](https://hackernoon.com/mirror-cache-dockerhub-locally-for-speed-f4eebd21a5ca)


`;)`