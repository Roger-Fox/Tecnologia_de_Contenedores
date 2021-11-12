# Tecnologia de Contenedores  
## Proyecto de Tecnologia de Contenedores - Hot Warm and Cold
## Integrantes: James Allan Weisnner - Verbo Julian Camacho  

## Descripción General  
La finalidad del proyecto es construir una arquitectura con varios contenedores en Docker que sea capaz de realizar el ciclo de vida Hot Warm y Cold de datos por medio de una arquitectura ELK (Elasticsearch Logstach y Kibana), esto se logra mediante un "cluster" que agrupa 3 contenedores para Elasticsearch, uno para Logstach y otro para Kibana, todos interconectados por medio de una red, y agrupados por el archivo docker compose *"Proyecto-compose.yml"*.  

Con una arquitectura ELK operativa, solo quedaría el paso de poblarla con datos provenientes de otro contenedor docker, en este caso se ha considerado hacer un programa sencillo en Python, cuya salida sean lecturas regulares del CPU del computador. Al haber la suficiente cantidad de datos podremos apreciar de forma práctica como cada cierta cantidad de tiempo los documentos creados por el contenedor con el programa de Python son procesados pasando primero por la fase hot, luego a la fase warm, y finalmente a la fase de congelamiento, donde se realizará un almacenamiento masivo con pocos recursos dedicados de datos que pasan a ser "caducos", ya sea por una configuración de límite de tiempo o peso por fase. *Esta decisión se ve más claramente más adelante cuando definimos el comando para crear las políticas del ciclo de vida de los datos*  

## Bitácora del proyecto  
Primero fue necesario instalar la imagen de elasticsearch, la cual puede conseguirse en la pagina oficial y en el portal Docker hub.  

>docker pull docker.elastic.co/elasticsearch/elasticsearch:7.15.  

Al haber descargado la imagen, construiremos un contenedor apartir de dicha imagen con el comando  

>docker run --name elasticsearch -d -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.15.1

En este comando estamos usando el puerto 9200 del cliente y del contenedor, y usando un solo nodo de elasticsearch. En este caso lo utilizamos únicamente para probar que Elasticsearch esta funcionando correctamente  

A continuación, utilizamos parte del repositorio estudiado en clase, esto con el fin de tener una base para ejecutar el comando Docker compose, que iniciaría una arquitectura ELK (Elasticsearch, Logstash, Kibana) que será usada a detalle más adelante  
>docker-compose -f Proyecto-compose.yml up


![recursos](https://github.com/Roger-Fox/Tecnologia_de_Contenedores/blob/main/pictures/recursos.png)

Tras asignar mas recursos a la máquina virtual Linux, fue posible ejecutar el comando docker-compose exitosamente.  

![running_cluster](https://github.com/Roger-Fox/Tecnologia_de_Contenedores/blob/main/pictures/cluster%20running.png)

Ejecutamos el comando docker ps, y nos damos cuenta de los tres nodos de frio, tibio y caliente, están corriendo adecuadamente junto a los contenedores de Logstash y Kibana.  

Con toda nuestra arquitectura ELK funcionando, empezamos a escribir el comando que definiría las políticas de ciclo de vida de los datos. Este paso tomo un tiempo considerable, pues no comprendíamos bien como funcionaban dichos comandos, finalmente utilizamos una política que representa bien el ciclo de vida Hot, Warm y Cold. Este puede ser visto en el repositorio del proyecto bajo el nombre “ILM Command”, el comando fue:  

>curl -X PUT "localhost:9200/_ilm/policy/hot_warm_cold_project?pretty" -H 'Content-Type: application/json' -d' 
>{
>"policy": {
>"phases": {
>"hot": {
>"actions": {
>"rollover": {
>"max_size":"940k",
>"max_age":"4m"
>},
>"set_priority": {
>"priority": 50
>}
>}
>},
>"warm": {
>"min_age": "5m",
>"actions": {
>"forcemerge": {
>"max_num_segments": 1
>},
>"shrink": {
>"number_of_shards": 1
>},
>"allocate": {
>"require": {
>"data": "warm"
>}
>},
>"set_priority": {
>"priority": 25
>}
>}
>},
>"cold": {
>"min_age": "8m",
>"actions": {
>"set_priority": {
>"priority": 0
>},
>"freeze": {},
>"allocate": {
>"require": {
>"data": "cold"
>}
>}
>}
>},
>"delete": {
>"min_age": "1d",
>"actions": {
>"delete": {}
>}
>}
>}
>}
>}
>'*    

Al ingresar al endpoint de elasticsearch se puede ver que la política fue correctamente creada. La política se llama “hot_warm_cold_project”.  

![politics_list](https://github.com/Roger-Fox/Tecnologia_de_Contenedores/blob/main/pictures/Captura%20de%20pantalla%20de%202021-11-07%2021-51-08.png)
Al haber creado la política procedimos a crear el template del índice mediante el comando:  

>curl -X PUT "localhost:9200/_index_template/hot_warm_cold_template?pretty" -H 'Content-Type: application/json' -d'
>{
>"index_patterns": ["hotWarmCold-*"],
>"template": {
>"settings": {
>"number_of_shards": 1,
>"number_of_replicas": 0,
>"index.lifecycle.name": "hot_warm_cold_project",
>"index.lifecycle.rollover_alias": "hotWarmCold",
>"index.mapping.total_fields.limit":"2000"
>}
>}
>}'


Teniendo en cuenta que no es posible que Logstash haga uso de Data streams, hicimos cambios en la configuración de pipeline de Logstash, para lo cual detuvimos el proceso de todos los clusters de la arquitectura. Al terminar esta modificación, el archivo logstash.conf quedo así:  

>input {
>	beats {
>		port => 5044
>	}
>
>	tcp {
>		port => 5000
>	}
>}
>
>'## Add your filters / logstash plugins configuration here'
>
>output {
>	elasticsearch {
>		hosts => "elastic01:9200"
>		ilm_enabled => true
>		ilm_rollover_alias => "hotWarmCold"
>		ilm_policy => "hot_warm_cold_project"
>		ecs_compatibility => disabled
>	}
>}*  


Tambien se creó un índice con el comando:  
>curl -X PUT "localhost:9200/hot_warm_cold_index-000001?pretty" -H 'Content-Type: application/json' -d'
>{
>"aliases": {
>"hotWarmCold": {
>"is_write_index": true
>}
>}
>}
>'  

![List_of_indexes](https://github.com/Roger-Fox/Tecnologia_de_Contenedores/blob/main/pictures/index_management.png)  

Con todo configurado, solo faltaría agregar algunos documentos para ver como son procesados por las tres fases principales de hot, warm y cold.  
