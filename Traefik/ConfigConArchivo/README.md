## Configurar traefik con archivo 
Primero reemplazamos los comandos con **traefik.yaml** y luego le indicamos que vercion de TLS usar con  **dynamic.yaml**, para verificar la seguridad usamos https://www.ssllabs.com/ssltest

## Apuntar a otro servidor 
Redirecciono fuera la red docker el Host(`server.nginx.app.test`) con el servicio "apache-service" a la url: "http://127.0.0.1" 
````
http:
  routers:
    apache:
      rule: "Host(`server.nginx.app.test`)"
      service: apache-service

  services:
    apache-service:   
      loadBalancer:
        servers:
          - url: "http://127.0.0.1"
````
