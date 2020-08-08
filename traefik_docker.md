On va d'abord installer traefik avec le fichier docker-compose.yml suivant
```
version: "3.8"
services:
  traefik:
    image: "traefik:latest"
    container_name: "traefik"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      # Ici : où est stocké mon certificat
      - "./chiffrage:/letsencrypt"
	  # Pour Docker pour windows
      - "//var/run/docker.sock:/var/run/docker.sock:ro"
      # Ici : fichier de config
      - "./config:/etc/traefik"
```
et un fichier qui configue traefik 
```
providers:
  docker: {}
  file:
    directory: "/etc/traefik"
    filename: "dynamic.yml"
    watch: true

log:
  level: DEBUG

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

certificatesResolvers:
  montlschallenge:
    acme:
      email:  "xianledatascientist@unmondesansxian.fr"
      storage:  "/letsencrypt/acme.json"
      tlsChallenge: {}

api:
  dashboard: true
```

Avec 
```bash
docker-compose up -d
```
on crée le container traefik.
```bash
afe7523f65f7        traefik:latest      "/entrypoint.sh trae…"   5 seconds ago       Up 5 seconds        0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:8080->8080/tcp   traefik
```

Ensuite pour le routing dynamique on crée un fichier de config
```
http:
  routers:
    api:
      entryPoints:
        - "web"
      rule: "Host(`localhost`)"
      service: api@internal
```

Et on relance le fichier compose afin de mettre à jour les configs.

En mettant l'adresse `http://traefik.localhost:8080/` ou simplement `http://localhost:8080`, le dashboard doit s'afficher avec 4 services réussis.

Pour que Traefik puisse retrouver automatiquement de nouveaux services ajoutés, on mets un réseau `traefik_reseau` dans le fichier Docker Compose et y joint le service `traefik`.

On va mettre en place maintenant un premier `whoami`.
```
  whoami_1:
    image: "containous/whoami"
    container_name: "whoami_1"
    networks:
      - traefik_reseau
    labels:
      - "traefik.backend=whoami"
      - "traefik.frontend.rule=Host:first.localhost"
```

Maintenant le service `whoami_1` est accesible à cette adresse. On y ajoute 


```
docker-compose --compatibility up -d
```