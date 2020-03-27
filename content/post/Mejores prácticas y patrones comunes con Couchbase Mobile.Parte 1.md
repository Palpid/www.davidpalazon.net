---
title: "Mejores prácticas y patrones comunes con Couchbase Mobile"
date: 2020-03-27T16:26:45+01:00
draft: false
slug: "best_practice_couchbase_sync"
tags: ["couchbase","sync"]
categories: ["Database","Mobile","Sync"]
markup: mmark
comments: true 
noauthor: true 
---

# Mejores prácticas y patrones comunes con Couchbase Mobile  

Desde su primera publicación oficial en 2014, Couchbase Mobile ha permitido una amplia variedad de casos de uso de diversos grados de escala y complejidad. A pesar de la variación, hay algunos patrones de uso comunes para el uso de Couchbase Mobile.

He reunido una serie de entradas de blog que cubren algunos patrones de uso comunes y consejos y prácticas recomendadas para abordar estos casos de uso. En esta entrada del blog, discutimos los patrones comunes si estás desarrollando aplicaciones con Couchbase Lite. Esta entrada asume que estás familiarizado con los fundamentos de Couchbase Mobile. Si necesitas una introducción, consulta los documentos de Couchbase Lite.

## Patrón 1: Manejo de un gran volumen de datos públicos, no específicos de un usuario

**Imagen prebuilt.png** 


Con el soporte de Couchbase Lite para bases de datos pre-construidas, puedes precargar la aplicación con datos en lugar de sincronizarla desde Sync Gateway durante el inicio.

## Caso de uso

Evitar la sincronización inicial ayudará a reducir el tiempo de inicio y los costos de transferencia de la red. Esto se aplicaría típicamente a los datos públicos/compartidos, no específicos del usuario que son en su mayoría estáticos. Incluso si los datos no son estáticos, se pueden aprovechar las ventajas de precargar los datos y de sincronizar los cambios sólo al inicio.

### Aproximación
- Crear una copia no codificada de la base de datos .cblite con los datos pertinentes. Hay dos opciones para crear un .cblite
	- Utiliza una aplicación basada en Couchbase Lite para obtener datos relevantes a través de la puerta de enlace de sincronización. Esto creará una instancia de un archivo cblite (con extensión .cblite2). La ubicación del archivo generado es específica de la plataforma y se puede obtener del objeto DatabaseConfiguration.
	- Utiliza la herramienta de código abierto, couchbaselabs [cblite](https://github.com/couchbaselabs/couchbase-mobile-tools/blob/master/README.cblite.md). Esta herramienta le permite cargar datos de una variedad de fuentes.
	- Una vez cargado, sigue los pasos indicados a continuación para copiar el archivo .cblite2 en la aplicación. Esto creará una instancia de la base de datos única para el cliente

	#### Cargando una base de datos pre-construida (C#)
	La forma de que nuestra aplicación sincronice muchos datos inicialment y que son practicamente estáticos , es incluir una base de datos en nuestra aplicación e instalarla en el primer lanzamiento. Incluso si parte del contenido cambiase en el servidor después de crear la aplicación, la primera replicación de la aplicación pondrá al día la base de datos.

####     En este caso necsitaremosla base de datos preconstruida como indicabamos arriba. Será necesario configurar la base de datos e  incorporarla al paquete de la aplicación como un recurso incrustado e instalarla la primera vez que lancemos la aplicación. Necesitará comprobar si la base de datos existe. Si la base de datos no existe, la aplicación deberá copiarla del app bundle utilizando el método Database.Copy(string, DatabaseConfiguration) como se muestra en el ejemplo a continuación.
    
    ```
    // Note: Getting the path to a database is platform-specific.  For .NET Core / .NET Framework this
    // can be a simple filesystem path.  For UWP, you will need to get the path from your assets.  For
    // iOS you need to get the path from the main bundle.  For Android you need to extract it from your
    // assets to a temporary directory and then pass that path.
    var path = Path.Combine(Environment.CurrentDirectory, "travel-sample.cblite2" + Path.DirectorySeparatorChar);
    if (!Database.Exists("travel-sample", null)) {
        _NeedsExtraDocs = true;
        Database.Copy(path, "travel-sample", null);
    }
    ```

## Patrón 2: Segregación de datos locales y sincronizados del cliente

** Multidb.png**

Puedes soportar múltiples instancias de Couchbase Lite dentro de tu aplicación. Aunque no hay límites estrictos en cuanto al número de instancias de la base de datos, sí hay límites y restricciones prácticas, por ejemplo, no se pueden hacer uniones de bases de datos cruzadas.

## Caso de uso

Un caso de uso común de tener múltiples bases de datos dentro de la aplicación es segregar los datos que son locales sólo para el cliente en una instancia de base de datos separada del resto de los datos. Hay otras formas de hacer cumplir las restricciones de sólo local, como veremos en un patrón separado que se discute más adelante. Sin embargo, la segregación de los datos permitiría a las aplicaciones aplicar el control de acceso a nivel de la base de datos y asegurar que las consultas y las operaciones de la base de datos se amplíen al nivel de la base de datos. Por ejemplo, se puede eliminar la base de datos de sólo local sin que ello afecte al resto de los datos sincronizados.

### Aproximación

- Crear una instancia de la base de datos Couchbase Lite para los datos de sólo local. La base de datos puede ser creada o copiada de una copia de la base de datos pre-construida
- No configure ningún replicador para la base de datos local.

# Patrón 3: Soportar una aplicación multiusuario

*** multiuser.png ***

Las aplicaciones potenciadas por Couchbase Lite pueden soportar múltiples usuarios. Este es otro patrón que se habilita con la capacidad de soportar múltiples instancias de Couchbase Lite dentro de tu aplicación. Las aplicaciones multiusuario imponen requisitos estrictos de segregación de datos y control de acceso a los datos para garantizar que los usuarios no accedan o pisoteen los datos de los demás de forma deliberada o inadvertida.

## Caso de uso
Las aplicaciones multiusuario son comunes, especialmente en escenarios donde los dispositivos son compartidos.

### Aproximación

- Crear una instancia separada de la base de datos Couchbase Lite para cada usuario
- Aunque no es necesario, cada instancia de la base de datos puede estar en una carpeta específica del usuario, cuya ubicación puede especificarse mediante la DatabaseConfiguration en el momento de la creación de la base de datos. Si se desea, cada base de datos de Couchbase Lite se puede cifrar utilizando una contraseña/clave específica del usuario.
- Los datos comunes compartidos entre los usuarios pueden almacenarse en una instancia compartida de la base de datos en una carpeta "común
- Al cambiar de usuario o "desconectarse", asegúrese de lo siguiente
    - Eliminar todos los listener en las consultas
    - Detener a todos los replicadores
    - Cerrar la base de datos

# Patrón 4: Compartir datos entre aplicaciones de un dispositivo

*** multiapp.png ***

Las aplicaciones para móviles funcionan en un entorno sandbox. Así que una determinada aplicación sólo tiene acceso o sólo puede modificar sus propios datos. Sin embargo, en ciertas plataformas existen opciones que facilitan el intercambio de recursos entre aplicaciones con los derechos adecuados. Utilizando estos mecanismos especificados por la plataforma, una instancia de la base de datos Couchbase Lite puede ser compartida por múltiples aplicaciones. Ello se ajusta a las directrices prescritas por la plataforma.


|Configuración | Implicación |
|--------------|----------------|
|Todos los lectores, no escritores.|Ok|
|Un solo escritor, varios lectores|De acuerdo, no hay notificaciones de cambios entre procesos.
|Múltiples escritores| No probados. La base de datos se bloqueará con múltiples accesos simultáneos (YMMW*)

## Caso de uso
Múltiples aplicaciones del mismo proveedor podrían estar trabajando con los mismos datos. En lugar de que cada aplicación mantenga una copia idéntica de los datos, el hecho de compartir la base de datos reducirá los costos de transferencia así como los costos de almacenamiento local en el dispositivo.

### Aproximación

- La idea es alojar la base de datos en una carpeta o en un lugar del sistema de archivos al que puedan acceder las aplicaciones que la comparten. La configuración de las aplicaciones para compartir datos es una implementación a nivel de plataforma
    - En iOS, utilizarías grupos de aplicaciones similares a los descritos en este blog para configurar el acceso a los recursos compartidos de Couchbase Lite desde varias aplicaciones.
    - En Android, podrías aprovechar los proveedores de contenido para permitir compartir datos de Couchbase Lite entre aplicaciones

# Patrón 5: Mantener los datos actualizados mientras la aplicación está en segundo plano

*** backhround-sync.png ***

El soporte de fondo en las aplicaciones móviles varía drásticamente según la plataforma. De hecho, en ciertas plataformas como Android, el concepto de "aplicaciones de fondo" es bastante nebuloso/complicado. Así que la forma en que sincronizarías los datos mientras tu aplicación está en segundo plano depende mucho de la plataforma. Puedes encontrar una discusión sobre cómo Couchbase Lite reacciona al ciclo de vida de las aplicaciones en iOS, Android y Windows en nuestra documentación.

## Caso de uso

Al mantener los datos actualizados cuando no están en uso activo (o en primer plano), las aplicaciones pueden mejorar la experiencia del usuario final al reducir el tiempo de inicio en los lanzamientos subsiguientes.


### Aproximación

- Cabe señalar que no hay garantía de que se permita que su aplicación funcione en segundo plano. Eso es típicamente una decisión a nivel de sistema o a nivel de usuario.
- La forma en que una aplicación maneja la sincronización en segundo plano depende de la plataforma
- Se recomienda generalmente que, cuando sea posible, su aplicación reaccione a los acontecimientos adecuados del ciclo de vida de la aplicación y cierre el replicador antes de pasar a segundo plano
- ***IOS***
    - Las réplicas continuas pasan a modo offline cuando la aplicación se pasa a segundo plano
    - Utilice el Apple Push Notification Sevice (APNS) para enviar una notificación silenciosa que despierte la aplicación en segundo plano o utilice Background App Refresh para que el sistema despierte la aplicación de forma oportunista.
    - Cuando la aplicación se despierte en segundo plano, haz una réplica de una sola vez para sincronizar los datos
    - Más sobre el soporte de fondo en iOS en esta entrada de blog.
- ***Android***
	- Hay un par de opciones disponibles si las aplicaciones tienen que ser ejecutadas en segundo plano
		- Use el servicio de primer plano para las réplicas de larga duración en segundo plano
		- Utilice Work Manager para programar réplicas de un solo disparo para que se ejecuten asincrónicamente en segundo plano o utilice Remote Firebase Cloud Messaging(FCM) para iniciar una solicitud de trabajo para ejecutar una réplica de un solo disparo
- ***UWP***
    - Las réplicas continuas no pasan al modo offline cuando una aplicación se pasa a segundo plano. Sin embargo, se recomienda que las aplicaciones cierren el replicador cuando pasen a segundo plano
    - Usar las notificaciones sin procesar enviadas a través del Servicio de Notificaciones Push de Windows (WPNS) para despertar la aplicación mientras está en segundo plano
    - Cuando la aplicación se despierte en segundo plano, programe una tarea en segundo plano para hacer una réplica de una sola vez para sincronizar los datos

# Patrón 6: Purga después del push
***purgeaftepush.png***

Couchbase Lite admite la posibilidad de que las aplicaciones sean notificadas sobre el estado de replicación de un documento o conjunto de documentos mediante la capacidad de Eventos de Replicación. Las aplicaciones pueden aprovechar esta capacidad para tomar las medidas adecuadas en el documento dependiendo del estado, como por ejemplo la purga del documento.

## Caso de uso
Puede ser conveniente eliminar los documentos de la tienda local del cliente después de que se hayan sincronizado con el servidor por razones de cumplimiento de las normas de gobierno o para evitar que la base de datos local se hinche.

### Aproximación
- Registrar el observador de eventos de replicación en Replicator
- Al recibir el evento onPushed en el documento(s), invoca la purga() API para deshacerse del documento. Los documentos purgados no están sincronizados.

#Patrón 7: Aplicar la vida útil de los documentos del lado del servidor en clientes desconectados
***docexpiratoin.png***

Couchbase Lite soporta la capacidad para que las aplicaciones establezcan fechas de caducidad en los documentos locales a través de la función de fecha de caducidad. Las aplicaciones pueden aprovechar esta capacidad para expirar localmente un documento en Couchbase Lite, independientemente de la conectividad con el servidor.

## Caso de uso
Los documentos creados en el servidor pueden estar asociados a un TTL o a una fecha de caducidad que dicta cuándo hay que purgar el documento del sistema (clientes y servidor). Cuando hay conectividad de red, los documentos eliminados en el lado del servidor se sincronizarán con el lado del cliente para que se eliminen también en el lado del cliente. Sin embargo, es probable que los clientes estén desconectados cuando los documentos expiren en el servidor y puede ser importante que esos documentos se eliminen de los clientes en el momento oportuno.

###Aproximación
- Modele sus documentos para incluir una propiedad definida por el usuario que especifique la hora UTC cuando el documento expire. Llamémosla validityUntil.
- Registrar el observador de eventos de replicación en el cliente Replicador
- Al recibir el evento Pull en el/los documento/s, utilice el valor validUntil para establecer el TTL en los documentos utilizando la API setExpirationDate()
- Cuando los documentos caducan en el cliente, se purgan automáticamente del cliente independientemente de la conectividad de la red. Los documentos purgados no se sincronizan.
- En el lado del servidor, los documentos pueden ser purgados independientemente
- ***Advertencia***: Si la fecha de caducidad (como se define en la propiedad validUntil) se actualiza mientras los clientes están fuera de línea, entonces los clientes no serán notificados de este cambio. Así que es posible que los documentos se purguen prematuramente en los clientes cuando estén desconectados. Por lo tanto, este enfoque es seguro siempre y cuando la vida útil del documento no cambie después de la creación. Discutiremos otro enfoque para manejar estos casos.

#Patrón 8: Controlar cuándo y qué documentos se sincronizan con el servidor
***replicator_filter.png***
Couchbase Lite soporta la capacidad de las aplicaciones de establecer filtros de grano fino en el replicador que determina qué documentos son empujados al servidor. Esta capacidad puede ser aprovechada para controlar cuándo y qué documentos se sincronizan con el servidor.

##Caso uso
En un blog anterior, discutimos cómo una instancia separada de una base de datos puede ser utilizada para mantener datos locales. Aunque ese patrón funciona cuando se necesita una estricta segregación de datos y si no se requieren consultas de unión de bases de datos cruzadas, hay casos de uso en los que se querría tener un control detallado sobre cuándo se deben sincronizar los cambios locales. También se podría utilizar este patrón como alternativa al patrón de base de datos preconstruida para hacer cumplir que los documentos son sólo locales (es decir, nunca se sincronizan).


###Aproximación
- Modele sus documentos para incluir una propiedad de "estado" definida por el usuario que controle el flujo de trabajo del negocio. Por ejemplo, utilice un valor de "en curso" para indicar cuando los cambios no están listos para ser sincronizados y un valor de estado de "comprometido" cuando los cambios necesitan ser subidos. Si necesita hacer los documentos sólo localmente, utilice una propiedad "scope" definida por el usuario con un valor de "local" para indicar que los documentos no se van a sincronizar
- Configure la función de filtro de replicación adecuada en el replicador. El filtro debería inspeccionar la propiedad del documento (como se ha definido anteriormente) para determinar si un documento necesita ser sincronizado o no
- Cada vez que hay cambios listos para ser empujados, la función de filtro es aplicada por el replicador Couchbase Lite y los documentos son sincronizados sólo si se cumplen los criterios de filtro.

# Patrón 9: Controlar el orden de prioridad del conjunto inicial de documentos sincronizados con el cliente

Los canales del Sync Gateway permiten a las aplicaciones segregar los datos en función del tipo, el propósito, los permisos de control de acceso, etc. Esto puede ser aprovechado para segregar los datos en función de la prioridad.

##Caso Uso
La capacidad de priorizar los documentos recibidos por los clientes aseguraría que éstos puedan ponerse en marcha con los cambios de documentos más relevantes tan pronto como los haya sacado sin tener que esperar a que el resto de los documentos se sincronicen. El resto de los cambios de documentos de menor prioridad se retiran más tarde. Esto reduce los tiempos de respuesta de la aplicación en el inicio inicial.


###Aproximación
- Asignar documentos a diferentes canales en función de la prioridad. Por ejemplo, se podrían asignar documentos a canales de prioridad "baja", "media" o "alta".
- En el lado del cliente, en el lanzamiento inicial, haga una réplica pull de un canal de mayor prioridad especificando el filtro de canales. Una vez que se completa la sincronización de los documentos en el canal, la aplicación puede iniciar otra réplica para los canales restantes.
- También puede configurar un replicador continuo para el canal de mayor prioridad y tirar de los canales restantes sólo a petición utilizando la replicación de un solo disparo. Esto asegurará que el canal de mayor prioridad se mantenga sincronizado en tiempo real.

