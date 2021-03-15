#  Nginx-proxy
Vamos a explicar brevemente como configurar el nginx-proxy de jwilder con letsencrypt-nginx-proxy-companion de jrcs. Vamos a implementar una aplicación que va a tener un nginx en el puerto 80 y la api grafena en el 3000
````
version: '3.1'
	services:

  nginx-proxy:
	    image: jwilder/nginx-proxy
	    restart: always
	    ports:
	      - "443:443"  
	      - "80:80"
	      - "3000:3000"
	    volumes:
	      - /var/run/docker.sock:/tmp/docker.sock:ro
	      - certs:/etc/nginx/certs:ro
	      - vhostd:/etc/nginx/vhost.d
	      - html:/usr/share/nginx/html
	      
	      #Certificados pagos
	      - config-nginx.conf:/etc/nginx/conf.d/config-nginx.conf
	      - certs:/etc/nginx/certs
	    labels:
	      - com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy
	

	  letsencrypt:
	    image: jrcs/letsencrypt-nginx-proxy-companion
	    restart: always
	    environment:
	      - NGINX_PROXY_CONTAINER=nginx-proxy
	    volumes:
	      - certs:/etc/nginx/certs:rw
	      - vhostd:/etc/nginx/vhost.d
	      - html:/usr/share/nginx/html
	      - /var/run/docker.sock:/var/run/docker.sock:ro
````
En estos Dockers esta lo necesario para que función el proxy inverso con letsencrypt

Si queremos pasar certificados pagos, tenemos que tener un archivo de configuracion ``config-nginx.conf``
````
	    volumes:
	      - config-nginx.conf:/etc/nginx/conf.d/config-nginx.conf
````
y mantamos los certificados en 
````
	      - /certs:/etc/nginx/certs
```` 
### Configuracion dockers
````
  nginx1:
	    image: nacamza/githubactions
	    expose:
	      - "80"
	    environment:
	      - VIRTUAL_HOST=docker1.viziongps.net
	      - LETSENCRYPT_HOST=docker1.viziongps.net
	      - LETSENCRYPT_EMAIL=ncano@globalis-sa.com        

````
Para dirigir el tráfico a los Dockers necesitamos indicarle las siguientes env
````
	      - VIRTUAL_HOST=docker1.viziongps.net
	      - LETSENCRYPT_HOST=docker1.viziongps.net
	      - LETSENCRYPT_EMAIL=ncano@globalis-sa.com        
````
En donde:
- VIRTUAL_HOST: es el DNS utilizado
- LETSENCRYPT_HOST: es el puerto que va a utilizar el Docker 
- LETSENCRYPT_EMAIL: es el mail que va a utilizar si hay un problema con la licencia
