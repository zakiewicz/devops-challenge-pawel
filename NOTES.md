# Notes

This README file should contain notes used while working on the coding challenge.

---
### Run with compose
In repository root:
```
docker compose up
```
or
```
docker-compose up
```

### Test with curl
```
curl http://localhost/health
curl http://localhost/nodejs/hello
curl http://localhost/golang/hello
for n in a=2 b=4 c=9; do for s in $(seq ${n#*=}); do echo "name ${n%%=*} hit $s" ; curl http://localhost/golang/inc-score?name=${n%%=*} ; done ; echo ; done
curl http://localhost/golang/get-scores
```
---
## Challenge notes

To keep setup simple, I am going to use host network. Alternative would be to use bridged network and:
- Change Nodejs app to listen `eth0` rather than `lo`
- In compose file:
    - configure bridged network
        ```
        networks:
          bridged_net:
            driver: bridge
            ipam:
            config:
                - subnet: 172.17.1.0/29
                  gateway: 172.17.1.1
        ```
    - specify IP addresses for both app containers and expose ports
        ```
        services:
          nodejs:
            ...
            networks:
            bridged_net:
                ipv4_address: 172.17.1.2
            expose:
              - "3000"
          golang:
            ...
            networks:
            bridged_net:
                ipv4_address: 172.17.1.3
            expose:
              - "3002"
        ```
    - map nginx port to host's localhost:80
        ```
        nginx:
          ports:
            - "127.0.0.1:80:18181/tcp"
        ```
- In nginx `default.conf` use corresponding IP addresses for `proxy_pass` targets.
Above solution is not optimal, as it requires hard coded addresses in compose file and matching nginx config.

Alternatively, all addresses could be passed as variables to compose, while for nginx template config could be created and entrypoint script that would generate config from template using `envsubst` using container env variables set by compose. 

<br>

### Golang app
- Modified Go app to listen on loopback only.
- Empty file `dockerfile` - replaced with `Dockerfile` that:
    - Uses official golang image from dockerhub - version can be passed as build arg `GO_IMG_TAG` (default latest alpine)
    - Installs `curl` - needed by container healthcheck (executed locally!)
    - Copies Go app `app.go` to `/app/` (owned as `nobody:nobody`)
    - Sets `GOCACHE` env var to `/app/.cache/go-build` (so app can be build and run by `nobody`)
    - Changes user that app will be executed to `nobody` (no need for `root` since app binds to high port)
    - Changes working dir to `/app`
    - Executes `go run app.go` at container start

Adding `RUN go build` to build the app could be considered, however, since it is very small, executing `go run` at startup seems to be simpler (however I am not a Golang expert! Do not even speak Go really).

<br>

### Nodejs app
- Modified `Dockerfile` to:
    - add argument for 'build' node image - `NODE_BUILD_IMG_TAG`, default `16` (Debian based, big!)
    - add argument for 'run' node image - `NODE_RUN_IMG_TAG`, default `alpine` (Node 18, alpine based, small)
    - introduce stages:
        - `npm-install` using Debian based image to execute `npm install`
        - 'run' stage for final container based on alpine image to run application (I am not very familiar with node, I am not sure I am coping right files from 'build' stage)
    - install `curl` - needed by container healthcheck (executed locally!)
    - run app as `node` user

<br>

### Nginx
- Modified `Dockerfile` to:
    - add image tag as argument `NGINX_IMG_TAG` default `apline`
    - removed installing `jq` and `curl` - not needed for normal operations
    - change default number of worker processes from `auto` to 1
- Modified configuration for `conf.d/default.conf`:
    - listen on loopback only and on port 80
    - replaced `localhost` with `127.0.0.1` to improve performance
    - passing all traffic matching `^/golang/.*$` to Golang app, so any endpoint is supported (same can be done for Nodejs location)
    - added support for websockets as per [nginx docs](https://nginx.org/en/docs/http/websocket.html)

---
## TODO
### Log collection!
Only nginx container exposes (some) logs that available via `docker logs`. Some means of collecting logs, either by writing to mounted volumes or to docker logs, so they can consumed by log collection solution.
Nodejs app writes logs in `/home/node/.npm/_logs/`.
I am not aware that Golang app writes any.

### Tests
Containers should be tested after build, however I am not sure how to do this with compose (yet!). After docker build step it would be beneficial to be able to execute, for example InSpec tests to verify that build produced container with all required files, etc.

### Limits
It is advisable to set CPU, mem limits for containers, so individual containers do not drain resources in case of problems.

### Handling for failed healthchecks

### Container and image tagging
### App configuration
In production applications' parameters, like ports to listen on, should be configurable, not hard-coded.

### Proxy configuration
For Golang app `location /golang/` is used meaning all requests with path beginning with `/golang/` are proxied to the app. This could be replaced with a regex covering all API endpoints. This would result with only requests to valid endpoints to be passed to the app and other requests being served 404 by proxy, thus reducing load on the app. However, as a downside, introducing new API endpoint would require rebuilding/reconfiguring proxy, not only app container.
