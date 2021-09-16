# docker-multi-domain
Hay que añadir estas dos líneas al /etc/hosts:

```
127.0.0.1	hosta
127.0.0.1	hostb
```

Una vez tenemos todo esto en un mismo directorio hacemos build para crear la imagen y up para hacerla funcionar:

```
docker-compose build
docker-compose up
```
