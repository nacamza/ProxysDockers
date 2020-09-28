# Traefik (v2.2)
Vamos a realizar una implementación de una aplicación que contiene varios Docker y vamos a utilizar un proxy inverso Traefik para obtener HTPPS

Vamos a crear 4 contenedores:
- traefik: es el encargado de manejar el proxy inverso
- frontend: es un 
- backend: es una aplicación escuchando en el puerto 4000
- apiexpress: es una api escuchando en el puerto 8000

## Configuración Traefik
Vamos a ver la configuración de traefik para nuestra aplicación 
````
  traefik:
    image: "traefik:v2.2"
    container_name: "traefik"
    command:
      #- "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.back.address=:4000"
      - "--entrypoints.api.address=:8000"
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      #- "--certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
      - "--certificatesresolvers.myresolver.acme.email=nahuelcano@gmail.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
      - "8000:8000"
      - "8080:8080"
      - "4000:4000"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
````
vamos a explicar las partes importantes es esta configuración
### HTTPS automático con Let's Encrypt
Vamos a habilitar la encriptación y vamos a dirigir el tráfico HTTP del puerto 80 al 443 con HTTPS 
````
    command:
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myresolver.acme.email=nahuelcano@gmail.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
````
Montamos un volumen donde se va a guardar los certificados
````
    volumes:
      - "./letsencrypt:/letsencrypt"
````
### Entrypoints
Los entrypoints son utilizados para indicarle a traefik que Docker tiene q acceder según el puerto. De esta manera podemos redireccionar varios Docker con un mismo DNS

Vamos a configurar traefik para que escuche en los puertos 80, 443, 8000 y 4000

Lo primero que tenemos q hacer es exponer estos puertos en el servicio traefik
````
  traefik:
    image: "traefik:v2.2"
    container_name: "traefik"    
    ports:
      - "80:80"
      - "443:443"
      - "8000:8000"
      - "8080:8080"
      - "4000:4000"
````
Luego le asignamos un entrypoints a cada puerto con los siguientes comandos
````
    command:
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.back.address=:4000"
      - "--entrypoints.api.address=:8000"
````
Donde:
- Puerto 80: lo llamamos web
- Puerto 443: lo llamamos websecure
- Puerto 8000: lo llamamos back
- Puerto 4000: lo llamamos api
Ya tenemos configurado los puertos en los que escucha traefik, ahora le tenemos que indicar a que Docker tiene que redirigir el tráfico. Para esto se utilizan **labels** en cada Docker.
## Dockers 
Vamos a explicar cómo dirigimos el tráfico al backend
````
  backend:
    image: dockerback
    expose: 
      - "4000"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.backend.rule=Host(`docker1.viziongps.net`)"
      - "traefik.http.routers.backend.entrypoints=back"
      - "traefik.http.routers.backend.tls.certresolver=myresolver"
 ````     
 primero debemos indicar si el Docker va a estar manejado por traefik
 ````
     labels:
      - "traefik.enable=true"
 ````
luego le indicamos con que DNS y puerto se acede al Docker
````
     - "traefik.http.routers.apiexpress.rule=Host(`docker1.viziongps.net`)"
     - "traefik.http.routers.apiexpress.entrypoints=api"
````
también le indicamos que certificado utilizar
````
      - "traefik.http.routers.apiexpress.tls.certresolver=myresolver"
````
y por último le indicamos que puerto está exponiendo el Docker
````
      - "traefik.http.services.apiexpress.loadbalancer.server.port=8000"
````  
    
