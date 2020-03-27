---
title: "Sincronizando  con Couchbase Mobile Aplicaciones de dibujo"
date: 2020-03-27T16:15:45+01:00
draft: false
slug: "sync_couchbase_mobile_draw_app"
tags: ["coubase","sync"]
categories: ["Database","Mobile"]
markup: mmark
comments: true 
noauthor: true 
---

#Sincronizando  con Couchbase Mobile Aplicaciones de dibujo
 
*** Nota: Esta publicación es una traducción libre de una entrada del blog de couchbase. Para uso personal ***

Couchbase Mobile contiene una robusta y potente base de datos NoSQL integrada, llamada Couchbase Lite, que puede ser utilizada dentro de tus aplicaciones iOS, Android y Xamarin. La pila de Couchbase Mobile también contiene Sync Gateway. Sync Gateway permite la sincronización segura de datos entre los clientes habilitados para Couchbase Lite. Sync Gateway ha existido por años, pero el año pasado Couchbase Mobile 2.0 introdujo oficialmente un nuevo protocolo de replicación basado en web-sockets para la sincronización de datos que es más eficiente que su predecesor basado en HTTP.

Mientras consideraba todos los cambios en Couchbase Mobile, y el reciente lanzamiento de la versión 2.5, empecé a pensar en modos únicos de visualizar las mejoras usando una aplicación. Empecé por definir un par de requisitos sencillos. Quería que la aplicación:

- Sea simple.
- Sea divertido.
- Muestre el poder de Couchbase Mobile

Finalmente, decidí investigar qué se necesitaría para crear una aplicación que incluyera algo que probablemente todos hemos hecho de niños, por diversión o cuando simplemente estamos aburridos. ¡Dibujar! ¿Y qué es más divertido que dibujar? Así es, ¡el dibujo colaborativo!

***snake.jpg***

## Empezando...
No había creado antes una aplicación que soportara el dibujo en tiempo real, pero estaba familiarizado con algunos enfoques que podrían hacer el trabajo.  La primera opción para mí fue una biblioteca que ha tenido cierto éxito durante algunos años: Skia.

Skia es una biblioteca de gráficos 2D de código abierto que proporciona API comunes que funcionan en una variedad de plataformas de hardware y software. Sirve como motor de gráficos para Google Chrome y Chrome OS, Android, Mozilla Firefox y Firefox OS, y muchos otros productos, incluidos los móviles. Skia es mantenido en su mayor parte por Google, pero es completamente gratis para que cualquiera lo use.
Me gustó la idea de usar Skia para apoyar mis dibujos con efectos táctiles, pero quería crear una aplicación multiplataforma. Teniendo más de unos pocos años de experiencia en Xamarin en mi haber, ya estaba familiarizado con SkiaSharp, una implementación de Skia basada en .NET y multiplataforma.

Así que creé una solución Xamarin.Forms, e incluí el paquete nuget SkiaSharp. Quedé impresionado, ya que después de sólo unas pocas líneas de código tenía un Skia Canvas (SkCanvasView) en mi UI

```
<skia:SKCanvasView x:Name="canvasView" PaintSurface="OnCanvasViewPaintSurface" />
```
y fue capaz de empezar a escuchar los eventos táctiles en el código de mi code behind.

```
void OnTouchEffectAction(object sender, TouchActionEventArgs args)
{
    var point = new Models.Point
    {
        X = args.Location.X,
        Y = args.Location.Y
    };

    switch (args.Type)
    {
        case TouchActionType.Pressed:
            ...
            break;
        case TouchActionType.Moved:
            ...
            break;
    }
}
```

A partir de esos eventos pude reunir los fundamentos para las representaciones vectoriales 2D en un lienzo. Después de todo, los dibujos bidimensionales se reducen a un componente central; un punto. Una colección de puntos crea un camino, y una colección de caminos crea... un dibujo.

¡Hurra! Ahora que podía capturar los puntos, crear caminos desde esos puntos y, finalmente, dibujar desde esos caminos, tenía los datos que necesitaba para compartir entre los lienzos en aplicaciones separadas.


## The Power of Couchbase Mobile
Como se mencionó anteriormente, la pila Couchbase Mobile contiene tanto una base de datos NoSQL incrustada llamada Couchbase Lite, como un mecanismo de sincronización de datos llamado Sync Gateway. Combinado con el poder del Servidor Couchbase, una base de datos distribuida de NoSQL basada en JSON, los datos pueden ser empujados y tirados hacia y desde el extremo.

La compatibilidad con Sync Gateway está totalmente integrada en los paquetes Couchbase.Lite y Couchbase.Lite.Enterprise Nuget, y puedes encontrar más información sobre cómo hacerlo aquí.

La clave para almacenar datos en una base de datos basada en documentos se centra en cómo se modelan los datos. Por suerte, en esta aplicación, el modelado de datos fue fácil. Todo dio un giro completo porque todo lo que necesitaba almacenar, a través de JSON, era una colección de puntos dentro de colecciones de caminos.

```

{
    "color": "#000000",
    "createdBy": "cca6ebe8-a713-49ac-bb86-cff0fb095ab2",
    "id": "059fee8c-fbb3-450e-a1f1-61d82a28e68b",
    "points": [
        {
            "type": "point",
            "x": 101.333,
            "y": 339.667
        },
        {
            "type": "point",
            "x": 101.333,
            "y": 340.3333282470703
        }
    ],
    "type": "path"
}

```

A continuación, simplemente guardé la información de punto y ruta en la base de datos de Couchbase Lite. Desde allí, Sync Gateway se encarga del resto, y, voilà, un lienzo compartido hecho simple con Couchbase Mobile!

***N1QL_Rick.gif***

## Presentamos CouchDraw

Obviamente, he pasado por alto algunos de los detalles esenciales de la aplicación de los que he estado hablando, pero no teman, ¡he creado un nuevo [repositorio](https://github.com/couchbaselabs/CouchDraw) para ello! En el nuevo repositorio "CouchDraw", junto con el código fuente, también he incluido un LÉAME con muchos más detalles sobre cómo descargar, configurar y ejecutar la solución. Siéntanse libres de dejar todo lo que están haciendo ahora mismo y vayan a comprobarlo!

CouchDraw es una aplicación muy básica, y como tal, hay muchas formas de mejorarla (aquí es donde entras tú). Desafío a la comunidad de móviles y Couchbase a que se sumerjan y amplíen la funcionalidad de CouchDraw!

Algunas nuevas ideas de características pueden incluir:

- Reemplazar los botones de color con un deslizador de rango para la selección de color.
- Añadir la capacidad de cambiar el grosor de la línea.
- Añadir la capacidad de borrar las líneas que se han dibujado.

