---
title: 🔀 Proxies and Certificates
tags:
  - Configuration
  - Proxy
  - Advanced
  - Traefik
  - Security
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';




## Allowing self-signed certificates
Some users may come across a barrier, where they're unable to receive a 200 response from the Ping widget for some apps, while using self-signed certificates or a local certificate authory.

:::note What's going on?

Homarr is trying to communicate to your apps via the integrations.
It usually doesn't matter if Homarr is running on ``http`` or ``https``.
Your apps have a self-signed certificate - Homarr will recognize that the certificate was signed by an unknown authority and requests will be blocked.

:::

Sadly, you can't add your self-signed certificates to Homarr yet.
But you can deactivate the rejection for unauthorized TLS requests.
Simply add the ``NODE_TLS_REJECT_UNAUTHORIZED`` environment variable and set it to ``0``.

<Tabs>
  <TabItem value="docker_run" label="Example with Docker Run" default>

```bash title=Terminal
docker run  \
  --name homarr \
  --restart unless-stopped \
  -p 7575:7575 \
  -v ./homarr/configs:/app/data/configs \
  -v ./homarr/icons:/app/public/icons \
  -e NODE_TLS_REJECT_UNAUTHORIZED=0 \
  -d ghcr.io/ajnart/homarr:latest
```

  </TabItem>
  <TabItem value="docker_compose" label="Example with Docker Compose">

```yml title=docker-compose.yml
version: '3'
#---------------------------------------------------------------------#
#                Homarr -  A homepage for your server.                #
#---------------------------------------------------------------------#
apps:
  homarr:
    container_name: homarr
    image: ghcr.io/ajnart/homarr:latest
    restart: unless-stopped
    volumes:
      - ./homarr/configs:/app/data/configs
      - ./homarr/icons:/app/public/icons
    environment:
      NODE_TLS_REJECT_UNAUTHORIZED: 0
    ports:
      - '7575:7575'
```

  </TabItem>
</Tabs>

---

## Securing Homarr with Traefik

Copying the configuration straight from the docker-compose file won't work if you are running Homarr behind Traefik, such as a Portainer setup, or docker-swarm. In that case, you should use the following slightly modified configuration:
```yaml
version: '3'
apps:
  homarr:
    container_name: homarr
    image: ghcr.io/ajnart/homarr:latest
    restart: unless-stopped
    volumes:
      - ./homarr/configs:/app/data/configs
      - ./homarr/icons:/app/public/icons
    environment:
      - BASE_URL=your.internal.dns.address.here.com
    networks:
      - proxy
    labels:
      traefik.enable: true
      traefik.http.routers.homarr.rule: Host(`your.internal.dns.address.here.com`)
      traefik.http.routers.homarr.entrypoints: websecure
      traefik.http.routers.homarr-secure.app: homarr
      traefik.http.apps.homarr.loadbalancer.server.port: 7575

networks:
  proxy:
    external: true
```

<br/>

A sample Traefik docker-compose.yml using Cloudflare for certificate generation that works with the configuration above would be:
```yaml
version: '3'

apps:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    ports:
      - 80:80
      - 443:443
    environment:
      - CF_API_EMAIL=yourcfemail@here.com
      - CF_DNS_API_TOKEN=long-token-from-cf

    command:
      - "--log.level=DEBUG"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.endpoint=unix:///var/run/docker.sock"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.web.http.redirections.entrypoint.permanent=true"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.websecure.http.tls.certResolver=cloudflare"
      - "--certificatesresolvers.cloudflare.acme.storage=acme.json"
      - "--certificatesResolvers.cloudflare.acme.email=yourcfemail@here.com"
      - "--certificatesResolvers.cloudflare.acme.dnsChallenge=true"
      - "--certificatesResolvers.cloudflare.acme.dnschallenge.provider=cloudflare"
      - "--certificatesResolvers.cloudflare.acme.dnschallenge.resolvers=1.1.1.1:53,1.0.0.1:53"
      - "--serversTransport.insecureSkipVerify=true" # Or proxmox gives an error 500 due to its own self-signed cert

    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/acme.json:/acme.json

networks:
  proxy:
    external: true
```


:::tip
Of particular note here is that both configurations explicitly define which network they are using, in this case "proxy", but it can be named anything. It just has to be the same across all apps for which Traefik is serving as a proxy. These are marked as external because the proxy network was manually created by running: `docker network create proxy` but this might be unnecessary depending on HOW exactly you are running Traefik. For example, if running [Traefik with Portainer](https://docs.portainer.io/advanced/reverse-proxy/traefik#deploying-in-a-docker-standalone-scenario), you can follow their official docs on how to set up Traefik and Portainer together, and you can just focus on the Homarr docker labels instead.
:::
