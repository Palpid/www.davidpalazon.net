#Monitoreo de la puerta de sincronización de la base del sofá con Prometeo y Grafana

El lanzamiento de Couchbase Mobile 2.5 introdujo amplias capacidades de reporte de estadísticas en el Sync Gateway. Las estadísticas proporcionan información clave sobre la salud de su despliegue Couchbase Mobile y constituye una parte integral de cualquier despliegue.
En este post, discutimos cómo puede utilizar Prometheus, una plataforma de monitorización y alerta de código abierto para supervisar sus nodos Sync Gateway y Grafana para visualizar las estadísticas. En un próximo post relacionado, discutiremos cómo puede configurar la monitorización con Prometeo en un cluster de Couchbase Mobile Kubernetes.


Background
Reporting Estadisticas Sync Gateway 

Las estadísticas del Sync Gateway se reportan en formato JSON y están disponibles a través del endpoint _expvar por medio de la interfaz REST del Sync Gateway Admin.
Estas son las categorías de estadísticas que se reportan.

***stats_2.5.category.png***

## Prometheus

Prometheus es una plataforma de vigilancia y alerta de sistemas de código abierto y está alojada en la Cloud Native Computing Foundation. En el centro de la misma se encuentra el Servidor Prometeo que se encarga de sondear los "objetivos de Prometeo" para las estadísticas y almacenarlos como datos de series temporales. Los objetivos de Prometeo están configurados estáticamente o pueden ser descubiertos por Prometeo.

## Grafana

Grafana es una plataforma abierta de visualización de datos y de alerta. Soporta el Prometeo como fuente de datos y puede ser usado para construir cuadros de mando integrales.

##Introduciendo el Exportador de la Gateway Sync de Prometheus

Para que Prometeo monitorice la puerta de sincronización, necesitamos un " target Prometheus" que corresponda a la puerta de sincronización. Este objetivo es el Exportador de la Puerta de Sincronía, en lo sucesivo denominado "el Exportador". En pocas palabras, el Exportador es responsable de exportar las estadísticas de la pasarela de sincronismo a la métrica de Prometeo.

*** exporter.png ***

# Arquitectura de despliegue

Uniendo todo esto, un despliegue típico sería como el siguiente...

- **Sync Gateway Exporter** El Sync Gateway Admin REST API está, por defecto, sólo expuesto en el localhost. Esta configuración se recomienda encarecidamente en entornos de producción también. Como resultado, ya que el Exportador encuesta la API de administración de la puerta de enlace de sincronización para las estadísticas, el Exportador tendría que estar en el mismo host/nodo que la puerta de enlace de sincronización. Además, las estadísticas de la puerta de enlace de sincronización se informan por cada nodo. Así que en un grupo de dos o más nodos Sync Gateway, cada nodo Sync Gateway debe tener su propio Exportador.
- El servidor de Prometeo continuamente busca estadísticas del Exportador, que a su vez busca el  endpoint   RESTSync Gateway. El servidor usa las reglas definidas en el rules.yaml para enviar alertas al **Alerting Manager**.
- El servicio **Grafana** encuesta al Servidor Prometeo por estadísticas y lo grafica en un tablero basado en la web al que se puede acceder a través del navegador web.

#Instalación

En el resto del post, recorreremos los pasos para configurar un cluster de Couchbase Mobile para el monitoreo con el Exportador, el Server Prometheus y Grafana. Aunque las instrucciones están pensadas para un entorno de desarrollo, se pueden personalizar fácilmente para un entorno de producción.

Usaremos docker, lo que hará que sea extremadamente sencillo de poner en marcha.

##Descargando el código fuente del exportador
El Sync Gateway Exporter es un proyecto de código abierto de couchbaselabs y está disponible en GitHub. Es compatible con el Sync Gateway 2.5 de Couchbase. Para ello, aunque nos esforzaremos por mantenerlo actualizado con la última versión de Sync Gateway, no podemos garantizarlo. Pero la buena noticia es que es de código abierto e incluye [instrucciones](https://github.com/couchbaselabs/couchbase-sync-gateway-exporter/blob/master/docs/develop/adding-new-metrics.md) detalladas sobre cómo ampliarlo para soportar estadísticas adicionales.

Descargue la fuente del Exportador de github

```
git clone https://github.com/couchbaselabs/couchbase-sync-gateway-exporter
``` 

#Desplegando el cluster móvil de Couchbase

Todos los comandos de esta sección deben ser ejecutados desde la carpeta raíz del repo de github clonado.

```
cd /path/to/exporter/cloned/repo
```

Si ya tienes un grupo Couchbase Mobile corriendo en docker, puedes saltarte este paso y pasar al paso "Desplegando el exportador de la puerta de enlace de sincronización". Sólo recuerda sustituir el nombre de la red de dockers correspondiente a tu configuración existente en las instrucciones de abajo.

#Creando una red de dockers

Cuando se utiliza docker, se recomienda ejecutar todos los componentes en la misma red de dockers.
Crea una red de conexiones con el nombre "demo".

```
docker network create demo
```

##Desplegando el servidor de la base del sofá

Utilizaremos una imagen especial de acoplamiento del Servidor Couchbase v6.0.1 que incluye un cubo de muestra preinstalado llamado "TravelSample" así como un usuario RBAC preconfigurado con "Acceso a la Aplicación" que permitiría a Sync Gateway conectarse al Servidor Couchbase.
Si lo prefiere, puede instalar la imagen de la última versión del Servidor Couchbase y configurarlo manualmente con un cubo y un usuario RBAC de Sync Gateway.

```
docker run -d –name cb –network demo -p 8091–8094:8091–8094 -p 11210:11210 connectsv/server:6.0.1-enterprise
```

##Desplegando Sync Gateway

El Sync Gateway se lanzará con el archivo de configuración llamado sync-gateway-config.json que se encuentra en la carpeta "testdata" del repo clonado.

```
docker run -p 4984–4985:4984–4985 –network demo –name sync-gateway -d -v `pwd`/testdata/sync-gateway-config.json:/etc/sync_gateway/sync_gateway.json couchbase/sync-gateway:2.5.0-enterprise -adminInterface :4985 /etc/sync_gateway/sync_gateway.json
```

Pruébalo.

Confirme que su puerta de sincronización está en marcha y que el punto final de _expvar es accesible ejecutando el siguiente comando curl. Si las cosas funcionan como se espera, deberías ver un montón de estadísticas en la consola.

```
curl GET http://localhost:4985/_expvar
```

##Desplegando el exportador de la puerta de enlace de sincronización
La imagen del exportador de la puerta de enlace de sincronización está disponible en couchsamples/sync-gateway-prometheus-exporter:latest.

La opción de configuración sgw.url es de particular interés. Esta apunta al servidor de la puerta de enlace de sincronización. Si está trabajando con un despliegue existente de Couchbase Mobile, asegúrese de configurar sgw.url para que apunte al nodo Sync Gateway en su configuración.

El Exportador se vincula a 0.0.0.0 por defecto. Puede anularlo usando la opción --web.listen-address.

```
docker run -p 9421:9421 -network demo -name exporter -d couchbasesamples/sync-gateway-prometheus-exporter:latest --log.level=debug --sgw.url=http://sync-gateway:4985
```

Pruébalo.
Confirme que el exportador está en marcha con el siguiente comando de rizo.

```
curl http://localhost:9421/metrics
```

Si las cosas han funcionado, habrá un montón de métricas de salida a la consola. Confirma que el "sgw_up 1".

*** Prometheus metrics.png ***

##Desplegando el servidor de Prometeo
Ahora que tenemos al Exportador exportando estadísticas de la Sync Gateway, prepararemos a Prometeo para que haga una encuesta al Exportador para las métricas.

###Explorando el archivo prometheus.yml
El prometheus.yml especifica la configuración con la que se lanza el servidor de Prometheus. Se encuentra en la carpeta "testdata" del repo clonado. Examinemos el contenido de este archivo.

```
 global:
 scrape_interval: 5s
 evaluation_interval: 5s
```

```
rule_files:
- &amp;apos;/etc/prometheus/rules/*&amp;apos;

scrape_configs:
- job_name: swg
    static_configs:
    - targets:
    - exporter:9421&lt;/code&gt;&lt;/pre&gt;
```

- El *scrape_interval* especifica el intervalo de la votación. Puede ajustarlo según sus necesidades
- Los *rule_files* especifican la ubicación del fichero de las Reglas de Prometeo. El archivo de reglas de Prometeo, sync-gateway.rules.yml se encuentra en la carpeta testdata/rules/ en el repositorio clonado. Prometheus utiliza las reglas para activar las alertas. Hemos proporcionado un conjunto básico de reglas como punto de partida. Puede personalizar las reglas según sea necesario. El archivo de reglas se monta en /etc/prometheus/rules cuando se lanza el contenedor del muelle.
- Los *targets* especifican los objetivos de Prometeo que es el Exportador. Puede tener varios exportadores si tiene varios Sync Gateways. En ese caso, puede especificar estáticamente todos los puntos finales del Exportador o configurar Prometeo para que descubra los objetivos.

###Instalando la imagen de la base
Desplegaremos la imagen oficial del docker *prom/prometheus* con el archivo prometheus.yml incluido en el repo clonado.

```
docker run -p 9090:9090 –network demo –name prometheus -d -v `pwd`/testdata/prometheus.yml:/etc/prometheus/prometheus.yml -v `pwd`/testdata/rules:/etc/prometheus/rules prom/prometheus
```

#### Try it out
Abre la URL *http://localhost:9090/graph* en un navegador web.
- Haz clic en el botón "Insert metric at Cursor" para ver la lista de métricas disponibles del Sync Gateway.
- Haga clic en "Status”->“Targets" para ver la lista de exportadores y sus estadísticas
- Haga clic en el botón del menú "Alerts" para ver el estado de las alertas

*** prometheus_ui.gif ***

#Desplegando Grafana

Aunque podrías usar la interfaz web de Prometeo para visualizar las estadísticas, usaremos Grafana ya que puedes crear unos cuadros de mando muy convincentes con ella y Grafana se integra bien con Prometeo.

- Instalaremos la imagen oficial de *grafana/grafana:6.2.0* docker. La carpeta grafana/data en el repositorio clonado contiene el grafana.db que se utiliza para la persistencia del tablero y otros metadatos. Este es un volumen montado en *var/lib/grafana*

```
 docker run -p 3000:3000 –network demo –name grafana -d grafana/grafana:6.2.0 -v `pwd`/grafana/data:/var/lib/grafana
```

- Un tablero de mandos predeterminado de "Couchbase Sync Gateway" llamado dashboard.jsonnet está disponible en el repositorio clonado como un archivo jsonnet. Haga que el objetivo de Grafana genere el correspondiente archivo *dashboard.json* que luego puede ser importado a Grafana. El dashboard grafica todas las métricas exportadas por el Exportador. Puede personalizarlo según sus necesidades.

```

# Pull in relevant submodules that jsonnet needs
git submodule update –init –rebase –remote –recursive

# Make the grafana target to generate the dashboard.json file
make grafana

```

#### Try it out
Abre la URL http://localhost:3000 en un navegador web. Debería ver la pantalla de acceso. Inicie sesión con las credenciales por defecto de "admin" y la contraseña de "admin". Puede cambiarlo después del inicio de sesión inicial.

*** grafana_login.png ***

El siguiente paso sería añadir "Prometeo" como la "Fuente de Datos" e importar el archivo json "Sync Gateway dashboard" que se generó anteriormente. Podrías hacer esto manualmente siguiendo las opciones del menú. Hemos simplificado este proceso y proporcionado un guión que hará todo eso por ti!
Ejecuta el script desde la raíz de tu repo clonado usando el comando de abajo ejecutando make en el objetivo grafana-dev. Esto hace lo siguiente
- (Re)Genera el tablero de mandos *dashboard.json* a partir del archivo dashboard.jsonnet
- Utiliza la API de Grafana para añadir el Prometeo como fuente de datos
- Utiliza el API de Grafana para cargar el *dashboard.json* generado en el paso anterior

```
make grafana-dev
```

Una vez que el script se ejecute con éxito, deberías refrescar la interfaz de Grafana en tu navegador. Verás el "Couchbase Sync Gateway Dashboard" en la lista de tableros disponibles. El tablero grafica todas las estadísticas reportadas por el Sync Gateway. Puedes personalizar el tablero.

*** dashboard1.png ***

Haga clic en el "Tablero de la Puerta de Sincronización de Couchbase" para ver las estadísticas. Puedes filtrarlo por el Sync Gateway (si tienes más de uno) o por la base de datos

*** stats-1.png ***

¡Eso es! Has configurado con éxito el monitoreo con Prometheus y Grafana. Ahora puedes hacer réplicas con clientes de Couchbase Lite y monitorearlo. El tablero de mandos del Sync Gateway por defecto es un punto de partida. Puedes personalizar el tablero, ya sea editando el archivo dashboard.jsonnet o directamente a través de la interfaz de usuario de Grafana.

#### TL;DR

...puede exportar las estadísticas de Sync Gateway a Prometeo y visualizarlas usando herramientas de visualización como Grafana. Esto simplifica enormemente la monitorización de sus grupos de Couchbase Mobile. Además de Exportador, hemos proporcionado un tablero de Grafana predeterminado que usted puede personalizar. personalizando el tablero mismo, también puede personalizar las reglas para las alertas. 
