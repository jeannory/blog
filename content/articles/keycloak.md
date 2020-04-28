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
Il est reconnu pour la protection qu'il propose en termes de cyber sécurité et aussi par les très nombreuses librairies/fonctionnalités qui permettent une utilisation étendue (Java, AngularJs, Typescript, Ldap, etc..). Beaucoup d'entreprises lui font confiance et lui délèguent la partie authentification/autorisation.
C'est un projet open source codé en java, qui fait preuve d'une grande maturité (version 9 à ce jour).

Mohamed Youssfi formateur java explique comment sécuriser des applications en micro-service avec Keycloak à travers 11 vidéos.

<!-- <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe src="https://www.youtube.com/watch?v=2DYWyE9jkEY" style="position: absolute; top: 0; left: 0; width: 50%; height: 75%; border:0;" allowfullscreen title="YouTube Video"></iframe>
</div> -->

### Vidéos de M. Mohamed Youssfi ###
Sécuriser des applications en micro-service avec Keycloak.

* [Part 1 - Basic Concepts](https://www.youtube.com/watch?v=2DYWyE9jkEY)
* [Part 2 - Distributed App Use Case](https://www.youtube.com/watch?v=hiu4uHjdWbY&t=35s)
* [Part 3 - Getting Started with Keycloak](https://www.youtube.com/watch?v=mFJSxX9d-Cg)
* [Part 4 - Spring MVC Application](https://www.youtube.com/watch?v=QksFYgRUxGQ&t=188s)
* [Part 5 - Spring Security Keycloak Adapter](https://www.youtube.com/watch?v=NvpbR_wvPlQ&t=6s)
* [Part 6 - Micro-Service Rest API Bearer Only case](https://www.youtube.com/watch?v=grvDI2kvgsc&t=4s)
* [Part 7 - Angular Application to Secure](https://www.youtube.com/watch?v=Seq2qCCwxZ0&t=37s)
* [Part 8 - Keycloak js in Angular App](https://www.youtube.com/watch?v=5n0DhOdLxPc&t=29s)
* [Part 9 - Http Authorization Header Interceptor](https://www.youtube.com/watch?v=nIm78BIIeFE&t=27s)
* [Part 10 - Back to some improvements](https://www.youtube.com/watch?v=ObHhIvSILOs&t=21s)
* [Part 11 - Other Concepts Themes](https://www.youtube.com/watch?v=13vIJR8vT3w&t=110s)

---

Exemple d'implémentation, avec des tests et une application entièrement portable en docker.

### Dépôts Gitlab ###

* [Back-end](https://gitlab.com/phou.jeannory/keycloak-back-end.git)
* [Front-end] (https://gitlab.com/phou.jeannory/keycloak-front-end.git)

---

### Projet : spécificités techniques ###

* Back-end en java avec Springboot
* Front-end en Angular Material
* Docker de la base de données Postgresql
* Docker du serveur keycloak
* Docker de la base de données Mysql de keycloak
* Tests unitaires
* Tests d'intégration

### Projet : spécificités fonctionnelles ###

* L'utilisateur peut se connecter/déconnecter
* L'utilisateur peut créer un compte
* L'utilisateur peut accéder/modifier son profil
* L'utilisateur disposant du rôle manager peut accéder à la liste de tous les utilisateurs

---

## Le projet back-end ##

toDo

---

### L'implémentation de Keycloak ###

toDo

---

### Application 100 % portable avec docker ###

Toute l'infrastructure de l'application fonctionne sous docker. Elle est donc portable d'un environnement à un autre.
Elle comprend l'application principale, sa base de données PostgreSql, le serveur d'authentification Keycloak et sa base de données MySql.

---

### Exemple de déploiement sur une VM distante ###

Se connecter à la VM.

    ssh -p 18380 digital@jeannory.dynamic-dns.net

---

Build les images de Postgresql, Keycloak et sa bdd (Mysql).

Le répertoire /home/digital/keycloak-project/docker-back-end-environment/ a le même contenu que le répertoire du même nom du projet.

    cd /home/digital/keycloak-project/docker-back-end-environment
    docker-compose -f docker-compose.yml up -d

---

Vérifier que les images ont bien été déployées.

    docker ps -a

---

    CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS                             PORTS                                                                    NAMES
    de2face102da        jboss/keycloak       "/opt/jboss/tools/do…"   22 seconds ago      Up 16 seconds                      0.0.0.0:8443->8443/tcp, 0.0.0.0:9990->9990/tcp, 0.0.0.0:8099->8080/tcp   keycloak-dev-docker
    2909874a449a        postgres             "docker-entrypoint.s…"   33 seconds ago      Up 24 seconds                      0.0.0.0:5432->5432/tcp                                                   postgresql-dev-docker
    1cef41fa3f93        mysql/mysql-server   "/entrypoint.sh mysq…"   33 seconds ago      Up 23 seconds (health: starting)   0.0.0.0:3306->3306/tcp, 33060/tcp                                        mysql-dev-docker

---

Récupérer l'ip de l'image mysql-dev-docker.

    docker inspect mysql-dev-docker

---

    "NetworkID": "909b56f4c2caa1c594f8d691e0204cc236849fb25ddb2302bb75fcba516a8698",
    "EndpointID": "21d1c92defc16198b971ecf87be244bcff13c3e8bb5750753c8d2c428a85685e",
    "Gateway": "172.25.0.1",
    "IPAddress": "172.25.0.3",

---

Restaurer la base de données Mysql.

    cat backup.sql | docker exec -i mysql-dev-docker /usr/bin/mysql -h '172.25.0.3' -u keycloak -ppassword keycloak

---

Un warning indique qu'il n'est pas recommandé de renseigner le password dans une ligne de commande...

    mysql: [Warning] Using a password on the command line interface can be insecure.

---

Vérifier que la sauvegarde a bien été réalisée.

    cd /home/digital/keycloak-project/target
    ls

---

    keycloak-dev-docker  mysql-dev-docker

---

Si soucis de lancement de keycloak apres la restauration de la base de données.

    CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS                          PORTS                               NAMES
    c3246b20fde4        jboss/keycloak       "/opt/jboss/tools/do…"   3 minutes ago       Restarting (1) 11 seconds ago                                       keycloak-dev-docker
    99c412204936        postgres             "docker-entrypoint.s…"   3 minutes ago       Up 3 minutes                    0.0.0.0:5432->5432/tcp              postgresql-dev-docker
    ed42957b5ff5        mysql/mysql-server   "/entrypoint.sh mysq…"   3 minutes ago       Up 3 minutes (healthy)          0.0.0.0:3306->3306/tcp, 33060/tcp   mysql-dev-docker

---

Il faut arrêter/supprimer les images.

    docker stop keycloak-dev-docker
    docker rm keycloak-dev-docker
    docker stop postgresql-dev-docker
    docker rm postgresql-dev-docker
    docker stop mysql-dev-docker
    docker rm mysql-dev-docker
    docker system prune -a

---

    Total reclaimed space: 1.339GB

---

Rebuild les images.

    cd /home/digital/keycloak-project/docker-back-end-environment
    docker-compose -f docker-compose.yml up -d

---

En relancant la commande docker ps -a nous constatons que tout est rentré dans l'ordre.

    CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS                             PORTS                                                                    NAMES
    97b1d64bf536        jboss/keycloak       "/opt/jboss/tools/do…"   25 seconds ago      Up 18 seconds                      0.0.0.0:8443->8443/tcp, 0.0.0.0:9990->9990/tcp, 0.0.0.0:8099->8080/tcp   keycloak-dev-docker
    3b9e6f4a82e3        postgres             "docker-entrypoint.s…"   38 seconds ago      Up 23 seconds                      0.0.0.0:5432->5432/tcp                                                   postgresql-dev-docker
    2bd8051a1850        mysql/mysql-server   "/entrypoint.sh mysq…"   38 seconds ago      Up 24 seconds (health: starting)   0.0.0.0:3306->3306/tcp, 33060/tcp                                        mysql-dev-docker

--

### Les profils maven et Springboot ###

toDo

### Les tests ###

toDo

---
