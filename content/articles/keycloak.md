---
title: "Keycloak"
date: 2020-04-23T19:40:34+02:00
draft: false
author: "Jeannory"
tags: ["articles"]
categories: ["Keycloak"]
---
## Introduction ##

Keycloak est serveur d'authentification standalone.
Il est reconnu par la protection qu'elle propose en terme de cyber sécurité et de très nombreuses librairies permettent une utilisation étendue (Java, AngularJs, Typescript, Ldap, etc..). Beaucoup d'entreprise lui font confiance et délèguent la partie authentification/autorisation à Keycloak.
C'est un projet open source codé en java, qui fait preuve d'une grande maturité (version 9 à ce jour).

Mohamed Youssfi formateur java explique comment sécuriser des applications en micro service avec Keycloak à travers 11 vidéos.

<!-- <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe src="https://www.youtube.com/watch?v=2DYWyE9jkEY" style="position: absolute; top: 0; left: 0; width: 50%; height: 75%; border:0;" allowfullscreen title="YouTube Video"></iframe>
</div> -->

### Sécuriser des applications en micro service avec Keycloak - M. Mohamed Youssfi ###

* [Part 1 - Basic Concepts](https://www.youtube.com/watch?v=2DYWyE9jkEY)
* [Part 2 - Distributed App Use Case](https://www.youtube.com/watch?v=hiu4uHjdWbY&t=35s)
* [Part 3 - Getting Started with Keycloak](https://www.youtube.com/watch?v=mFJSxX9d-Cg)
* [Part 4 - Spring MVC Application](https://www.youtube.com/watch?v=QksFYgRUxGQ&t=188s)
* [Part 5 - Spring Security Keycloak Adapter](https://www.youtube.com/watch?v=NvpbR_wvPlQ&t=6s)
* [Part 6 - Micro Service Rest API Bearer Only case](https://www.youtube.com/watch?v=grvDI2kvgsc&t=4s)
* [Part 7 - Angular Application to Secure](https://www.youtube.com/watch?v=Seq2qCCwxZ0&t=37s)
* [Part 8 - Keycloak js in Angular App](https://www.youtube.com/watch?v=5n0DhOdLxPc&t=29s)
* [Part 9 - Http Authorization Header Interceptor](https://www.youtube.com/watch?v=nIm78BIIeFE&t=27s)
* [Part 10 - Back to some improvements](https://www.youtube.com/watch?v=ObHhIvSILOs&t=21s)
* [Part 11 - Other Concepts Themes](https://www.youtube.com/watch?v=13vIJR8vT3w&t=110s)

---

Après avoir suivi cette formation, je vous propose un exemple d'implémentation.
Le projet va comporter des tests, avec un environnement entièrement portable.

---

## Projet : spécificités ##

* Back-end en java avec Springboot
* Front-end en Angular
* Docker de la base de données Postgresql
* Docker du serveur keycloak
* Docker de la base de donnée Mysql de keycloak
* Tests unitaires
* Tests d'intégrations

---
