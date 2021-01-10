---
title: "Spring Cloud Part 2"
date: 2021-01-06T21:18:30+01:00
draft: false
author: "Jeannory"
tags: ["6/Micro-services-spring-cloud-part-2"]
categories: ["6/Micro-services"]
---

## Micro-service Spring Cloud part 2 ##

Nous allons désormais faire communiquer 2 services à travers un exemple très simple.

Pour ce faire nous allons utiliser implémenter "choreography-based saga" tel que définit [ici](https://chrisrichardson.net/post/sagas/2019/08/15/developing-sagas-part-3.html).

---

| Step | Triggering event | Participant | Command | Events |
|--:|---------|---------|:--:|:----|
| 1 | | Order Service | createPendingOrder() | OrderCreated |
| 2 | OrderCreated | Customer Service | reserveCredit() |Credit Reserved, Credit Limit Exceeded |
| 3a | Credit Reserved | Order Service | approveOrder() |
| 3b | Credit Limit Exceeded | Order Service | rejectOrder() |

---

Voici comment les services vont communiquer entre eux par cette implémentation:

---

![schema](/blog/img/6/schema-choreography-micro-service-1.png)

---


