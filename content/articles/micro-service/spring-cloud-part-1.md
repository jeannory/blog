---
title: "Spring Cloud Part 1"
date: 2020-10-30T09:05:35+01:00
draft: false
author: "Jeannory"
tags: ["6/Micro-services-spring-cloud-part-1"]
categories: ["6/Micro-services"]
---

## Micro-service Spring Cloud part 1 ##

## Le projet ##

A travers ce projet, je vais illustrer un exemple d'implémentation d'un micro service avec Spring-Cloud. Ce projet va être déployé sur plusieurs VM du cloud de googl-cloud-plateform. Il va s'agir de la 1ère partie ou sera développé un seul service sur plusieurs instances.

```
Objectif
Schéma de'infrastructure du projet réalisé.
```

![objectif](/blog/img/6/Micro-service-1.png)

---

```
Projet de base
```
Reprise du projet du paragraphe [Jms Websocket]({{< ref "/articles/jms-websocket.md" >}} "Jms Websocket")
![schéma-actuel](/blog/img/6/Micro-service-2.png)

---

Dépôts

Ce projet a nécessité 8 dépôts gitlab.

* active-mq : [ici](https://gitlab.com/phou.jeannory/active-mq)
* configuration : [ici](https://gitlab.com/phou.jeannory/configuration)
* configuration-service : [ici](https://gitlab.com/phou.jeannory/configuration-service)
* registartion-service : [ici](https://gitlab.com/phou.jeannory/registration-service)
* proxy-service : [ici](https://gitlab.com/phou.jeannory/proxy-service)
* user-service : [ici](https://gitlab.com/phou.jeannory/user-service)
* spam-service : [ici](https://gitlab.com/phou.jeannory/spam-service)
* front-end-server : [ici](https://gitlab.com/phou.jeannory/front-end-server)

---

Et 10 VM ont été utilisé pour tous les déploiements.

![gcp](/blog/img/6/gcp.png)

---


## Configuration-service ##

Configuration service va servir à mapper les fichiers de configurations vers les services. Les fichiers de configurations sont centralisés vers le dépôt git [configuration](https://gitlab.com/phou.jeannory/configuration).
Par exemple, un fichier de configuration peut contenir le port de déployement, les différents profils, les connecteurs de la base de données, du serveur d'authentification, etc...


    server:
    port: 8081

    logging:
    file: logs/user-service.log
    pattern:
        console: "%d %-5level %logger : %msg%n"
        file: "%d %-5level [%thread] %logger : %msg%n"
    level:
        org.springframework.web: ERROR
        com.howtodoinjava: DEBUG
        org.hibernate: ERROR

    ---

    spring:
    profiles: dev-docker

    jpa:
        database: POSTGRESQL
        show-sql: true
        hibernate:
        ddl-auto: create-drop

    datasource:
        platform: postgres
        url: jdbc:postgresql://localhost:5432/keycloak_back_end_db
        username: user1
        password: 1234!*MyJea@99
        driverClassName: org.postgresql.Driver
        initialization-mode: always
    ...

---

configuration-service est un module SpringCloud.
Il a besoin des dépendances suivants:

	<properties>
		<java.version>1.8</java.version>
		<spring-cloud.version>Hoxton.SR8</spring-cloud.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
			<exclusions>
				<exclusion>
					<groupId>org.junit.vintage</groupId>
					<artifactId>junit-vintage-engine</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

---

Son fichier application.yml devra obligatoirement indiquer le lien vers le git de configuration (il peut aussi s'agir d'un répertoire si vous essayez en local, dans ce cas remplacer https:// par file:)

    server:
        port: 8888

    spring:
        name: "configuration-service"
            profiles:
                active: "dev-docker"
            cloud:
                config:
                    server:
                        git:
                            uri: https://gitlab.com/phou.jeannory/configuration.git

    ---

    spring:
        profiles: dev-docker
        cloud:
            config:
                server:
                    git:
                        uri: file:/home/jeannory/gitlab/keycloak/local-configuration

---

Il faut également ajouter l'annotation EnableConfigServer

    @EnableConfigServer
    @SpringBootApplication
    public class ConfigurationServiceApplication {

---

## Configuration ##

C'est le dépôt git qui va contenir toutes les configurations des différents services.

![configuartion](/blog/img/6/configuration.png)

---

## Registartion-service ##

C'est le service qui va enregistrer tous les services du projet. Ces informations seront affichés dans sa console.
Ici nous voyons qu'il y a proxy-service, spam-service et user-service qui lui comporte 3 instances.

![console-eureka](/blog/img/6/eureka-1.png)

---

Dépendances

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
			<exclusions>
				<exclusion>
					<groupId>org.junit.vintage</groupId>
					<artifactId>junit-vintage-engine</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

---

Annotation à rajouter dans le main

    @EnableEurekaServer
    @SpringBootApplication
    public class RegistrationServiceApplication {

        public static void main(String[] args) {
            SpringApplication.run(RegistrationServiceApplication.class, args);
        }

    }

---

Il faut pour ce service aussi renseigner bootstrap.yml vers configuration-service

    spring:
        name: "registration-service"
        application:
            name: registration-service
        profiles:
            active: "dev-docker"

        main:
            banner-mode: "off"

    ---

    spring:
        profiles: dev-docker
        cloud:
            config:
                uri: http://localhost:8888

    ---

    spring:
        profiles: dev-docker-integration-deploy #deployment
        cloud:
            config:
                uri: http://10.156.0.2:8888

---

Qui va mapper vers la configuration (registration-service.yml) dans le dépôt git

    server:
        port: 8761

    eureka:
        client:
            fetch-registry: false
            register-with-eureka: false


---

## Proxy-service ##

Ce service va servir de gateway entre les appels client http et les différents services. Il sert également d'équilibreur de charge : dans le cas ou un service comporte plusieurs instances, proxy-service va décider quelle instance sera sollicitée pour une même requête.

Dépendances

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
			<exclusions>
				<exclusion>
					<groupId>org.junit.vintage</groupId>
					<artifactId>junit-vintage-engine</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

---

Annotations à ajouter dans le main

    @EnableDiscoveryClient
    @EnableZuulProxy
    @SpringBootApplication
    public class ProxyServiceApplication {

        public static void main(String[] args) {
            SpringApplication.run(ProxyServiceApplication.class, args);
        }

    }

---

Il faut pour ce service aussi renseigner bootstrap.yml vers configuration-service

    spring:
        application:
            name: proxy-service
        profiles:
            active: "dev-docker"

    main:
        banner-mode: "off"

    ---

    spring:
        profiles: dev-docker
        cloud:
            config:
                uri: http://localhost:8888

    ---

    spring:
        profiles: dev-docker-integration-deploy #deployment
        cloud:
            config:
                uri: http://10.156.0.2:8888

---

Qui va mapper vers la configuration (proxy-service.yml) dans le dépôt git.
Pour le mapping des api, la requête doit pointer vers l'url du service-proxy + service appelé + api. Exemple si nous souhaitons appeler la requete de user-service http://35.208.214.143:8081/api/test, il faut lancer la requête suivante http://35.214.184.168:9999/user-service/api/test. Il est possible de personnaliser les appels.
Dans la configuration suivante, l'appel devient http://35.214.184.168:9999/user/api/test.
Zuul prend en charge les redirections de sécurité. Il est possible de le définir de manière globale ou pour un ou plusieurs services (ajouter sensitiveHeaders: Cookie,Set-Cookie).
Dans le cas de multiples instances, il n'y a pas besoin de déclarations spécifique.

    server:
        port: 9999

    logging:
        file: logs/proxy-service.log
        pattern:
            console: "%d %-5level %logger : %msg%n"
            file: "%d %-5level [%thread] %logger : %msg%n"
        level:
            org.springframework.web: ERROR
            com.howtodoinjava: DEBUG
            org.hibernate: ERROR

    zuul:
        routes:
            user:
                path: /user/** # request pre-path
                serviceId: user-service # mapping service
                sensitiveHeaders: Cookie,Set-Cookie # add bearer on header

    # Disable Hystrix timeout globally (for all services)
    hystrix:
        command:
            default:
                execution:
                    timeout:
                        enabled: false

    ---

    spring:
        profiles: dev-docker

    eureka:
        client:
            serviceUrl:
                defaultZone: http://localhost:8761/eureka/

    ---

    spring:
        profiles: dev-docker-integration-deploy #deployment

    eureka:
        client:
            serviceUrl:
                defaultZone: http://10.166.0.3:8761/eureka/

---


## User-service ##

Enfin le dernier service qui va nous servir pour tester les micro-services.

Dépendances


        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

---

Il faut rajouter les annotations EnableDiscoveryClient et EnableConfigurationProperties

    @EnableDiscoveryClient
    @EnableConfigurationProperties
    @SpringBootApplication
    public class UserServiceApplication {

---

Renommer le fichier application.yml en bootstrap.yml et lui indiquer, comment va se nommer le service (ici c'est application: name: user-service) et ou se trouve le serveur de configuration (http://localhost:8888 ou http://34.107.109.238:8888 en fonction du profile). 


    spring:
        name: "user-service"
        application:
            name: user-service
        profiles:
            active: "dev-docker"

        main:
            banner-mode: "off"

    lombok:
        anyConstructor:
            addConstructorProperties: true

    ---

    spring:
    profiles: dev-docker
    cloud:
        config:
        uri: http://localhost:8888

    ---

    spring:
        profiles: dev-docker-integration #integration test
        cloud:
            config:
                uri: http://35.207.169.107:8888

    ---

    spring:
        profiles: dev-docker-integration-deploy #deployment
        cloud:
            config:
                uri: http://10.156.0.2:8888

---

Configuration-service fera ainsi le mapping grâce au nommage (fichier user-service.yml dans le serveur git configuration).

    server: #to comment on local
        port: 8081

    logging:
        file: logs/user-service.log
        pattern:
            console: "%d %-5level %logger : %msg%n"
            file: "%d %-5level [%thread] %logger : %msg%n"
        level:
            org.springframework.web: ERROR
            com.howtodoinjava: DEBUG
            org.hibernate: ERROR

    ---

    spring:
        profiles: dev-docker

---

### Actuator ###

Spring Boot Actuator est un sous projet du projet Spring Boot. Il est créé pour collecter, superviser les informations de l'application par un ensemble de endpoints fournits.

Son utilisation est très simple, il suffit juste de rajouter la dépendance et d'indiquer son port d'exposition.

Un exemple d'utilisation sera illustré avec user-service.

---

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

---

Configuration de actuator

---

    management: #actuator not in use for integration test
        server:
            port: 8080 #0 random port on local if more than 1 service else 8080
        endpoints:
            web:
                exposure:
                    include: "*"
            shutdown:
                enabled: true

---

Dans le fichier de configuration central, le port d'exposition en local doit être en random (soit 0), si plus d'une instance de user-service est déployé. En effet 2 instances différents ne peuvent pas exposer un service sur le même port en local!

Dans cet exemple nous allons ouvrir le port 8080 ou une des instances de user-service est déployé (ip 35.208.214.143) 

Ne pas oublier d'ajouter le tag réseau qui va autoriser l'exposition du port 8080 sur le web!

![gcp-2](/blog/img/6/gcp-2.png)

L'url http://35.208.214.143:8080/actuator/ va lister tous les end-points disponibles.

http://35.208.214.143:8080/actuator/beans pour connaître tous les beans chargés en mémoire.

http://35.208.214.143:8080/actuator/env/ va indiquer l'environnement du service.

Son utilisation est donc très intuitive.


### Requête simple ###

Etapes d'un appel simple :

* 1/Le client lance une requête vers le proxy-service
* 2/Le proxy-service va router vers une instance disponible (instance 1)
* 3/Le serveur d'authentification va vérifier les informations de sécurité du header et authoriser la requête
* 4/L'instance 1 va retourner la response au proxy-service
* 5/Proxy service va retourner le résultat de la requête au client

Il s'agit d'un exemple très simple. Le service-proxy a la possibilité de faire traitement lorsqu'il recoit une requête (filtre, traitement avant la response, etc...)

![Etapes](/blog/img/6/Micro-service-3.png)

---

### Mise à jour des bases de données des différentes instances ###

Les instances disposent d'une propre base de données. Il faut en conséquence pouvoir gérer lors d'insertions ou de modifications de données.
Une des solutions consiste à utiliser un messaging middleware:

* 1/Dans notre exemple, l'instance 1 va faire une insertion dans la base de donnée.
* 2/L'instance 1 va envoyer un producer dans un topic. Il va contenir l'objet qui a été persisté.
* 3/Toutes les instances disposent d'un listener de ce même topic, et vont le consommer (récupérer l'objet)
* 4/Ils vont se mettre à niveau en faisant également l'insertion

![update-database](/blog/img/6/Micro-service-4.png)

```
Implémentation
```

Dépendance

        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-broker</artifactId>
        </dependency>

---

Configuration

    @Configuration
    @EnableJms
    public class JmsConfig {

        @Value("${activemq.broker-url}")
        private String brokerUrl;

        @Bean
        public DefaultJmsListenerContainerFactory userFactory() {
            DefaultJmsListenerContainerFactory containerFactory = new DefaultJmsListenerContainerFactory();
            containerFactory.setPubSubDomain(true);
            containerFactory.setConnectionFactory(connectionFactory());
            containerFactory.setMessageConverter(jacksonJmsMessageConverter());
            return containerFactory;
        }
        @Bean
        public CachingConnectionFactory connectionFactory() {
            CachingConnectionFactory cachConnectionFactory = new CachingConnectionFactory();
            ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory();
            connectionFactory.setBrokerURL(brokerUrl);
            cachConnectionFactory.setTargetConnectionFactory(connectionFactory);
            return cachConnectionFactory;
        }
        @Bean
        public MessageConverter jacksonJmsMessageConverter() {
            MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
            converter.setTargetType(MessageType.TEXT);
            converter.setTypeIdPropertyName("_type");
            return converter;
        }

---

Création du topic

    private static Topic userTopic;

    public static void main(String[] args) throws JMSException {
        final ConfigurableApplicationContext context = SpringApplication.run(UserServiceApplication.class, args);
        final JmsTemplate jmsTemplate = context.getBean(JmsTemplate.class);
        userTopic = jmsTemplate.getConnectionFactory().createConnection()
                .createSession().createTopic(Constant.TOPIC_USER_SERVICE_SAVE_USER);
    }

    public static Topic getUserTopic() {
        return userTopic;
    }

---

Envoie du topic. Pour l'utiliser on va passer en argument le topic userTopic et l'objet à persister

    public T pushOnJms(final Topic topic, final T t) throws JMSException {
        try {
            jmsTemplate.convertAndSend(topic, t);
            jmsTemplate.setReceiveTimeout(10_000);
            return t;
        }catch (Exception ex){
            throw new CustomJmsException("Message cannot be sent : " + ex.getMessage());
        }
    }

---

Le listener va récupérer les objets du topic pour persister l'objet.

        @org.springframework.jms.annotation.JmsListener(destination = Constant.TOPIC_USER_SERVICE_SAVE_USER, containerFactory = "userFactory")
        public void saveUserByConsumer(Message message) throws JMSException, JsonProcessingException {
            final ActiveMQTextMessage amqMessage = (ActiveMQTextMessage) message;
            final String messageResponse = amqMessage.getText();
            User user = singletonBean.getObjectMapper().readValue(messageResponse, User.class);
            try{
                user = testCreateUser(user);
            }catch (CustomTransactionalException ex){
                System.out.println(ex.getMessage());
            }
            /**
            * toDo do something with return
            */

            System.out.println("user return " + user.toString());
        }

Cette méthode comporte plusieurs points de vigilance

* Les id auto-incrémenté vont se décaler entre les bases de données des différentes instances
* L'instance qui va produire va également consommer (Exception à gérer en cas de re-persistence!)
* Vérifier que les bases de données de toutes les instances comportent tous les mêmes éléments

## Conclusion ##

Il est très simple avec Spring-Cloud d'implémenter un projet en micro-service. Elle prend également en charge le système de sécurité OpenID. Avec l'intégration continue et docker il est également très aisée de scaler les services, et même de changer de vm.






