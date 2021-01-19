---
title: "Spring Cloud Part 2"
date: 2021-01-06T21:18:30+01:00
draft: false
author: "Jeannory"
tags: ["6/Micro-services-spring-cloud-part-2"]
categories: ["6/Micro-services"]
---

## Micro-service Spring Cloud part 2 ##

*Article en cours de rédaction...*

### Introduction ###

A travers cet article, nous allons aborder un moyen pour réaliser des échanges entre 2 micro-services, tout en assurant la sécurisation des transactions.
Nous allons expliquer les patterns choreography-based saga et oubox.
Je vous mettrais un exemple d’implémentation avec le code source à travers un cas métier concret.

### Dépôts git ###

* user-service : [ici](https://gitlab.com/phou.jeannory/user-service)
* order-service : [ici](https://gitlab.com/phou.jeannory/order-service)

Les fonctions ont été développées sous la branch "dev/add-order-service".

---

## choreography-based saga ##

*source : [ici](https://chrisrichardson.net/post/sagas/2019/08/15/developing-sagas-part-3.html)*

### Problématique ###

Une caractéristique distinctive de l’architecture des microservices est que, pour garantir un couplage faible, les données de chaque service sont décorrélés. Contrairement à une application monolithique, vous n'avez plus une seule base de données que n'importe quel module de l'application peut mettre à jour. Par conséquent, l'un des principaux défis auxquels vous serez confronté est de maintenir la cohérence des données entre les services.

Pour ce faire nous allons utiliser implémenter "choreography-based saga".

### Cas métier ###

Illustrons cela par un exemple, celui d’un cas métier d’une commande client.
L’utilisateur passe une commande, s’il dispose du montant suffisant sur son compte alors la commande est approuvée, et le montant est déduit de son compte; sinon la commande est refusé.
Dans le cas de notre micro-service le service order-service s’occupera exclusivement d’enregistrer les commandes, et user-service s’occupera de gérer le compte de l’utilisateur.

### Etapes ###

| Etape | Déclencheur | Participant | Commande | Action | Compensation |
|--:|---------|---------|---------|:--:|:----|
| 1 | | order-service | createPendingOrder() | Enregistrement d'une nouvelle commande | commande en erreur |
| 2 | Enregistrement d'une nouvelle commande | user-service | reserveCredit() | approvisionnenement suffisant: déduction du montant ||
| 2 | Enregistrement d'une nouvelle commande | user-service | reserveCredit() | approvisionnenement insuffisant: rejet de la commande ||
| 3a | déduction du montant | order-service | approveOrder() | enregistre la commande ||
| 3b | rejet de la commande | order-service | rejectOrder() | rejette la commande ||

* Etape 1 : order-service va enregistrer une nouvelle commande dans sa base de donnée.
Il retourne au client (réponse de la requête http) que la cliente est en attente. Il transmet l’information à user-service.
La compensation est le cas ou un problème est survenu pendant l’étape. Dans ce cas nous retournons au client une erreur serveur (500) en lui demandant de ressayer.
* Etape 2a : user-service reçoit la demande pour un utilisateur donné. Il va vérifier si le client dispose des fonds nécessaire sur son compte. Le compte est suffisamment approvisionné. user-service déduit le montant du compte utilisateur (reserveCredit()) et transmet l’information à order-service.
* Etape 2b : user-service reçoit la demande pour un utilisateur donné. Il va vérifier si le client dispose des fonds nécessaire sur son compte. Le compte n’est pas suffisamment approvisionné. user-service va transmettre cette information à order-service.
* Etape 3a : order-service va enregistrer dans sa base que la commande est validée.
* Etape 3b : order-service va enregistrer dans sa base que la commande est rejetée.

### Orchestration ###

A travers cet exemple nous vous montrerons un exemple d'implémentation sur un projet springcloud.

*Dans notre application, les micro-services order-service et user-service seront chacun composés de 3 instances. Nous ne ferons intervenir qu’un front-end et un gateway pour l’explication.*
Nous allons utiliser le messaging middleware activemq pour router les messages entre micro-services. Chaque micro service pourra produire topic (pouvant être consommé/récupéré par tous les micro-service qui sont à son écoute) ou queue (ne peut être consommé/récupéré uniquement par le 1er micro-service qui l’aura intercepté).
J’ai choisit activemq par sa simplicité. Pour une application en micro-service en entreprise, un messaging middleware sur un système distribué serait plus adapté (kafka, rabbitmq, etc..) 

![schema](/blog/img/6/schema-choreography-micro-service-1.png)

* Etape 1 : Le client envoi la requête http au gateway qui ira router vers l’instance de order-service disponible.
* Etape 2 : order-service energistre la nouvelle commande dans sa base de donnée. Il envoie ensuite un topic demandant à toutes les instances de order-service de faire la même sauvegarde. Toutes les instances ont ainsi le même niveau d’information.
* Etape 3 : order-service envoie une queue pour informer à une des instances de user-service qu’une commande est en attente. 
* Etape 4a : user-service va vérifier pour ce client s’il dispose des fonds nécessaires. Le compte est suffisamment approvisionné. Il déduit le montant du compte client dans sa base de donnée et envoie un topic à toutes les instances de user-service pour qu’ils mettent à niveau leurs bdd.
* Etape 4b : user-service va vérifier pour ce client s’il dispose des fonds nécessaires. Le compte n’est pas suffisamment approvisionné.
* Etape 5 : user-service envoie un topic à toutes les instances de order-service pour que ces derniers enregistrent dans leurs bases si la commande a été approuvée ou rejetée.
* Etape 6 : Après un certain délai (1 seconde par exemple), le client réinterroge l’application. N’importe quelle instance de order-service pourra lui répondre si la commande a été approuvée ou rejetée.

### Implémentation ###

toDo


