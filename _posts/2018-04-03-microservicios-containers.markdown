---
layout: post
title:  "Microservicios con Docker & Vagrant"
date:   2018-04-03 14:44:00
categories: DevOps
---

Recientemente construi un ambiente para ayudarme a entender de que va esto de los contenedores y meterme un poco en el mundo de DevOps. 

[Docker](https://www.docker.com/) es una de las tecnologias que mayor popularidad ha alcanzado en este ultimo tiempo y ha logrado una gran adopcion por parte de developers y devOps.

__Docker__ permite crear y gestionar multiples sistemas (*containers*) totalmente aislados entre si, sobre una misma maquina. Estos *containers* se caracterizan por ser: 

* __Autosuficientes__ - tienen todo lo que necesitan para trabajar

* __Portables__ - ejecutan sobre cualquier plataforma que tenga el engine de Docker (similar a una app Java sobre una JVM)

* __Ligeros__ - no se virtualiza todo un sistema completo, sino solo lo necesario (la virtualizacion se realiza a nivel SO)

Estas, son solo algunas de las ventajas que hicieron que grandes del sector como Google, Red Hat, IBM y Microsoft, etc. se hayan implicado en su desarrollo e implantacion, ofreciendolos ademas a traves de sus servicios Cloud que todos usamos a diario (AWS, DigitalOcean, Azure, etc.)



### La idea

El objetivo de este trabajo es poder empaquetar un microservicio dentro de un *Docker container*, que sea capaz de ejecutar sobre cualquier plataforma, independientemente de las librerias, configuraciones, sistema operativo, etc. que tenga, y que pueda escalar facilmente en base a la demanda, sin necesidad de hacer *re-working* sobre el ambiente ya creado.



### Comenzando

Lo primero que necesitamos es tener Docker instalado. Los contenedores de Docker extienden [LXC (LinuX Containers)](https://linuxcontainers.org/lxc/introduction/) y no funcionan de forma nativa en Windows, por lo que en mi caso tenia estas opciones:

1. Usar [Docker Toolbox](https://docs.docker.com/toolbox/) 

2. Crear una maquina virtual Linux  usando [Vagrant](https://www.vagrantup.com/) 

Aunque a *priori* la primer opcion parecia ser la mas rapida y fiable de tener Docker funcionando en mi maquina, tuve algunos problemas al momento de crear la maquina virtual que hicieron que me decante por la segunda opcion. 
Ademas, de esta manera se tiene mucho mas control sobre la maquina virtual en la que se ejecutara Docker y el escenario es mas parecido a como se usaria en ambientes productivos.

### Vagrant al rescate

[Vagrant](https://www.vagrantup.com/) es una herramienta para la creacion y gestion de entornos de desarrollo virtualizados (*boxes*). Esta permite generar entornos de desarrollo reproducibles y compartibles de forma muy sencilla. 

Para poder instalarlo basta con correr el [instalador](https://www.vagrantup.com/downloads.html) y validar la instalacion a traves de `vagrant version`:

~~~~~~
C:\Desarrollo\Vagrant\MicroserviceWithDocker\host>vagrant version
Installed Version: 1.8.1
~~~~~~~~~~~~

Nota: Por defecto, Vagrant utiliza [VirtualBox](https://www.virtualbox.org/) como motor de maquinas virtuales (tambien existe la opcion de utilizar VMWare), por lo que tambien es necesario tenerlo instalado para que funcione.

### Creando nuestro ambiente (Vagrant box)

En este punto es donde vamos a describir que tipo de maquina necesitamos, como configurarla y como aprovisionarla. En Vagrant todo esto se realiza a traves de un [Vagrantfile](https://www.vagrantup.com/docs/vagrantfile/). En nuestro caso, el Vagrantfile luce algo parecido a esto:

~~~~~~
...
Vagrant.configure(2) do |config|
  config.vm.box = "debian/jessie64"
  ...
  config.vm.provision "docker"
  config.vm.provision "docker_compose"
  ...
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "forwarded_port", guest: 7070, host: 7070
  config.vm.network "forwarded_port", guest: 1936, host: 1936  
  ...
  config.vm.synced_folder "../docker_data", "/vagrant/docker_data"
  
~~~~~~~~~~~~

Basicamente lo que aca le estamos diciendo a Vagrant es:

* Crear una maquina virtual con [Vanilla Debian 8 "Jessie"](https://app.vagrantup.com/debian/boxes/jessie64) como SO
* Aprovisionar la maquina virtual con [Docker](https://www.docker.com/) y [Docker-Compose](https://docs.docker.com/compose/)
* Redirigir los puertos __80, 7070 y 1936__ de la maquina virtual a nuestra maquina local
* Compartir la carpeta __docker_data__ entre nuestra maquina local y la maquina virtual

Vagrant leera nuestro Vagrantfile y entonces construira e iniciara la maquina. Para esto basta con hacer `vagrant up`: 

~~~~~~
C:\Desarrollo\Vagrant\MicroserviceWithDocker\host>vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Checking if box 'debian/jessie64' is up to date...
...
==> default: Forwarding ports...
    default: 80 (guest) => 8080 (host) (adapter 1)
    default: 7070 (guest) => 7070 (host) (adapter 1)
    default: 1936 (guest) => 1936 (host) (adapter 1)
    default: 22 (guest) => 2222 (host) (adapter 1)
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 127.0.0.1:2222
    default: SSH username: vagrant
...
==> default: Setting hostname...
==> default: Mounting shared folders...
    default: /vagrant => C:/Desarrollo/Vagrant/MicroserviceWithDocker/host
    default: /vagrant/docker_data => C:/Desarrollo/Vagrant/MicroserviceWithDocker/docker_data
...
~~~~~~~~~~~~

Si todo va bien, veremos una salida similar a la de arriba y podremos acceder a la misma a traves de `vagrant ssh`:

~~~~~~
C:\Desarrollo\Vagrant\MicroserviceWithDocker\host>vagrant ssh

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have new mail.
Last login: Sun Apr  1 18:40:08 2018 from 10.0.2.2

vagrant@docker-host:/$ docker --version
Docker version 18.03.0-ce, build 0520e24

vagrant@docker-host:/$ docker-compose --version
docker-compose version 1.11.2, build dfed245
~~~~~~~~~~~~

Como podemos ver, la maquina ya se encuentra aprovisionada con Docker y Docker-Compose como fue especificado en el Vagrantfile.

### Creando nuestros contenedores (Docker)

Ya creada nuestra maquina, aprovisionada con Docker y Docker-Compose, es momento de crear nuestros Docker *containers*. 

Para crear un *container* en Docker, primeramente es necesario crear una imagen. Una imagen es una especie de plantilla a partir de la cual se instancian los *containers*. Seria algo asi como un *snapshot* de una maquina virtual, pero mucho mucho mas ligera. 

![docker-images-containers](/assets/images/docker-images-containers.png)

Existe un [registro publico de Docker](https://hub.docker.com/) donde se puede obtener la mayoria de las imagenes base que se utilizan para la creacion de nuestras propias imagenes.

En nuestro caso, partimos de la imagen [__frolvlad/alpine-oraclejdk8:slim__](https://hub.docker.com/r/frolvlad/alpine-oraclejdk8/) que esta basada en Alpine Linux, una distribucion Linux de solo 5 MB, a la cual se le ha a&#241;adido una OracleJDK 8.

~~~~~~
FROM frolvlad/alpine-oraclejdk8:slim
MAINTAINER jpolivo
ADD encripter3des-aas-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 7070
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
~~~~~~~~~~~~

Similar a Vagrant, Docker utiliza un [Dockerfile](https://docs.docker.com/engine/reference/builder/) para construir la imagen en forma declarativa. 

Aqui, lo que le estamos diciendo a Docker a traves de nuestro Dockerfile es:

* __FROM__: Tomar como imagen base __frolvlad/alpine-oraclejdk8__
* __ADD__: Copiar el archivo __encripter3des-aas-0.0.1-SNAPSHOT.jar__ al contenedor con el nombre __app.jar__
* __EXPOSE__: Exponee el puerto __7070__ hacia fuera (es el puerto por defecto en el que escuchara el tomcat embebido de nuestro microservicio)
* __ENTRYPOINT__: El comando a ejecutar cuando se levante el contenedor


Para crear la imagen basta con hacer `docker build` sobre el path que contiene nuestro Dockerfile. Una vez creada nuestra imagen podemos comprobar que fue agregada a nuestro repo de imagenes a traves de `docker images` y compartirla con `docker push`

~~~~~~
vagrant@docker-host:~$ docker images

REPOSITORY                          TAG                 IMAGE ID            CREATED             SIZE
jpolivo/encripter3des-aas-backend   0.0.1-SNAPSHOT      526acf1a9ebe        8 days ago          194MB
frolvlad/alpine-oraclejdk8          slim                9f2fc54fc35a        5 weeks ago         167MB
dockercloud/haproxy                 latest              4d6ae6c16c4d        3 months ago        42.6MB
~~~~~~~~~~~~

Cada imagen se identifica por un ID, y un par nombre-version. En nuestro caso : [jpolivo/encripter3des-aas-backend:0.0.1-SNAPSHOT](https://hub.docker.com/r/jpolivo/encripter3des-aas-backend/)

Con la imagen ya definida, vamos a crear nuestros containers y balancearlos a traves de [HA proxy](http://www.haproxy.org/)

### Orquestando contenedores

Docker-Compose permite gestionar aplicaciones compuestas por varios contenedores relacionados entre si a traves de un archivo YML. En este archivo se definen todos los contenedores y sus relaciones.

Con esto, nuestro archivo __docker-compose.yml__ queda formado de la siguiente manera:

~~~~~~
version: '2'
services:
   microservice:
	image: 'jpolivo/encripter3des-aas-backend:0.0.1-SNAPSHOT'
	expose:
	  - "7070"

   loadbalancer:
	image: 'dockercloud/haproxy:latest'
	environment:
	  - STATS_PORT=1936
	  - STATS_AUTH="admin:admin"
	links:
	  - microservice
	volumes:
	  - /var/run/docker.sock:/var/run/docker.sock
	ports:
	  - '80:80'
	  - '1936:1936'
~~~~~~~~~~~~

Resumiendo, estamos definiendo:

* Usar la version 2 de docker-compose 
* Crear los *containers*:
	* __microservice__ a partir de la imagen 'jpolivo/encripter3des-aas-backend:0.0.1-SNAPSHOT' previamente creada
	* __loadbalancer__ a partir de la imagen ['dockercloud/haproxy:latest'](https://github.com/docker/dockercloud-haproxy)
* Exponer el puerto 7070 sobre el *container* __microservice__
* Exponer los puertos 80 y 1936 sobre el *container* __loadbalancer__

Una vez declarados nuestros *containers* y sus relaciones, vamos a instanciarlos a traves de `docker-compose`:

~~~~~~
vagrant@docker-host:~$ docker-compose -f /vagrant/docker_data/docker-compose.yml up -d

Creating dockerdata_microservice_1
Creating dockerdata_loadbalancer_1
~~~~~~~~~~~~

Mediante `docker ps` podemos ver que nuestros contenedores ya se encuentran *up and running*:

~~~~~~
vagrant@docker-host:~$ docker ps
CONTAINER ID        IMAGE                                              COMMAND                  CREATED              STATUS              PORTS                                                 NAMES
ea5bc684f6b9        dockercloud/haproxy:latest                         "/sbin/tini -- docke..."   About a minute ago   Up About a minute   0.0.0.0:80->80/tcp, 0.0.0.0:1936->1936/tcp, 443/tcp   dockerdata_loadbalancer_1
6af38d21803e        jpolivo/encripter3des-aas-backend:0.0.1-SNAPSHOT   "java -Djava.securit..."   About a minute ago   Up About a minute   7070/tcp                                              dockerdata_microservice_1
~~~~~~~~~~~~

Hacemos algunas llamadas a nuestro microservicio:

~~~~~~
C:\Desarrollo\sources\encripter3des-aas-backend>curl --request GET http://localhost:8080/key/encrypt?password=73738184
b825fc55c339a28d

C:\Desarrollo\sources\encripter3des-aas-backend>curl --request GET http://localhost:8080/key/encrypt?password=11111111
5ba7ad645b48d969

C:\Desarrollo\sources\encripter3des-aas-backend>curl --request GET http://localhost:8080/key/encrypt?password=98988117
01674b983f270b7f
~~~~~~~~~~~~

Revisamos las [estadisticas del balanceador](http://localhost:1936/):

![haproxy-microservicios](/assets/images/haproxy-microservicios.png)

Podemos ver que todas las llamadas a nuestro microservicio son atendidas por la unica instancia que existe. Pero, que pasaria si la cantidad de llamadas a nuestro microservicio comienza a crecer?
Seguramente, llegara un momento en el que la unica instancia que tenemos no podra satisfacer toda la demanda. Llegado ese momento, necesitaremos escalar la solucion.

### Escalando la solucion

Mediante `scale`, podemos sumar tantas instancias de nuestro microservicio como necesitemos: 

~~~~~~
vagrant@docker-host:~$ docker-compose -f /vagrant/docker_data/docker-compose.yml scale microservice=3

Creating and starting dockerdata_microservice_2 ... done
Creating and starting dockerdata_microservice_3 ... done
~~~~~~~~~~~~
 
En este caso, vemos que se han creado 2 nuevas instancias que se suman a la ya existente

![haproxy-microservicios_2](/assets/images/haproxy-microservicios_2.png)


Volvemos a repetir las llamadas:

~~~~~~
C:\Desarrollo\sources\encripter3des-aas-backend>curl --request GET http://localhost:8080/key/encrypt?password=73738184
b825fc55c339a28d

C:\Desarrollo\sources\encripter3des-aas-backend>curl --request GET http://localhost:8080/key/encrypt?password=11111111
5ba7ad645b48d969

C:\Desarrollo\sources\encripter3des-aas-backend>curl --request GET http://localhost:8080/key/encrypt?password=98988117
01674b983f270b7f
~~~~~~~~~~~~

Ahora, las llamadas a nuestro microservicio son redistribuidas por el balanceador entre las 3 instancias existentes 

![haproxy-microservicios_3](/assets/images/haproxy-microservicios_3.png)

Nada mal no? Imaginemos si tenemos que hacer esto en una infraestructura tradicional, el costo por agregar/quitar nuevas instancias del microservicio seria altisimo.

### Resumen

Como pudimos ver, los contenedores permiten la creacion de infraestructuras agiles, consistentes y replicables, y al utilizarlos con aplicaciones orientadas a microservicios, permiten que los cambios se implementen de manera simple y con bajo costo operacional.

Todo el codigo que utilizado en esta guia puede descargarse desde este [repositorio](https://github.com/jpOlivo/microservice-with-docker-vagrant.git)



