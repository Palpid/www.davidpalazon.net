---
title: "Usando Docker para desarrollar con Couchbase Mobile"
date: 2020-03-03T16:15:45+01:00
draft: false
slug: "manual-couchbase-sync-introduccion"
tags: ["couchbase","manuales","sync"]
categories: ["general","manuales","couchbase"]
markup: mmark
comments: true 
noauthor: true 
---

Las tecnologías de contenedores como Docker han simplificado enormemente el proceso de desarrollo, prueba e implementación de software, permitiéndole empaquetar aplicaciones junto con su entorno de ejecución completo que abstrae las diferencias de infraestructura y sistemas operativos.  Este post es una introducción a cómo puedes usar Docker para configurar tu entorno de desarrollo Couchbase Mobile y ofrece algunos consejos para la resolución de problemas. 


Todo lo que se discute en este post se aplica a un simple entorno de desarrollo con el objetivo de acelerar el proceso de desarrollo para que puedas empezar a construir aplicaciones para móviles de forma rápida y sencilla utilizando Couchbase Mobile.

En un post relacionado, discutimos cómo puede aprovechar Kubernetes para ampliar y gestionar el despliegue del clúster de Couchbase Mobile en entornos de producción. El Operador Autónomo de Couchbase simplifica enormemente la tarea de desplegar y gestionar su clúster.

#Antecedentes

Explicando a un alto nivel, la pila completa de Couchbase Mobile comprende los siguientes componentes
- El Couchbase Lite, que es la base de datos NoSQL integrada en tus aplicaciones móviles
- El Sync Gateway, que es la puerta responsable de la sincronización de datos entre los clientes y el servidor Couchbase
- El servidor Couchbase, responsable de la persistencia de los datos

Para empezar a desarrollar con Couchbase Mobile, necesitas una instancia de Couchbase Server y Sync Gateway. Integrarás el framework Couchbase Lite en tus aplicaciones.

En este post, aprenderemos a utilizar Docker para desplegar una instancia de un nodo Sync Gateway y un nodo único Couchbase Server cluster en un entorno de escritorio adecuado para el desarrollo.

#Instalación de Docker

Si aún no lo has hecho, por favor instala Docker según la guía de instalación para tu entorno de escritorio.

Puedes verificar la instalación de Docker escribiendo el siguiente comando en una ventana de terminal.

```
    docker --version
```

Deberías obtener una respuesta similar a la siguiente

```
	    Docker version 18.03.0-ce, build 0520e24
```

#Installing Couchbase Server

El servidor Couchbase está disponible en el Docker Hub en el couchbase repo. En el momento de escribir este post, esta versión es la 6.0.1.

- Primero obtendrás la imagen del Docker Hub. Abre una ventana de terminal y ejecuta el siguiente comando

```
    docker pull couchbase/server
``` 

- Crear una red de docker local llamada "cbnetwork" si no existe ya. Abra una ventana de terminal y ejecute el siguiente comando

```
    docker network ls
    docker network create -d bridge cbnetwork
```

#Configurando el servidor Couchbase

Una vez que tu servidor Couchbase esté instalado y funcionando, tendrás que configurarlo antes de que puedas empezar a usarlo con tu Sync Gateway.

Aquí está el conjunto mínimo de las cosas que tendrás que hacer :-

- Crear un cluster con los servicios apropiados. Un solo nodo agrupado es suficiente para las necesidades de desarrollo
- Configurar la cuenta del administrador para el acceso al servidor
- Configurar el grupo
- Crear un cubo predeterminado
- Crear un usuario RBAC con acceso a nivel de bucket apropiado. Las credenciales del usuario RBAC son utilizadas por el Sync Gateway para autenticarse con el servidor Couchbase

Tienes dos opciones para configurar el servidor Couchbase: manual y automático. Salta a la sección apropiada dependiendo de tu elección.

#Opción 1: Configuración mediante la interfaz de administración

Puedes configurar el servidor de Couchbase manualmente a través de la interfaz de la consola de administración de Couchbase.
Para acceder a la interfaz de administración, debemos ejecutar la imagen del cargador que obtuvimos antes.

- Ejecute el servidor Couchbase con el siguiente comando. Esto ejecuta el servidor Couchbase como un proceso demoníaco.

```
    docker run -d --name cb-server --network cbnetwork -p 8091-8094:8091-8094 -p 11210:11210 couchbase/server
```

- Puede ver los logs en cualquier momento ejecutando el siguiente comando

```
    docker logs -f cb-server
```

- Si tu servidor se ha lanzado con éxito, deberías ver algo como esto en tu salida

```
    Starting Couchbase Server -- Web UI available at http://&lt;ip&gt;:8091 and logs available in /opt/couchbase/var/lib/couchbase/logs
```

- El servidor puede tardar unos segundos en arrancar. Verifica que la imagen de la plataforma se está ejecutando con el siguiente comando

```
    docker ps
```

- Una vez que el servidor esté en marcha, accede a él abriendo la URL http://localhost:8091 en tu navegador
- Siga las instrucciones de la guía de configuración para configurar la cuenta de administrador y crear un único grupo de nodos.
- Siga las instrucciones aquí para crear un cubo
- A continuación, crea un usuario RBAC para que la puerta de sincronización se conecte. Este usuario será creado con el rol "con acceso a la aplicación" como se especifica en las instrucciones aquí
- Una vez que tu servidor Couchbase esté configurado, asegúrate de tomar nota de
    - El nombre del bucket que creaste
    - Las credenciales de usuario de RBAC que usó para la configuración

Las credenciales RBAC y el nombre del bucket serán requeridos cuando esté listo para configurar su Sync Gateway

El proceso manual está bien, pero puede resultar tedioso si tienes que repetir este proceso varias veces en cada una de tus configuraciones de desarrollo. ¿No sería genial si los pasos de configuración pudieran ser automatizados? Si estás interesado en aprender sobre eso, procede a la siguiente sección, si no, salta a la sección de configuración del Sync Gateway.

#Opción 2: Configuración usando el CLI

He generado una imagen personalizada del docker a partir de la imagen del Couchbase Server 6.0.1 que le permitirá lanzar el contenedor con los valores de configuración adecuados a través de la línea de comandos. Esto será particularmente útil si quieres automatizar/escribir el proceso de instalación/automatización.

Descargue esta versión de desarrollo personalizado de la imagen del Servidor Couchbase basada en el Servidor Couchbase 6.0.1

```
    docker pull priyacouch/couchbase-dev-6.0
```

- Una vez que haya descargado con éxito la imagen personalizada, puede lanzarla proporcionándole los valores de configuración adecuados como parte del comando de lanzamiento. Asegúrese de registrar las credenciales de usuario de RBAC y el nombre del cubo. Estos serán relevantes durante la configuración de la puerta de enlace de sincronización
    - COUCHBASE_ADMINISTRATOR_USERNAME es el nombre del Administrador de Couchbase
    - COUCHBASE_ADMINISTRATOR_PASSWORD es la contraseña del administrador de Couchbase
    - CUBO_BASE_BUCKET es el nombre del cubo de la base de datos que te gustaría crear
    - COUCHBASE_RBAC_USERNAME es el nombre del usuario RBAC de la puerta de enlace Sycn con acceso al cubo a nivel de aplicación
    - COUCHBASE_RBAC_PASSWORD es la contraseña del usuario de RBAC
    - COUCHBASE_RBAC_NAME es un nombre fácil de usar para el usuario de RBAC
    - CLUSTER_NAME el nombre del cluster del Couchbase Server
	Abra una ventana de terminal y escriba el siguiente comando. Puede proporcionar valores adecuados para cada uno de los parámetros configurables.
    
```
    $ docker run -d --name cb-server --network cbnetwork -p 8091-8094:8091-8094 -p 11210:11210 -e COUCHBASE_ADMINISTRATOR_USERNAME=Administrator -e COUCHBASE_ADMINISTRATOR_PASSWORD=password -e COUCHBASE_BUCKET=demobucket -e COUCHBASE_RBAC_USERNAME=admin -e COUCHBASE_RBAC_PASSWORD=password -e COUCHBASE_RBAC_NAME="admin" -e CLUSTER_NAME=demo-cluster priyacouch/couchbase-dev-6.0
```

- You can view the logs at any time by running the following command

```
   docker logs -f cb-server
```

- You have to be patient. It takes a few minutes for the server to get up and running. If succesful, your output should look something like this
   
```   
    &lt; 
    100    50    0     0  100    50      0   1172 --:--:-- --:--:-- --:--:--  1219
    * Connection #0 to host 127.0.0.1 left intact
    SUCCESS: Bucket created
    SUCCESS: RBAC user set
    /entrypoint.sh couchbase-server
```

- ¡Eso es! Ahora ya puedes probar tu instalación.
Accede a ella abriendo la URL http://localhost:8091 en su navegador y verifique que su configuración es la especificada

*** docker_cbs_6.gif ***

#Construyendo una imagen personalizada y configurable del Docker
Si se preguntan cómo generé la imagen personalizada con opciones configurables, hay un par de maneras de hacerlo. Pero tomé un enfoque inspirado en el tutorial. Esencialmente construí una imagen personalizada del docker a partir de la imagen base del servidor de Coucbase y la configuré para nuestras necesidades de desarrollo!

Hay una gran cantidad de valores configurables a medida como se describe en las especificaciones de la interfaz de Coucbase CLI y REST. En la imagen personalizada de la plataforma, permití la configuración de algunos parámetros críticos y dejé los otros por defecto.

Si desea generar su propia imagen basada en una versión diferente de Couchbase Server y/o si desea personalizar los parámetros configurables, puede hacerlo siguiendo los pasos especificados en esta [guía](https://gist.github.com/rajagp/8d05314d85fcbf169ee39a671077a566)

#Instalando la Sync Gateway

Ahora que tienes el servidor Couchbase configurado y en funcionamiento, instalaremos el Sync Gateway. Es importante que el servidor Couchbase esté configurado y funcionando antes de que empieces a usar la puerta de enlace.

El Sync Gateway está disponible en Docker Hub en el repositorio de Couchbase.

- Primero obtendrás la imagen del Docker Hub. Abre una nueva ventana de terminal y ejecuta lo siguiente.

```
	    docker pull couchbase/sync-gateway 
```

- El Sync Gateway debe ser lanzado con un archivo de configuración en el que se especifica, entre otras cosas, la URL del Couchbase Server al que se debe conectar, el bucket al que se debe acceder y las credenciales RBAC que se deben utilizar para el acceso al bucket. El archivo de configuración determina el comportamiento en tiempo de ejecución de la pasarela de sincronización.

La imagen del docker que sacaste está construida con un archivo de configuración predeterminado. Si no especificas ninguno, eso es lo que se usará (y probablemente no te funcionará).

- Si tienes una configuración que te gustaría usar, ábrela en un editor de tu elección. Si no, crea un nuevo archivo de configuración llamado sync-gateway-config.json y copia la siguiente configuración.

```
{
  "interface":":4984",
   "logging": {
    "log_file_path": "/var/tmp/sglogs",
    "console": {
      "log_level": "debug",
      "log_keys": ["*"]
    },
    "error": {
      "enabled": true,
      "rotation": {
        "max_size": 20,
        "max_age": 180
      }
    },
    "warn": {
      "enabled": true,
      "rotation": {
        "max_size": 20,
        "max_age": 90
      }
    },
    "info": {
      "enabled": false
    },
    "debug": {
      "enabled": false
    }
  },
  "databases": {
    "demobucket": {
      "import_docs": "continuous",
      "enable_shared_bucket_access":true,  
      "bucket":"demobucket",
      "server": "http://cb-server:8091",
      "enable_shared_bucket_access":true,
      "username": "admin",
      "password": "password",
      "num_index_replicas":0,
      "users":{
          "GUEST": {"disabled":true},
          "admin": {"password": "password", "admin_channels": ["*"]}
      },
      "revs_limit":20
   }
}
}		
```
Puede añadir una función de sincronización apropiada o cualquiera de las otras propiedades de configuración. Nos centraremos en las principales que son esenciales para nuestro entorno de desarrollo. Debe hacer las modificaciones apropiadas en el archivo de configuración como se especifica a continuación.

- La URL del servidor especifica el nombre del contenedor del Couchbase Server. En el comando docker run usado para lanzar el Servidor Couchbase, especificamos el nombre usando la opción --name.
- La base de datos y el bucket deben corresponder al valor $COUCHBASE_BUCKET que se utilizó al configurar el Servidor Couchbase. En nuestro ejemplo, eso se especificó para ser demobucket.
- El nombre de usuario corresponde al nombre de usuario de la cuenta RBAC que creó para el acceso al cubo, como se especifica en el valor $COUCHBASE_RBAC_USERNAME que utilizó al configurar el servidor Couchbase. En nuestro ejemplo, eso fue especificado para ser admin.
- La contraseña corresponde a la contraseña de la cuenta RBAC que creó para el acceso al cubo, especificada por el valor $COUCHBASE_RBAC_PASSWORD que utilizó al configurar el servidor de Couchbase. En nuestro ejemplo, eso fue especificado como bepassword`.
- Una vez que el archivo de configuración esté configurado, iniciará la puerta de enlace de sincronización con el archivo. Para ello, abre una terminal y ejecuta los siguientes comandos

```
    cd /path/to/sync-gateway-config.json

    docker run -p 4984-4985:4984-4985 --network cbnetwork --name sync-gateway -d -v `pwd`/sync-gateway-config.json:/etc/sync_gateway/sync_gateway.json couchbase/sync-gateway -adminInterface :4985 /etc/sync_gateway/sync_gateway.json
```

- Puede ver los registros en cualquier momento ejecutando el siguiente comando

```
    docker logs -f sync-gateway
```

- Puede tomar unos segundos para que la puerta de sincronización se inicie. Verifica que la imagen docker se está ejecutando con el siguiente comando

```
    docker ps
```

- Verifica que tu puerta de sincronización se está ejecutando abriendo la URL http://localhost:4984 en tu navegador.
Debería ver la siguiente salida
```
    {"couchdb":"Welcome","vendor":{"name":"Couchbase Sync Gateway","version":"2.1"},"version":"Couchbase Sync Gateway/2.1.2(2;35fe28e)"}
```

¡Eso es! Tienes tu instancia docker de la puerta de sincronización hablando con el servidor de Couchbase

#Manejando su entorno
En esta sección, repasamos algunos comandos básicos de los dockers que le ayudarán a gestionar su entorno.

##Detener/ Iniciar contenedores
Puede detener y reiniciar los contenedores de muelle en cualquier momento usando los comandos de parada y arranque de muelle de la siguiente manera.
- Detener los contenedores

```
    docker stop sync-gateway
    docker cb-server
```

- Starting Containers

```
    docker start cb-server
    docker start sync-gateway
```

Ten en cuenta que si paras el servidor Couchbase, el Sync Gateway intentará reconectarse con el servidor durante unos minutos antes de rendirse. Así que si el servidor se detiene durante un período de tiempo prolongado, tendrá que detener y reiniciar el contenedor del Sync Gateway también o utilizar la API **_online** para volver a ponerlo en línea.

#Actualizando la configuración  Sync Gatway
- Si desea actualizar la configuración de  Sync Gatway, deberá volver a ejecutar  Sync Gatway con un archivo de configuración de la puerta de enlace de sincronización actualizado. Para ello, deberá detenerse y eliminar el contenedor de Sync Gatway.

```
   docker stop sync-gateway
   docker rm sync-gateway
```

Si no eliminas la imagen de la puerta de enlace de sincronización, verás un “name conflict error” similar al siguiente si intentas iniciar Sync Gateway de nuevo con la configuración actualizada.

```
   docker: Error response from daemon: Conflict. The container name "/sync-gateway" is already in use by container "bc67153afda9b90303b2965b62c5e34751ce3748fd8d5fb7ed38a418d7b77cfd". You have to remove (or rename) that container to be able to reuse that name.

   See 'docker run --help'.
```

#Actualizando la configuración del servidor Couchbase

- Del mismo modo, si quieres volver a ejecutar el servidor Couchbase con una configuración actualizada, tendrás que detener y eliminar el servidor Couchbase.

```
   docker stop cb-server
   docker rm cb-server
```

Sin embargo, dependiendo de la configuración del servidor que se cambie, puede que también tenga que detener y eliminar el contenedor de Sync Gatewat y volver a iniciarlo con el archivo de configuración actualizado de Sync Gateway. Por ejemplo, si ha cambiado las credenciales RBAC del cubo o si ha cambiado el nombre del Bucket.


#Corriendo comandos en el contenedor
A veces, es posible que quieras ejecutar los comandos directamente en el contenedor de ejecución. Para ello, puedes usar el comando docker exec para abrir un shell en el contenedor. Esto es muy útil para la depuración y demás. Necesitarás privilegios de root para poder ejecutar el comando.

- servidor couchbase

```
	sudo docker exec -i -t cb-server /bin/bash
```
- sync gateway

```
 sudo docker exec -i -t sync-gateway /bin/bash
```

# Prueba de API de la interfaz REST de Couchbase Sync Gateway usando Postman

La puerta de sincronización de Couchbase es uno de los componentes principales de la pila de Couchbase Mobile. A un alto nivel, es responsable del enrutamiento seguro y la sincronización de datos entre los clientes [web y móvil] y el servidor de Couchbase. Soporta una API REST que permite a los clientes realizar operaciones administrativas y no administrativas en la base de datos del servidor Couchbase. Si está desarrollando una aplicación cliente que hace interfaz con el Sync Gateway, entonces necesitaría una forma conveniente de explorar la API y probarla, probablemente incluso burlándose de todas las llamadas al Sync Gateway.  Podemos usar Postman para este propósito.

Postman es una herramienta de prueba, desarrollo y documentación de la API. El Sync Gateway soporta las colecciones Postman correspondientes a su interfaz Admin y Public REST.  

Todo lo que se discute en este post utiliza la versión gratuita para la comunidad de la herramienta Postman.

##Antecedentes

La pila móvil de Couchbase comprende el servidor Couchbase, la puerta de sincronización Couchbase y la base de datos integrada Couchbase Lite. Este post asume que usted está familiarizado con la pila móvil de Couchbase.

##TL;DR

Si lo prefieres, puedes ver el siguiente video que es una demostración de todo lo que se discute en esta entrada del blog.

[Video Cocuhbase Sync Official](https://youtu.be/BaoB2wBPlD8)

#Instalando el Postman
Antes de proceder, debe descargar e instalar la herramienta Cartero. También tendrá que crear una cuenta gratuita.

#Importing the Sync Gateway Collection
Getting started with using Postman with Sync Gateway is very easy.

- First, download the Sync Gateway postman collections from GitHub repo

```
	git clone https://github.com/couchbaselabs/Couchbase-Sync-Gateway-Postman-Collection.git
```

- Follow the instructions detailed in this introductory post to import the collections and environment files into Postman. At the end of it, your setup should look something like this.

![Postman](https://blog.davidpalazon.net/images/collections_imported.png)


#Testing with a Sync Gateway Instance

- Si aún no lo has hecho, adelante y descarga la instancia de Sync Gateway y/o el servidor Couchbase
	- Siga las instrucciones aquí para descargar e instalar el Sync Gatway. Si quieres que tus datos persistan en el Servidor Couchbase, sigue las instrucciones de esta guía de inicio rápido para instalar el Servidor Couchbase.
	- Si prefieres usar Docker, puedes seguir las instrucciones aquí para empezar con Couchbase Mobile usando Docker.
- Hay un ejemplo de archivo sync-gateway-config que está disponible en el mismo repositorio github de Collections que sacaste antes. Los valores de configuración especificados en el archivo de configuración de muestra corresponden a las variables de entorno predeterminadas. Por lo tanto, si desea utilizar los valores predeterminados en el archivo de entorno, puede iniciar la pasarela de sincronización con el archivo de configuración de muestra sync-gateway-config.
- Actualice "adminurl" y "publicurl" para que apunten a los puntos finales de administración y público de su despliegue de la pasarela de sincronización.
- Si ha iniciado su Sync Gateway con un archivo de configuración diferente del archivo sync-gateway-config de muestra, entonces, además de actualizar las variables "adminurl" y "publicurl", asegúrese de actualizar las demás variables de entorno relevantes para que coincidan con su configuración. Algunas de ellas incluyen "db", "name", "password" y "doc".
Eso es todo, ¡estás listo para probar con un sistema vivo!

#Solicitar encadenamiento
Una de las cosas interesantes que apoyamos en nuestras colecciones es el encadenamiento de solicitudes, es decir, la salida de una solicitud puede ser utilizada como entrada para una solicitud posterior. Esto se logra actualizando dinámicamente las variables de entorno con los resultados de la ejecución de la solicitud sin necesidad de editar manualmente las solicitudes.

Por ejemplo,
- Cuando se crea un documento, se crea una nueva revisión del documento y se devuelve el ID de la revisión recién creada en la respuesta del campo "_rev". Tenemos una prueba que extrae el valor "_rev" y establece la variable de entorno "rev"

![createdoc](https://blog.davidpalazon.net/images/createdoc.png)

La solicitud posterior de actualización del documento requiere que se especifique la identificación de la revisión en la solicitud. Ésta se recupera de la variable de entorno "rev" que se pobló como resultado de la ejecución de la solicitud anterior.

![updatedoc](https://blog.davidpalazon.net/images/updatedoc.png)

