---
title: "Manual (Traducido) Couchbase Sync"
date: 2020-03-03T16:15:45+01:00
draft: false
slug: "manual-couchbase-sync-introduccion"
tags: ["couchbase","manuales","sync"]
categories: ["general","manuales","couchbase"]
markup: mmark
comments: true 
noauthor: true 
---

#Introducción

*Sync Gateway* es el servidor de sincronización en la implantación de Couchbase Mobile.

Couchbase Mobile lleva el poder de NoSQL al límite y está compuesto de tres componentes:

- Couchbase Lite, an embedded, NoSQL JSON Document Style database for your mobile apps
- Sync Gateway, an internet-facing synchronization mechanism that securely syncs data between mobile clients and server, and
- Couchbase Server, a highly scalable, distributed NoSQL database platform

El siguiente diagrama describe la arquitectura para un despliegue típico compuesto por Couchbase Lite, Couchbase Server SDKs, Sync Gateway y Couchbase Server.

![Couchbase](http://blog.davidpalazon.net/images/cbm-architecture800x600.png)

Sync Gateway está diseñado para proporcionar sincronización de datos para aplicaciones interactivas a gran escala en la web, móviles y de IoT. Entre sus características más importantes y más utilizadas está el control de acceso seguro.

Sync Gateway asegura un control de acceso seguro usando:

- Autenticación de usuario, lo que asegura que sólo los usuarios autorizados pueden conectarse a la Sync Gateway. Revise los temas de [Users and Roles](http://blog.davidpalazon.net/manual-couchbase-sync-users-and-roles/) | [Guia Autentificacion Usuarios](http://blog.davidpalazon.net/manual-couchbase-sync-authentication/)

- Enrutamiento de datos. Esto garantiza que los usuarios autorizados sólo puedan acceder a los documentos en los canales de la puerta de enlace de sincronización que les hayan sido asignados y sólo de acuerdo con sus privilegios asignados. Puede configurar esos privilegios para conferir acceso de lectura y/o escritura según sea necesario.
 
La lógica empresarial que subyace a la validación y autorización del acceso a los documentos es proporcionada por la función de sincronización personalizable.