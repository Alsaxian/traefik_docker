Ceci est une démo pour répondre à l'exercice posé par On Rewind.  
Dans cette démo, on installera par docker container un reverse proxy traefik puis 3 containers whoami, dont deux réponderont à `first.localhost` par répartition de charge, un répondera à `second.localhost`.  
La démo a été faite sous Windows car je n'ai pas d'ordi Linux à la main lol. Donc des nuances dans les commandes avec Linux à prévoir.   
  
On va d'abord installer traefik avec le fichier `docker-compose.yml` suivant
```yml
version: "3.8"
services:
  traefik:
    image: "traefik:latest"
    container_name: "traefik"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
	  # Pour Docker pour Windows c'est double slash au lieu d'un seul
      - "//var/run/docker.sock:/var/run/docker.sock:ro"
      # Ici : fichiers de config partagé
      - "./config:/etc/traefik"
```
  
Et un fichier qui configue traefik nommé `traefik.yml`
```yml
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

api:
  dashboard: true
  insecure: true
```
  
Avec 
```bash
docker-compose up -d
```
on crée le container traefik.
```bash
afe7523f65f7        traefik:latest      "/entrypoint.sh trae…"   5 seconds ago       Up 5 seconds        0.0.0.0:80->80/tcp, 0.0.0.0:8080->8080/tcp   traefik
```

Ensuite pour le routing dynamique on crée un fichier de config `dynamic.yml`
```yml
http:
  routers:
    api:
      entryPoints:
        - "web"
      rule: "Host(`localhost`)"
      service: api@internal
```

Et on relance le fichier Docker Compose afin de mettre à jour les configs.

En mettant l'adresse `http://traefik.localhost:8080/` ou simplement `http://localhost:8080`, le dashboard doit s'afficher avec 4 services réussis.

Pour que Traefik puisse retrouver automatiquement de nouveaux services ajoutés, on mets un réseau `traefik_reseau` dans le fichier Docker Compose et y joint le service `traefik`.

On va mettre en place maintenant un premier `whoami`
```yml
  whoami_1:
    image: "containous/whoami"
    container_name: "whoami_1"
    networks:
      - traefik_reseau
    labels:
      - "traefik.backend=whoami"
      - "traefik.frontend.rule=Host:first.localhost"
```
Et modifier le fichier `dynamic.yml`
```yml
    whoami_1:
      entryPoints:
        - "web"
      rule: "Host(`first.localhost`)"
      service: whoami-1-traefik-docker@docker # Dépend de comment s'appelle le répertoire dans lequel on a éxécuté le fichier Docker Compose.
```
Le service `whoami-1-traefik-docker` est accesible à cette adresse. On y met un 2-ème whoami qui répond au même domaine en ajoutant simplement un nombre de duplicata  
```yml
    deploy:
      mode: replicated
      replicas: 2
```
Dans le fichier Docker Compose. Maintenant quand on actualise le navigateur à "first.localhost", les deux whoami répondent à la requête de façon alternée. A noter que pour cette version de Docker Compose, la mise à jour du nombre de duplicata ne sera que prise en compte avec la commande légèrement différente comme suit
```bash
docker-compose --compatibility up -d
```

Finalement on peux mettre un 3-ème whoami qui répond à un autre domaine
```yml
  whoami_2:
    image: "containous/whoami"
    container_name: "whoami_2"
    networks:
      - traefik_reseau
```
Et préciser son domaine à qui répondre dans `dynamic.yml`de faèon similaire comme avant
```yml
    whoami_2:
      entryPoints:
        - "web"
      rule: "Host(`second.localhost`)"
      service: whoami-2-traefik-docker@docker
```

Maintenant on a 4 containers dans le même réseau, un de traefik qui sert comme un reverse proxy, trois whoami dont deux répondent à `first.localhost` et un répond à `second.localhost`. L'ensemble des fichiers `yml` sont à retrouver dans ce projet.

