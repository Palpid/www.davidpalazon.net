# Cómo usar los canales en el Sync Gateway

![](http://)

Continuando con la serie técnica del Sync Gateway, veremos específicamente cómo configurar la función de sincronización mediante el uso de canales para ejecutar la orquestación de datos.  En el siguiente vídeo, nos acompaña Chris Anderson, que repasará un ejemplo de cómo hacer el enrutamiento de datos.  Hablamos de los controles de acceso anteriormente y los Canales proveen acceso de LECTURA a los usuarios entre el cliente móvil y la base de datos remota.  El ejemplo es tomar las preguntas de Stackoverflow y cargarlas en Sync Gateway donde sólo se sincronizarán las preguntas específicas y por lo tanto las etiquetas específicas de los temas que le interesan al usuario.  A partir de las etiquetas, se pueden generar los nombres de los canales.  Así es como se sincronizan los subconjuntos de datos hasta el móvil según el interés del usuario.

## Channels

Las etiquetas se utilizan como la clave de tipo de documento y es una forma de controlar la accesibilidad a documentos específicos de la base de datos.  La forma en que esto se hace es utilizando una función de JavaScript que se construye dentro de Sync Gateway, donde somos capaces de hacer que cada documento pase y enrutar el documento al canal particular al que pertenecen. Somos capaces de pasar el conjunto de etiquetas a la función de canal para crear los nombres de los canales de forma dinámica.


```
{  ...
  “tags” : [
    “android”,
    “java”,
    “nosql”,
    “couchbase"
  ],
  ...
}

```

El archivo de configuración JSON de la puerta de enlace de sincronización incluirá la función de sincronización donde el conjunto de etiquetas se pasan a la función de canal.

```
{
  "log" : ["REST"],
  "databases" : {
    "db": {
      "server" : "walrus:."
      "users": {"GUEST" : {"disabled" : false, "admin_channels": ["*"] }},
      "sync" :
      function(doc) {
        channel(doc.tags);
      }
    }
  }
```

En la consola de administración de Couchbase, se muestra el canal de diferentes etiquetas con el contenido respectivo que se captura.  Esta consola ayuda a desarrollar la lógica de la función de sincronización y a recopilar la información relevante que se intenta definir.

## Replication

Un cliente móvil es entonces capaz de sincronizar el contenido relevante del usuario utilizando la API de replicación incorporada de Couchbase Lite.  A continuación se muestra un ejemplo de código Objective-C que trata de establecer el interés del usuario por el tema en el lado del cliente para obtener datos relevantes.  El replicador sabrá entonces cómo interactuar con los datos desde dentro de los canales.  Primero creamos una réplica Pull
```
CBLReplication *pull = [database createPullReplication: url];
pull.channels = @[@“android”, @“java”, @“nosql”];
[pull start];
```

El replicador interactuará con los nombres de los canales específicos para llevar los datos a los dispositivos móviles.  Si los canales específicos no están configurados, entonces todos los datos que existen serán bajados de la puerta de enlace de sincronización.  Aquí es también donde podemos tener una experiencia asíncrona para los usuarios que están desconectados.  Cuando la conexión esté disponible, se reanudará la sincronización de los nuevos datos de esas etiquetas de canal.

Cómo se hace esto es a través de la API de alimentación de cambios, db/_cambios de la Sync Gateway donde la alimentación se compone de los metadatos de estos documentos.  El identificador de revisión _rev es utilizado por el cliente para omitir la extracción de documentos de versiones particulares que ya tienen, de modo que permita una reconexión eficiente.  Lo que impulsa la sincronización es el número de secuencia en el que el cliente le dice a Sync Gateway en qué secuencia concreta debe reanudarse para el nuevo contenido.  La clave "last_seq" es un número entero que representa el último cambio que el cliente ha sincronizado.

```
{ ...
-  {
          seq:  116,
          id:"123123kj12h310193",
       - changes: [
            - {
                rev:  "1-12313124324"
              }
          ]
    },
-  {
          seq:  117,
          id:"191123kj12h310193",
       - changes: [
            - {
                rev:  "1-12398124378"
              }
          ]
    }
  ],
    last_seq:  "117"
}
```

Desde la serie de blogs del Sync Gateway, podemos validar aún más los tipos de documentos, así como proporcionar seguridad mediante la autorización de los usuarios en el Sync Gateway.  Para aprender más sobre cómo desarrollar y solucionar los problemas de los canales en Sync Gateway, asegúrese de leer las guías de formación para obtener más información.