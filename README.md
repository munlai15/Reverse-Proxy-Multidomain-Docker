# Reverse Proxy Multi-domain on Docker
## Introducción
Un reverse proxy intercepta peticiones y las redirige al server apropiado.
La forma más sencilla que hay para gestionar esto es utilizar Nginx y Docker. En este ejemplo montaremos dos web sencillas más el Nginx reverse proxy, en diferentes contenedores Docker. Por lo que tendremos 3 contenedores: 2 webs y 1 proxy.

## Paso 1: Crear una web simple en un contenedor
1. Creamos el directorio para la primera web y entramos en él:  
``mkdir example1``  
``cd example1`` 
2. Creamos el docker-compose del primer contenedor que defina el service:  
``vim docker-compose.yml``  

```  
version: '2'  
services:  
 app:  
  image: nginx  
  volumes:  
   - .:/usr/share/nginx/html  
  ports:  
   - "80"    
 ```
Este docker-compose especifica que el servicio es una app que utiliza una imagen de nginx. Monta la raíz de example1 del Docker host a /usr/share/nginx/html/. Finalmente expone el servicio al puerto 80.  

3. Dentro del mismo directorio example1, creamos un index para la web:  
``vim index.html``
```
<!DOCTYPE html>
<html>
<head>
<title>Website 1</title>
</head>
<body>
<h1>Hola! Esta es la primera web</h1>
</body>
</html>
```

3. Creamos la imagen de todo el directorio:  
``docker-compose build``  
5. Arrancamos el contenedor:  
``docker-compose up -d``

## Paso 2: Crear una segunda web simple en un contenedor  
Básicamente es hacer una copia de lo que acabamos de realizar.
1. Creamos el directorio para la segunda web y entramos en él:  
``mkdir example2``  
``cd example2`` 
2. Creamos el docker-compose del segundo contenedor que defina el service:  
``vim docker-compose.yml``  

```  
version: '2'  
services:  
 app:  
  image: nginx  
  volumes:  
   - .:/usr/share/nginx/html  
  ports:  
   - "80"    
 ```
Este docker-compose especifica que el servicio es una app que utiliza una imagen de nginx. Monta la raíz de example1 del Docker host a /usr/share/nginx/html/. Finalmente expone el servicio al puerto 80.  

3. Dentro del mismo directorio example1, creamos un index para la web:  
``vim index.html``
```
<!DOCTYPE html>
<html>
<head>
<title>Website 2</title>
</head>
<body>
<h1>Hola! Esta es la segunda web.</h1>
</body>
</html>
```

3. Creamos la imagen de todo el directorio:  
``docker-compose build``  
4. Arrancamos el contenedor:  
``docker-compose up -d``  

## Paso 3: Listar contenedores
Hacemos una pausa para revisar que estén en correcto funcionamiento los dos contenedores que acabamos de crear:  
``docker ps -a``

## Paso 4: Montamos el Reverse Proxy Multidomain  
1. Creamos y accedemos al directorio del proxy:  
``mkdir proxy``  
``cd proxy``  
2. Configuramos el Dockerfile (instrucciones específicas a la hora de crear una imagen para un contenedor Docker):  
``vim Dockerfile``  
```
FROM nginx

COPY ./default.conf /etc/nginx/conf.d/default.conf

COPY ./backend-not-found.html /var/www/html/backend-not-found.html

COPY ./includes/ /etc/nginx/includes/

COPY ./ssl/ /etc/ssl/certs/nginx/
```  

Este Dockerfile se basa en la imagen nginx (FROM). También copia los ficheros necesarios que han de haber desde la máquina local (COPY).
- El fichero de configuración predeterminado del proxy
- Un HTML error response
- Configuraciones y certificados del proxy y SSL  

3. Configurar el fichero de backend-not-found:  
``vim backend-not-found.html``  
```
<html>
<head><title>Proxy Backend Not Found</title></head>
<body>
<h2>Proxy Backend Not Found</h2>
</body>
</html>
```  
4. Configurar el fichero default.conf:  
``vim default.conf``  
```
# web service1 config.
server {
listen 80;
listen 443 ssl http2;
server_name example1.test;

# Path for SSL config/key/certificate
ssl_certificate /etc/ssl/certs/nginx/example1.crt;
ssl_certificate_key /etc/ssl/certs/nginx/example1.key;
include /etc/nginx/includes/ssl.conf;

location / {
include /etc/nginx/includes/proxy.conf;
proxy_pass http://example_app_1;
}

access_log off;
error_log /var/log/nginx/error.log error;
}

# web service2 config.
server {
listen 80;
listen 443 ssl http2;
server_name example2.test;

# Path for SSL config/key/certificate
ssl_certificate /etc/ssl/certs/nginx/example2.crt;
ssl_certificate_key /etc/ssl/certs/nginx/example2.key;
include /etc/nginx/includes/ssl.conf;

location / {
include /etc/nginx/includes/proxy.conf;
proxy_pass http://example2_app_1;
}

access_log off;
error_log /var/log/nginx/error.log error;
}

# Default
server {
listen 80 default_server;

server_name _;
root /var/www/html;

charset UTF-8;

error_page 404 /backend-not-found.html;
location = /backend-not-found.html {
allow all;
}
location / {
return 404;
}

access_log off;
log_not_found off;
error_log /var/log/nginx/error.log error;
}
```  
La configuración consiste de dos servicios web (example1.test y example2.test) Ambos servicios atienden al puerto 80 y dirige el Nginx al certificado SSL apropiado.  

5. Configurar el fichero docker-compose.yml:  
``vim docker-compose.yml``  
```
version: '2'
services:
  proxy:
    build: ./
      networks:
        - example1
        - example2
      ports:
        - 80:80
        - 443:443

networks:
  example1:
    external:
      name: example1_default
   example2:
     external:
       name: example2_default
```  

En este fichero conectamos el proxy y las redes externas (example1 y example2). También los puertos 80/443 del proxy responden a los puertos 80/443 del Docker host.  

6. Generar las Keys y Certificados  
Creamos un subdirectorio (ssl) dentro del directorio actual (proxy) y accedemos:  
``mkdir ssl``  
``cd ssl``  

Creamos los ficheros necesarios:  
``touch example1.crt``  
``touch example1.key``  
``touch example2.crt``  
``touch example2.key``  

Una vez creados, generamos las keys y certificados para nuestras webs con OpenSSL:  
``openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout example1.key -out example1.crt``  
``openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout example2.key -out example2.crt``  

7. Editamos la configuración del proxy y del SSL:  
Salimos del subdirectorio ssl para volver al directorio proxy y creamos otro subdirectorio con el nombre de includes y accedemos a él:  
`` cd ..``  
``mkdir includes``  
``cd includes``  

Creamos el archivo proxy.conf:  
``vim proxy.conf``  
```
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_buffering off;
proxy_request_buffering off;
proxy_http_version 1.1;
proxy_intercept_errors on;
```  

Creamos el archivo ssl.conf:  
``vim ssl.conf``  
```
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-
ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-
SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-
GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-
AES128-SHAECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-
SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:
DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-
DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:
AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-
CBC3-SHA:!DSS';
ssl_prefer_server_ciphers on;
```  

8. Editamos el fichero de hosts:  
Tenemos que saber las IP's de nuestros contenedores donde tenemos los dos servicios web. Para ello hacemos:  
``docker inspect <NombreOrIDdelContenedor> | grep '"IPAddress"'``  
Una vez tenemos las IP's de ambos, las añadimos al fichero /etc/hosts para que nos haga la resolución de nombres:  
``sudo vim /etc/hosts``  
```
172.19.0.2	example1.test
172.20.0.2	example2.test
```  

## Paso 5: Encender el Reverse Proxy Multidomain  
Estando en el subdirectorio proxy, creamos la imagen y ejecutamos el contenedor:  
``docker-compose build``  
``docker-compose up -d``  

Volvemos a verificar los contenedores listándolos (esta vez debería de haber 3, los 2 webn y el proxy).  
``docker ps -a``  

## Paso 6: Comprobar que funciona  
Nos vamos a cualquier navegador y accedemos tanto a example1.test como a example2.test para comprobar que las dos webs están funcionando.  
Friki tip: Si nos da pereza abrir el navegador podemos hacer lo siguiente:  
``curl example1.test``  
Lo cual nos debería de dar el código del html.



