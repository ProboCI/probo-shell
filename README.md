# Probo Shell

Probo Shell is a web-based terminal emulator that allows users to run selected commands inside their Probo CI build containers to assist in debugging. It is built on [Wetty](https://github.com/krishnasrinivas/wetty) (Web + TTY) and uses Chrome's hterm library to provide a full terminal experience in the browser over HTTP.

## How It Works

1. A user connects to Probo Shell with a JWT token in the URL query string (`?auth=<token>`)
2. The server verifies the token and extracts the target container name from the JWT payload
3. The server locates the Docker container (starting it if stopped) and spawns a PTY session via `docker exec`
4. Keystrokes and terminal output are bridged between the browser and the container over Socket.IO
5. Output is sanitized to strip container names and internal IP addresses before reaching the client

## Operations

The `op` query parameter selects which command to run inside the container. If omitted, it defaults to `bash`.

| Operation | Description |
|---|---|
| `bash` | Interactive bash shell |
| `log.apache.access` | Tail Apache access log |
| `log.apache.error` | Tail Apache error log |
| `log.mysql.error` | Tail MySQL error log |
| `log.drupal.watchdog` | Drush watchdog log viewer |
| `log.php.error` | Tail PHP error log |

The `log-length` query parameter controls how many lines to show for tail-based operations. Set it to `all` to show the entire log.

Operations are defined in the YAML configuration and can be customized or extended.

## Setup

### Prerequisites

- Node.js 22+
- Access to a Docker socket (`/var/run/docker.sock`)
- Docker CLI available on the host

### Install

```
npm install
```

### Configuration

Configuration is loaded in layers: `defaults.yaml` → environment variables → CLI config files. Key settings:

| Setting | Description | Default |
|---|---|---|
| `shellSecret` | JWT verification secret | `SECRET` |
| `shellAuthQuery` | Query parameter name for the JWT token | `auth` |
| `server.port` | HTTP listen port | `3015` |
| `dockerConfig.socketPath` | Path to Docker socket | `/var/run/docker.sock` |
| `dockerConfig.version` | Docker API version | `v1.44` |
| `operations` | Map of operation names to command arrays | See `defaults.yaml` |
| `sanitizeStrings` | Regex patterns to scrub from terminal output | See `defaults.yaml` |
| `logging.enabled` | Write logs to a rotating file instead of stdout | `false` |
| `logging.path` | Log file path (parent directory is created if missing) | `/var/log/probo-shell/probo-shell.log` |
| `logging.period` | Rotation period (any value bunyan accepts, e.g. `1d`, `1h`) | `1d` |
| `logging.count` | Number of rotated files to retain | `7` |

When `logging.enabled` is `true`, output goes to the rotating log file and stdout logging is disabled. When `false`, logs are written to stdout and the file settings are ignored.

For local development, create a `shell.yaml` file (gitignored) by copying `defaults.yaml` and customizing as needed.

### Run Locally

```
./bin/startup-local.sh
```

This starts the server using `shell.yaml` for configuration.

### Run in Docker

```
docker build -t probo-shell .
docker run -v /var/run/docker.sock:/var/run/docker.sock -p 3015:3015 probo-shell
```

In production, mount your configuration file to `/etc/probo/shell.yaml`.

## Project Structure

```
app.js              Express + Socket.IO server, JWT auth, Docker/PTY bridging
lib/config.js       Layered YAML config loader with CLI and env var support
defaults.yaml       Default configuration (checked in)
shell.yaml          Local dev config (gitignored)
bin/probo-shell     Node entrypoint
bin/startup.sh      Production startup script (reads /etc/probo/shell.yaml)
bin/startup-local.sh  Local startup script (reads ./shell.yaml)
public/wetty/       Browser client (hterm terminal + Socket.IO bridge)
Dockerfile          Alpine-based production image with Docker CLI
Gruntfile.js        Grunt task to rebuild vendored hterm library
```
