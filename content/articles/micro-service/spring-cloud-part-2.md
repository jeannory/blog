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

* source : [ici](https://chrisrichardson.net/post/sagas/2019/08/15/developing-sagas-part-3.html)

*Problématique*

Une caractéristique distinctive de l’architecture des microservices est que, pour garantir un couplage faible, les données de chaque service sont décorrélés. Contrairement à une application monolithique, vous n'avez plus une seule base de données que n'importe quel module de l'application peut mettre à jour. Par conséquent, l'un des principaux défis auxquels vous serez confronté est de maintenir la cohérence des données entre les services.

Pour ce faire nous allons utiliser implémenter "choreography-based saga".

Illustrons cela par un exemple, celui d’un cas métier d’une commande par un utilisateur.
L’utilisateur passe une commande, s’il dispose du montant suffisant sur son compte alors la commande est approuvé, et le montant est déduit de son compte; sinon la commande est refusé.
Dans le cas de notre micro-service le service order-service s’occupera exclusivement d’enregistrer les commandes, et user-service s’occupera de gérer le compte de l’utilisateur.

---

| Etape | Déclencheur | Participant | Commande | Action | Compensation |
|--:|---------|---------|---------|:--:|:----|
| 1 | | order-service | createPendingOrder() | Enregistrement d'une nouvelle commande | commande en erreur |
| 2 | Enregistrement d'une nouvelle commande | user-service | reserveCredit() | approvisionnenement suffisant: déduction du montant | commande en erreur |
| 2 | Enregistrement d'une nouvelle commande | user-service | reserveCredit() | approvisionnenement insuffisant: rejet de la commande | commande en erreur |
| 3a | déduction du montant | order-service | approveOrder() | enregistre la commande | commande en erreur |
| 3b | rejet de la commande | order-service | rejectOrder() | rejette la commande | commande en erreur |

A la 1ere étape, order-service va enregistrer une nouvelle commande dans sa base de donnée et envoyer une commande en attente vers user-service.
Il retourne au client (réponse de la requête http) que la commande est en cours.
La compensation est le cas ou un problème est survenu pendant l’étape. Dans ce cas nous retournons au client une erreur serveur (statut code 500) en lui demandant de réssayer.

---

## choreography-based saga schéma ##

![schema](/blog/img/6/schema-choreography-micro-service-1.png)

---


