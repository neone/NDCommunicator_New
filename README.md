The server has been completely rewriten in Go and all the original
functionality works. An optional implementation of a Selective Forwarding Unit
(SFU) is available to make Peer Calls consume less bandwith for user video
uploads. This wouldn't haven been possible without the awesome
[pion/webrtc][pion] library.

[pion]: https://github.com/pion/webrtc

# Endpoints

GET('/') - returns index page where you can enter custom room name, else, one is generated for you with UUID

POST('/call’) - routes the initial requestor of the room to their newly created room

GET('/call/{callID}') - gets the room with the passed in callID

GET('/probes/liveness') - for monitoring, checks if the server is up

GET('/metrics') - authorization based metrics viewing – uses prometheus.

# Requirements

 - [Go 1.14][go]

Alternatively, [Docker][docker] can be used to run Peer Calls.

[go]: https://golang.org/
[docker]: https://www.docker.com/

# Stack

## Backend

 - [Golang][go]
 - [pion/webrtc][pion]
 - github.com/go-chi/chi
 - nhooyr.io/websocket

[pion]: https://github.com/pion/webrtc

See [go.mod](go.mod) for more information

## Frontend

 - React
 - Redux
 - TypeScript (since peer-calls `v2.1.0`)

See [package.json](package.json) for more information.

# Installation & Running

## Download Release

Head to [Releases](https://github.com/peer-calls/peer-calls/releases) and
download a precompiled version. Currently the binaries for the following
systems are built automatically:

 - linux amd64
 - linux arm
 - darwin (macOS) amd64
 - windows amd64

## Deploying onto Kubernetes

The root of this repository contains a `kustomization.yaml`, allowing anyone to
patch the manifests found within the `deploy/` directory. To deploy the manifests
without applying any patches, pass the URL to `kubectl`:

```bash
kubectl apply -k github.com/peer-calls/peer-calls
```

## Using Docker

Use the [`peercalls/peercalls`][hub] image from Docker Hub:

```bash
docker run --rm -it -p 3000:3000 peercalls/peercalls:latest
```

[hub]: https://hub.docker.com/r/peercalls/peercalls

## Building from Source

```bash
git clone https://github.com/peer-calls/peer-calls.git
cd peer-calls
npm install

# for production
npm run build
npm run build:go:linux

# for development
npm run start
```

## Building Docker Image

```bash
git clone https://github.com/peer-calls/peer-calls
cd peer-calls
docker build -t peer-calls .
docker run --rm -it -p 3000:3000 peer-calls
```

# Configuration

## Environment variables


| Variable                             | Type   | Description                                                                  | Default   |
|--------------------------------------|--------|------------------------------------------------------------------------------|-----------|
| `PEERCALLS_LOG`                      | csv    | Enables or disables logging for certain modules                              | `-sdp,-ws,-nack,-rtp,-rtcp,-pion:*:trace,-pion:*:debug,-pion:*:info,*` |
| `PEERCALLS_BASE_URL`                 | string | Base URL of the application                                                  |           |
| `PEERCALLS_BIND_HOST`                | string | IP to listen to                                                              | `0.0.0.0` |
| `PEERCALLS_BIND_PORT`                | int    | Port to listen to                                                            | `3000`    |
| `PEERCALLS_TLS_CERT`                 | string | Path to TLS PEM certificate. If set will enable TLS                          |           |
| `PEERCALLS_TLS_KEY`                  | string | Path to TLS PEM cert key. If set will enable TLS                             |           |
| `PEERCALLS_STORE_TYPE`               | string | Can be `memory` or `redis`                                                   | `memory`  |
| `PEERCALLS_STORE_REDIS_HOST`         | string | Hostname of Redis server                                                     |           |
| `PEERCALLS_STORE_REDIS_PORT`         | int    | Port of Redis server                                                         |           |
| `PEERCALLS_STORE_REDIS_PREFIX`       | string | Prefix for Redis keys. Suggestion: `peercalls`                               |           |
| `PEERCALLS_NETWORK_TYPE`             | string | Can be `mesh` or `sfu`. Setting to SFU will make the server the main peer    | `mesh`    |
| `PEERCALLS_NETWORK_SFU_INTERFACES`   | csv    | List of interfaces to use for ICE candidates, uses all available when empty  |           |
| `PEERCALLS_NETWORK_SFU_JITTER_BUFFER`| bool   | Set to `true` to enable the use of Jitter Buffer                             | `false`   |
| `PEERCALLS_NETWORK_SFU_PROTOCOLS`    | csv    | Can be `udp4`, `udp6`, `tcp4` or `tcp6`                                      | `udp4,udp6` |
| `PEERCALLS_NETWORK_SFU_TCP_BIND_ADDR`| string | ICE TCP bind address. By default listens on all interfaces.                  |           |
| `PEERCALLS_NETWORK_SFU_TCP_LISTEN_PORT`| int  | ICE TCP listen port. By default uses a random port.                          | `0`       |
| `PEERCALLS_ICE_SERVER_URLS`          | csv    | List of ICE Server URLs                                                      |           |
| `PEERCALLS_ICE_SERVER_AUTH_TYPE`     | string | Can be empty or `secret` for coturn `static-auth-secret` config option.      |           |
| `PEERCALLS_ICE_SERVER_SECRET`        | string | Secret for coturn                                                            |           |
| `PEERCALLS_ICE_SERVER_USERNAME`      | string | Username for coturn                                                          |           |
| `PEERCALLS_PROMETHEUS_ACCESS_TOKEN`  | string | Access token for prometheus `/metrics` URL                                   |           |

The default ICE servers in use are:

- `stun:stun.l.google.com:19302`
- `stun:global.stun.twilio.com:3478?transport=udp`

For production it is necessary to deploy our own custom STUN/TURN servers

Only a single ICE server can be defined via environment variables. To define
more use a YAML config file. To load a config file, use the `-c
/path/to/config.yml` command line argument.

See [config/types.go][config] for configuration types.

[config]: ./src/server/config/types.go

Example:

```yaml
base_url: ''
bind_host: '0.0.0.0'
bind_port: 3005
ice_servers:
 - urls:
   - 'stun:stun.l.google.com:19302'
- urls:
  - 'stun:global.stun.twilio.com:3478?transport=udp'
#- urls:
#  - 'turn:coturn.mydomain.com'
#  auth_type: secret
#  auth_secret:
#    username: "peercalls"
#    secret: "some-static-secret"
# tls:
#   cert: test.pem
#   key: test.key
store:
  type: memory
  # type: redis
  # redis:
  #   host: localhost
  #   port: 6379
  #   prefix: peercalls
network:
  type: mesh
  # type: sfu
  # sfu:
  #   interfaces:
  #   - eth0
prometheus:
  access_token: "mytoken"
```

Prometheus `/metrics` URL will not be accessible without an access token set.
The access token can be provided by either:

- Setting `Authorization` header to `Bearer mytoken`, or
- Providing the access token as a query string: `/metrics?access_token=mytoken`

To access the server, go to http://localhost:3000.

# Accessing From Network

Most browsers will prevent access to user media devices if the application is
accessed from the network (not via 127.0.0.1). If you wish to test your mobile
devices, you'll have to enable TLS by setting the `PEERCALLS_TLS_CERT` and
`PEERCALLS_TLS_KEY` environment variables. To generate a self-signed certificate
you can use:

```
openssl req -nodes -x509 -newkey rsa:4096 -keyout key.pem -subj "/C=US/ST=Oregon/L=Portland/O=Company Name/OU=Org/CN=example.com" -out cert.pem -days 365
```

Replace `example.com` with your server's hostname.

# Multiple Instances and Redis

Redis can be used to allow users connected to different instances to connect.
The following needs to be added to `config.yaml` to enable Redis:

```yaml
store:
  type: redis
  redis:
    host: redis-host  # redis host
    port: 6379        # redis port
    prefix: peercalls # all instances must use the same prefix
```

# Logging

By default, Peer Calls server will log only basic information. Client-side
logging is disabled by default.

Server-side logs can be configured via the `PEERCALLS_LOG` environment variable. Setting
it to `*` will enable all server-side logging:

- `PEERCALLS_LOG=*`

Client-side logs can be configured via `localStorage.DEBUG` and
`localStorage.LOG` variables:

- Setting `localStorage.log=1` enables logging of Redux actions and state
  changes
- Setting `localStorage.debug=peercalls,peercalls:*` enables all other
  client-side logging

# Development

Below are some common scripts used for development:

```
npm start              build all resources and start the server.
npm run build          build all client-side resources.
npm run start:server   start the server
npm run js:watch       build and watch resources
npm test               run all client-side tests.
go test ./...          run all server tests
npm run ci             run all linting, tests and build the client-side
```

# ICE TCP

Peer Calls supports ICE over TCP as described in RFC6544. Currently only
passive ICE candidates are supported. This means that users whose ISPs or
corporate firewalls block UDP packets can use TCP to connect to the SFU. In
most scenarios, this removes the need to use a TURN server, but this
functionality is currently experimental and is not enabled by default.

Add the `tcp4` and `tcp6` to your `PEERCALLS_NETWORK_SFU_PROTOCOLS` to enable
support for ICE TCP:

```
PEERCALLS_NETWORK_TYPE=sfu PEERCALLS_NETWORK_SFU_PROTOCOLS=`udp4,udp6,tcp4,tcp6` peer-calls
```

To test this functionality, `udp4` and `udp6` network types should be omitted:

```
PEERCALLS_NETWORK_TYPE=sfu PEERCALLS_NETWORK_SFU_PROTOCOLS=`tcp4,tcp6` peer-calls
```

Please note that in production the `PEERCALLS_NETWORK_SFU_TCP_LISTEN_PORT` should
be specified and external TCP access allowed through the server firewall.

# TURN Server

When a direct connection cannot be established, it might be help to use a TURN
server. The peercalls.com instance is configured to use a TURN server and it
can be used for testing. However, the server bandwidth there is not unlimited.

Here are the steps to install a TURN server on Ubuntu/Debian Linux:

```bash
sudo apt install coturn
```

Use the following configuration as a template for `/etc/turnserver.conf`:

```bash
lt-cred-mech
use-auth-secret
static-auth-secret=p4ssw0rd
realm=example.com
total-quota=300
cert=/etc/letsencrypt/live/rtc.example.com/fullchain.pem
pkey=/etc/letsencrypt/live/rtc.example.com/privkey.pem
log-file=/dev/stdout
no-multicast-peers
proc-user=turnserver
proc-group=turnserver
```

Change the `p4ssw0rd`, `realm`  and paths to server certificates.

Use the following configuration for Peer Calls:

```yaml
iceServers:
- urls:
  - 'turn:rtc.example.com'
  auth_type: secret
  auth_secret:
    username: 'example'
    secret: 'p4ssw0rd'
```

Finally, enable and start the `coturn` service:

```bash
sudo systemctl enable coturn
sudo systemctl start coturn
```

