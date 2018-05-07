### Examen 2
**Universidad ICESI**  
**Curso:** Sistemas Distribuidos  
**Nombre:** Carolina Zúñiga Ospina  
**Código:**  A00315292

**Url Repositorio:**  https://github.com/carozuniga/sd-exam2/tree/master/A00315292


### Desarrollo 

Para el desarrollo del parcial se tomó como referencia la guía que se encuentra en esta url: **http://www.littlebigextra.com/using-kibana-and-elasticsearch-for-log-analysis-with-fluentd-on-docker-swarm/**
En está guía crean el servicio web whoami que responde a una solicitud con la ip del host. Además, se emplean los servicios de fluentd, elasticsearch y kibana. Elastic Search es un motor de búsqueda de código abierto basado en Apache Lucene. Junto con Kibana, que es una herramienta de visualización, Elasticsearch se puede utilizar para análisis en tiempo real. 

**Diagrama UML**

![][7]

**Comandos y archivos empleados**

- Primero se debe inicializar swarm con el nodo manager 
```
docker swarm init
```
![][1]

- Una vez inicializado se crea el token para poder agregar los nodos workers
```
docker swarm join --token <token> IP_manager:2377
```
- Crear el archivo de configuración para fluentd **fluent.conf**
```json
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
<match tutum>
  @type copy
   <store>
    @type file
    path /fluentd/log/tutum.*.log
    time_slice_format %Y%m%d
    time_slice_wait 10m
    time_format %Y%m%dT%H%M%S%z
    compress gzip
    utc
    format json
  </store>
  <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    logstash_format true
    logstash_prefix logstash
    logstash_dateformat %Y%m%d
    include_tag_key true
    tag_key @log_name
    flush_interval 1s
  </store>
</match>
<match visualizer>
  @type copy
   <store>
    @type file
    path /fluentd/log/visualizer.*.log
    time_slice_format %Y%m%d
    time_slice_wait 10m
    time_format %Y%m%dT%H%M%S%z
    compress gzip
    utc
    format json
  </store>
    <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    logstash_format true
    logstash_prefix logstash
    logstash_dateformat %Y%m%d
    include_tag_key true
    tag_key @log_name
    flush_interval 1s
  </store>
</match>

```
- Crear una imagen para fluentd con un dockerfile, en el que se instala el plugin de elastic search 
```Docker
 # fluentd/Dockerfile
FROM fluent/fluentd:v0.12-debian
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.2"]
RUN rm /fluentd/etc/fluent.conf
COPY ./conf/fluent.conf /fluentd/etc 
```
 - Posterior, se debe iniciar sesión en Docker para poder subir dicha imagen creada a un repositorio de docker hub con los siguientes comandos:
```bash
docker build -t carozuniga/parcial2:latest 
docker push carozuniga/parcial2
```
 ![][2]
 ![][3]
 - Finalmente se necesita el docker compose que contiene los servicios whoami con varias replicas, además de elastic search, kibana y fluentd. Este archivo se debe llamar docker-swarm.yml

```yaml
version: "3"
 
services:
       
  whoami:
    image: tutum/hello-world
    networks:
      - net
    ports:
      - "80:80"
    logging:
      driver: "fluentd"# Logging Driver
      options:
        tag: tutum    # TAG 
    deploy:
      restart_policy:
           condition: on-failure
           delay: 20s
           max_attempts: 3
           window: 120s
      mode: replicated
      replicas: 3
      placement:
        constraints: [node.role == worker]
      update_config:
        delay: 2s
 
  vizualizer:
      image: dockersamples/visualizer
      volumes:
         - /var/run/docker.sock:/var/run/docker.sock
      ports:
        - "8080:8080"
      networks:
        - net
      logging:
        driver: "fluentd"
        options:
         tag: visualizer   #TAG 
      deploy:
          restart_policy:
             condition: on-failure
             delay: 20s
             max_attempts: 3
             window: 120s
          mode: replicated # one container per manager node
          replicas: 1
          update_config:
            delay: 2s
          placement:
             constraints: [node.role == manager]
 
        
  fluentd:
    image: carozuniga/parcial2
    volumes:
      - ./Logs:/fluentd/log
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    networks:
      - net
    deploy:
      restart_policy:
           condition: on-failure
           delay: 20s
           max_attempts: 3
           window: 120s
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      update_config:
        delay: 2s
 
  elasticsearch:
    image: elasticsearch
    expose:
      - 9200
    ports:
      - "9200:9200"
    networks:
      - net
    environment:
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    logging:
        driver: "json-file"
        options:
          max-size: 10M
          max-file: 1  
    deploy:
      restart_policy:
        condition: on-failure
        delay: 20s
        max_attempts: 3
        window: 120s
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      update_config:
        delay: 2s
      resources:
        limits:
          memory: 1000M
    volumes:
      - ./esdata:/usr/share/elasticsearch/data    
      
  kibana:
    image: kibana
    ports:
      - "5601:5601"
    networks:
      - net
    logging:
        driver: "json-file"
        options:
           max-size: 10M
           max-file: 1        
    deploy:
      restart_policy:
        condition: on-failure
        delay: 20s
        max_attempts: 3
        window: 120s
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      update_config:
        delay: 2s  
 
networks:
  net:

```
- Para desplegar en swarm el archivo anterior se ejecuta el siguiente comando:

```bash
docker stack deploy -c docker-swarm.yml test
```
 ![][4]

- Acceder a kibana para crear y descubrir los logs (es necesario crear el directorio esdata para almacenar los logs)

![][5]

- El servicio web comentado anteriormente se puede visualizar como se muestra en la siguiente imagen

![][6]

**Dificultades encontradas**

La principal dificultad se encontró al momento de desplegar los servicios en el swarm ya que solo desplegaba unos pero no todos. Esto se daba porque docker estaba desactualizado, para esto se tuvo que actualizar docker a la última versión. 

### Referencias
- http://www.littlebigextra.com/using-kibana-and-elasticsearch-for-log-analysis-with-fluentd-on-docker-swarm/
- https://docs.docker.com/engine/swarm/stack-deploy/

[1]: images/init.png
[2]: images/build.png
[3]: images/push.png
[4]: images/deploy.png
[5]: images/kibana.PNG
[6]: images/whoami.PNG
[7]: images/uml.PNG
