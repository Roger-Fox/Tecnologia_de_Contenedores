# Tecnologia de Contenedores  
## Proyecto de Tecnologia de Contenedores - Hot Warm and Cold
## Integrantes: James Allan Weisnner - Verbo Julian Camacho  
  
Primero fue necesario instalar la imagen de elasticsearch, la cual puede conseguirse en la pagina oficial y en el portal Docker hub.  

*docker pull docker.elastic.co/elasticsearch/elasticsearch:7.15.*  
Al haber descargado la imagen, construiremos un contenedor apartir de dicha imagen con el comando  
*docker run --name elasticsearch -d -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.15.1*  
En este comando estamos usando el puerto 9200 del cliente y del contenedor, y usando un solo nodo de elasticsearch. En este caso lo utilizamos únicamente para probar que Elasticsearch esta funcionando correctamente  

A continuación, utilizamos parte del repositorio estudiado en clase, esto con el fin de tener una base para ejecutar el comando Docker compose, que iniciaría una arquitectura ELK (Elasticsearch, Logstash, Kibana) que será usada a detalle más adelante  
*docker-compose -f Proyecto-compose.yml up*  

recursos imagen  ++++++++++++++

Tras asignar mas recursos a la máquina virtual Linux, fue posible ejecutar el comando docker-compose exitosamente.  

cluster corriendo imagen +++++++++++++

Ejecutamos el comando docker ps, y nos damos cuenta de los tres nodos de frio, tibio y caliente, están corriendo adecuadamente junto a los contenedores de Logstash y Kibana.  
cluster status +++++++++++++++  
Con toda nuestra arquitectura ELK funcionando, empezamos a escribir el comando que definiría las políticas de ciclo de vida de los datos. Este paso tomo un tiempo considerable, pues no comprendíamos bien como funcionaban dichos comandos, finalmente utilizamos una política que representa bien el ciclo de vida Hot, Warm y Cold. Este puede ser visto en el repositorio del proyecto bajo el nombre “ILM Command”  

