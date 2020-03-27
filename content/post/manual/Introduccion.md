---
title: "Manual (Traducido) Couchbase Sync"
date: 2020-03-03T16:15:45+01:00
draft: false
slug: "manual-couchbase-sync"
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

![Couchbase](http://blog.davidpalazon.net/images/cbm-architecture.png)

