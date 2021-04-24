# Traefik (v2.2)
Vamos a realizar una implementación de una aplicación que contiene varios Docker y vamos a utilizar un proxy inverso Traefik para obtener HTPPS

Vamos a crear 4 contenedores:
- traefik: es el encargado de manejar el proxy inverso
- frontend: es un 
- backend: es una aplicación escuchando en el puerto 4000
- api: es una api escuchando en el puerto 8000
- daemon: es una api que que no expone ningún puerto

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
      
      # Agregamos configuracion dinamica
      - "--providers.file.directory=/etc/traefik/dynamic" 
      
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.web.http.redirections.entrypoint.permanent=true" 
      
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
      # Montamos archivo de configuracion dinamica
      - ./conf/dynamic.yaml:/etc/traefik/dynamic/dynamic.yaml 
    labels:
      # compress whith gzip
      - "traefik.http.routers.traefik.middlewares=traefik-compress"
      - "traefik.http.middlewares.traefik-compress.compress=true"   
````
vamos a explicar las partes importantes es esta configuración
### HTTPS automático con Let's Encrypt
Vamos a habilitar la encriptación de nuestros servicios y vamos a dirigir el tráfico HTTP del puerto 80 al 443 con HTTPS 
````
    command:
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.web.http.redirections.entrypoint.permanent=true" 
      
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myresolver.acme.email=mail@gmail.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
````
Montamos un volumen donde se va a guardar los certificados y el socket Docker para que traefik obtenga información de los servicios. 
````
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./letsencrypt:/letsencrypt"
````
Si hay algún inconveniente con los certificados se envía un mail a:
````
      - "--certificatesresolvers.myresolver.acme.email=mail@gmail.com"
````
### Entrypoints
Los entrypoints son utilizados para indicarle a traefik a que Docker tiene que direccionar el tráfico teniendo en cuenta el puerto por el que entra la consulta. De esta manera podemos redireccionar varios Docker con un mismo DNS.

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
Ya tenemos configurado los puertos en los que escucha traefik, ahora le tenemos que indicar a que Docker tiene que redirigir el tráfico. Para esto se utilizan las **labels** en cada Docker.
## Dockers 
Vamos a explicar cómo dirigimos el tráfico al backend
````
  backend:
    image: dockerback
    expose: 
      - "4000"
    labels:
      - "traefik.enable=true"
      
      # Entrada por el puerto :4000
      - "traefik.http.routers.backend.rule=Host(`midominio.com`)"
      - "traefik.http.routers.backend.entrypoints=back"
      - "traefik.http.routers.backend.tls.certresolver=myresolver"
      
      # Entrada por el puerto :443 y PathPrefix /back
      - "traefik.http.routers.backend-path.rule=Host(`midominio.com`) && PathPrefix(`/back`)"
      - "traefik.http.routers.backend-path.entrypoints=websecure"
      - "traefik.http.routers.backend-path.tls.certresolver=myresolver"  

      # Podemos eliminar el path /back mediante prefix
      - "traefik.http.middlewares.delete_path.stripprefix.prefixes=true" 
      - "traefik.http.routers.backend-path.middlewares=delete_path"     
          
      # Enable gzip compression
      - "traefik.http.middlewares.backend_compress.compress=true" 
      - "traefik.http.routers.backend.middlewares=backend_compress"     
 ````     
 primero debemos indicar si el Docker va a estar manejado por traefik
 ````
     labels:
      - "traefik.enable=true"
 ````
luego le indicamos con que DNS y puerto se acede al Docker
````
     - "traefik.http.routers.backend.rule=Host(`midominio.com `)"
     - "traefik.http.routers.backend.entrypoints=api"
````
también le indicamos que certificado utilizar
````
      - "traefik.http.routers.backend.tls.certresolver=myresolver"
````
le indicamos que puerto está exponiendo el Docker
````
      - "traefik.http.services.backend.loadbalancer.server.port=8000"
````  
le indicamos si tiene que comprimir la respuesta con GZIP
````
      # Enable gzip compression
      # Configuramos el middlewar backend_compress
      - "traefik.http.middlewares.backend_compress.compress=true" 
      # le indicamos que aplique el middlewar backend_compress a routers backend
      - "traefik.http.routers.backend.middlewares=backend_compress"
````
Las respuestas se comprimen cuando:
- El cuerpo de la respuesta es mayor que 1400bytes.
- El ``Accept-Encodin`` gencabezado de la solicitud contiene ``gzip``.
- La respuesta aún no está comprimida, es decir, el Content-Encodingencabezado de respuesta aún no está configurado.

y por ultimo, podemos generar otro router, cambiando al path o puerto con el que se accede
```
      # Entrada por el puerto :443 y PathPrefix /back
      - "traefik.http.routers.backend-path.rule=Host(`midominio.com`) && PathPrefix(`/back`)"
      - "traefik.http.routers.backend-path.entrypoints=websecure"
      - "traefik.http.routers.backend-path.tls.certresolver=myresolver"  
```
donde:
- Path: se usa cuando la url es fija
- PathPrefix: se usa cuando despues del path se puede direccionar
 
Con estas **labels** queda configurado el backend, de esta manera cuando accedamos a **https://midominio.com:4000/** vamos a acceder al backend con certificados https.

### Si queremos seguridad (User y Pass) agregamos
````
      ## Seguridad
      - "traefik.http.routers.backend.middlewares=auth"
      ## Generar user/pass "https://hostingcanada.org/htpasswd-generator/"
      - "traefik.http.middlewares.auth.basicauth.users=ncano:{SHA}jkUa6APiSmiQwgp/s9BTu4h04F0="
````
con esto nos pide user y pass para acceder
### [Configuracion Abanzada](https://blog.thesparktree.com/traefik-advanced-config)
## Comando útiles de docker compose
Dejo unos comandos útiles de Docker compose, estos comando tienen que ejecutarse en el directorio en donde se encuentra el archivo Docker-compose.yml 
### Ejecutar docker compose 
````
docker-compose up -d
````
### Detener docker compose
````
docker-compose up -d
````
### Ver logs de todos los servicios 
````
docker-compose logs
````
### Descargar la última versión de las imágenes de los servicios
````
docker-compose pull
````
### Para ejecutar un comando en un servicio
````
docke
    
