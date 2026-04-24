# Tor Onion Service with Docker Compose and Lighttpd

This project is a simple way to publish a static website as a Tor onion service using Docker Compose and Lighttpd. The goal is to keep the setup small, understandable, and easy to maintain: Tor handles the onion service, Lighttpd serves the site, and the website files live in a local folder on the host machine so they can be edited directly without rebuilding the containers.

It is not meant to be a production platform with automation, orchestration, or advanced hardening out of the box. It is a clean starting point for anyone who wants to understand how a minimal onion site works and get one online quickly.

## Features

- Tor hidden service exposed only through the Tor network.
- Lighttpd serving static files from a local `html/` directory.
- Persistent onion hostname and keys via a mounted `hidden_service/` directory.
- Automatic content updates through Docker bind mounts.
- No public `80/443` port exposure on the host.

## Table of contents

- [Why this approach](#why-this-approach)
- [Project structure](#project-structure)
- [Requirements](#requirements)
- [How it works](#how-it-works)
- [Step 1: Create the project folders](#step-1-create-the-project-folders)
- [Step 2: Create the Tor configuration file](#step-2-create-the-tor-configuration-file)
- [Step 3: Create the Lighttpd configuration file](#step-3-create-the-lighttpd-configuration-file)
- [Step 4: Create a test page](#step-4-create-a-test-page)
- [Step 5: Create the Docker Compose file](#step-5-create-the-docker-compose-file)
- [Step 6: Start the stack](#step-6-start-the-stack)
- [Step 7: Retrieve the onion address](#step-7-retrieve-the-onion-address)
- [Step 8: Open the site in Tor Browser](#step-8-open-the-site-in-tor-browser)
- [Updating the website](#updating-the-website)
- [Useful commands](#useful-commands)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)
- [Final notes](#final-notes)
- [License](#license)

## Why this approach

There are more complex ways to host an onion site, but this setup has a few practical advantages. The web server stays private inside the Docker network, the onion hostname and keys are stored persistently, and the website content is stored locally on the host machine so it can be edited directly.

Because the site directory is mounted from the host into the container, changes made to the local files are immediately visible to the web server without rebuilding the image. That makes the setup convenient for testing, learning, and maintaining a small static onion site.

## Project structure

The directory layout is intentionally simple:

```text
tor/
├── compose.yml
├── lighttpd.conf
├── html/
│   └── index.html
└── tor/
    ├── torrc
    └── hidden_service/
```

## Requirements

Before starting, make sure you have:

- Docker installed
- Docker Compose installed
- A Linux host or VM
- Tor Browser for testing the final `.onion` address

## How it works

Tor publishes the onion service and forwards requests to the Lighttpd container over the internal Docker network. Lighttpd serves the static files from `/var/www/html`, while the actual files live in the local `html/` directory on the host through a bind mount.

Tor stores the generated onion hostname and private keys in the directory defined by `HiddenServiceDir`, which is why that folder must be kept private.

## Step 1: Create the project folders

Start by creating the working directories:

```bash
mkdir -p ~/tor/html
mkdir -p ~/tor/tor/hidden_service
cd ~/tor
```

This creates a directory for your website files, one for the Tor configuration, and one where Tor will store the onion hostname and keys.

## Step 2: Create the Tor configuration file

Create a file called:

```text
~/tor/tor/torrc
```

Open it with your preferred editor and paste in the following content:

```conf
DataDirectory /var/lib/tor
HiddenServiceDir /var/lib/tor/hidden_service
HiddenServicePort 80 lighttpd:80
```

### What this means

- `DataDirectory /var/lib/tor` tells Tor where to store its runtime data.
- `HiddenServiceDir /var/lib/tor/hidden_service` tells Tor where to store the onion service keys and generated hostname.
- `HiddenServicePort 80 lighttpd:80` tells Tor to forward incoming traffic on onion port 80 to the Lighttpd container on port 80.

## Step 3: Create the Lighttpd configuration file

Create a file called:

```text
~/tor/lighttpd.conf
```

Paste in the following content:

```conf
server.document-root = "/var/www/html"
server.port = 80
server.username = "lighttpd"
server.groupname = "lighttpd"
index-file.names = ( "index.html" )
mimetype.assign = (
  ".html" => "text/html",
  ".txt"  => "text/plain",
  ".css"  => "text/css",
  ".js"   => "application/javascript"
)
```

This configuration tells Lighttpd to serve files from `/var/www/html` and to use `index.html` as the default page. Defining the document root explicitly helps avoid the classic situation where the container is running but the browser returns a 404 because the web server is looking in the wrong place.

## Step 4: Create a test page

Now create a file called:

```text
~/tor/html/index.html
```

Paste in something simple like this:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Onion Service</title>
</head>
<body>
  <h1>Tor Onion Service is Working</h1>
  <p>If you can read this through Tor Browser, the setup is correct.</p>
</body>
</html>
```

This is only a test page, so you can replace it later with your actual site.

## Step 5: Create the Docker Compose file

Create a file called:

```text
~/tor/compose.yml
```

Paste in the following content:

```yaml
services:
  tor:
    image: alpine:latest
    container_name: tor
    command: >
      sh -c "
      apk add --no-cache tor &&
      mkdir -p /var/lib/tor/hidden_service &&
      chmod 700 /var/lib/tor/hidden_service &&
      tor -f /etc/tor/torrc"
    volumes:
      - ./tor/torrc:/etc/tor/torrc:ro
      - ./tor/hidden_service:/var/lib/tor/hidden_service
    depends_on:
      - lighttpd
    restart: unless-stopped
    networks:
      - tor-network

  lighttpd:
    image: alpine:latest
    container_name: lighttpd
    command: >
      sh -c "
      apk add --no-cache lighttpd &&
      lighttpd -D -f /etc/lighttpd/lighttpd.conf"
    volumes:
      - ./html:/var/www/html:ro
      - ./lighttpd.conf:/etc/lighttpd/lighttpd.conf:ro
    restart: unless-stopped
    networks:
      - tor-network

networks:
  tor-network:
    driver: bridge
```

A few important things are happening here. The Tor container reads its configuration from your local `torrc`, the hidden service directory is stored persistently on disk, and the site files are mounted into Lighttpd from the host. Lighttpd is not exposed on public host ports, so the site is intended to be reachable only through Tor.

## Step 6: Start the stack

Once all files are in place, start everything with:

```bash
docker compose up -d
```

If the configuration is correct, Docker should start both containers in the background. You can verify that with:

```bash
docker compose ps
```

## Step 7: Retrieve the onion address

After Tor starts, it generates the onion service keys and writes the hostname to the hidden service directory. To read the hostname, run:

```bash
cat ~/tor/tor/hidden_service/hostname
```

The output will be your `.onion` address.

## Step 8: Open the site in Tor Browser

Copy the hostname and open it in Tor Browser. If everything is working correctly, you should see the test page you created in `html/index.html`.

At that point, the onion service is live.

## Updating the website

One of the nicest parts of this setup is how easy it is to update the site. Since the `html/` directory is mounted from the host into the Lighttpd container, you do not need to rebuild anything when you change a page.

Just edit the file locally, save it, and refresh the browser. For example:

```bash
nano ~/tor/html/index.html
```

## Useful commands

### Check container status

```bash
docker compose ps
```

### View logs

```bash
docker logs tor
docker logs lighttpd
```

### Restart the stack

```bash
docker compose down
docker compose up -d
```

### Stop the stack

```bash
docker compose down
```

## Troubleshooting

### 404 Not Found

If the onion address opens but returns a 404 page, the most common causes are that `index.html` is missing from `~/tor/html/`, Lighttpd is using the wrong document root, or `lighttpd.conf` was not mounted correctly. The first thing to check is whether the file actually exists:

```bash
ls -l ~/tor/html/index.html
```

Then make sure your `lighttpd.conf` still points to `/var/www/html`.

### Tor cannot open `/etc/tor/torrc`

This usually means the host path expected to be a file was accidentally created as a directory. Docker bind mounts are strict about path types, so a directory cannot be mounted over a file path.

Make sure `~/tor/tor/torrc` is really a file:

```bash
ls -l ~/tor/tor/torrc
```

### Mount error: file vs directory

If Docker says:

```text
Are you trying to mount a directory onto a file (or vice-versa)?
```

it means the source path on the host and the destination path in the container do not match in type. In practice, this often happens after an earlier failed attempt where a missing file path was created as a directory by mistake.

### Hidden service permissions

Tor is strict about the hidden service directory and may refuse to use it if permissions are too open. If needed, run:

```bash
chmod 700 ~/tor/tor/hidden_service
```

## Security notes

This is a minimal setup, but a few precautions still matter:

- Do not expose the Lighttpd container on host ports unless you intentionally want the site reachable outside Tor.
- Keep the `tor/hidden_service/` directory private, because it contains the onion service keys.
- Prefer static content unless you really need something more complex.
- If you want a more reproducible setup over time, consider pinning image versions instead of always using `latest`.

## Final notes

This project is intentionally small and practical. It is useful for simple onion sites, experiments, personal projects, internal publishing, or just learning how a Tor hidden service works without introducing unnecessary complexity.

Once it is running, the workflow is pleasantly simple: edit the files inside `html/`, refresh Tor Browser, and keep going.
