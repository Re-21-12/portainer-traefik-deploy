# portainer-traefik-deploy

version: '3.9'

services:
  ## 🛠️ PORTAINER SERVER (ACCESO VIA TRAEFIK)
  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - "9443:9443  
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:rw 
      - portainer_data:/data
    networks:
      - traefik-public
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      labels:
        - traefik.enable=true
        - traefik.http.services.portainer.loadbalancer.server.port=9000 
        # 1. ROUTER HTTP
        - traefik.http.routers.portainer-web.rule=Host(`portainer.corazondeseda.lat`)
        - traefik.http.routers.portainer-web.entrypoints=web
        # 2. ROUTER HTTPS
        - traefik.http.routers.portainer-secure.rule=Host(`portainer.corazondeseda.lat`)
        - traefik.http.routers.portainer-secure.entrypoints=websecure
        - traefik.http.routers.portainer-secure.tls=true
        - traefik.http.routers.portainer-secure.tls.certresolver=le
        - traefik.http.routers.portainer-secure.service=portainer

  ## 🛡️ SERVICIO TRAEFIK (PROXY INVERSO Y SSL)
  traefik:
    image: traefik:v2.10
    command:
      - --api.dashboard=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --providers.docker.network=traefik-public
      - --providers.docker.swarmMode=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --entrypoints.webdashboard.address=:8081
      - --certificatesresolvers.le.acme.email=victora.caas@gmail.com
      - --certificatesresolvers.le.acme.storage=/etc/traefik/acme.json
      - --certificatesresolvers.le.acme.tlschallenge=true
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
    ports:
      - "80:80" 
      - "443:443" 
      - "8081:8081"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_data:/etc/traefik 
    networks:
      - traefik-public
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      labels:
        - traefik.enable=true 
        - traefik.http.services.api.loadbalancer.server.port=8080 
        
        # Dashboard de Traefik - Acceso seguro y Path Stripping
        - traefik.http.routers.dashboard.rule=Host(`traefik.corazondeseda.lat`) # ⬅️ REGLA SIMPLIFICADA
        - traefik.http.routers.dashboard.entrypoints=websecure
        - traefik.http.routers.dashboard.service=api@internal
        - traefik.http.routers.dashboard.tls=true
        - traefik.http.routers.dashboard.tls.certresolver=le

volumes:
  portainer_data: {}
  traefik_data: {} 

networks:
  traefik-public:
    driver: overlay
    external: true