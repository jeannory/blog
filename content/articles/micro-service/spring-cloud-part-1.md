---
title: "Spring Cloud Part 1"
date: 2020-10-30T09:05:35+01:00
draft: false
author: "Jeannory"
tags: ["6/Micro-services-spring-cloud-part-1"]
categories: ["6/Micro-services"]
---

## Micro-service Spring Cloud part 1 ##

### Le projet ###

```
Etat actuel
```

![schéma-actuel](/blog/img/6/Micro-service-1.png)

---

```
Objectif
```

![objectif](/blog/img/6/Micro-service-2.png)

---
Dépôts

* configuration-service : [ici](https://gitlab.com/phou.jeannory/configuration-service)
* configuration : [ici](https://gitlab.com/phou.jeannory/configuration)
* registartion-service : [ici](https://gitlab.com/phou.jeannory/registration-service)
* user-service : [ici](https://gitlab.com/phou.jeannory/user-service)
* spam-service : [ici](https://gitlab.com/phou.jeannory/spam-service)
* front-end-server : [ici](https://gitlab.com/phou.jeannory/front-end-server)

### Configuration-service ###

Configuration service va servir à mapper les fichiers de configurations vers les services. Les fichiers de configurations sont centralisés dans un dépôt git [ici](https://gitlab.com/phou.jeannory/configuration).
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

Son fichier application.yml devra obligatoirement indiquer le lien vers le git de configuration (il peut aussi s'agir d'un répertoire si vous essayer en local, dans ce cas remplacer https:// par file:)

    server:
        port: 8888

    spring:
        cloud:
            config:
                server:
                    git:
                        uri: https://gitlab.com/phou.jeannory/configuration.git

---

Il faut également ajouter l'annotation EnableConfigServer

    @EnableConfigServer
    @SpringBootApplication
    public class ConfigurationServiceApplication {

---


### User-service ###

user-service va utliser configuration-service pour récupérer sa configuration


        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>

---

Il faut rajouter l'annotation EnableConfigurationProperties

    @EnableConfigurationProperties
    @EnableJms
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
                uri: http://34.107.109.238:8888

    ---

    spring:
        profiles: dev-docker-integration-deploy #deployment
        cloud:
            config:
                uri: http://34.107.109.238:8888

---

Configuration-service fera ainsi le mapping grâce au nommage (fichier user-service.yml dans le serveur git configuration).

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

---

Actuator renseignera sur des informations sur user-service et permet que les données actualisés du fichier de configuration puisse aussi l'être pour le server (exemple à suivre)

    @Controller
    public class MainController {

        @ResponseBody
        @RequestMapping(path = "/")
        public String home(HttpServletRequest request) {

            String contextPath = request.getContextPath();
            String host = request.getServerName();

            // Spring Boot >= 2.0.0.M7
            String endpointBasePath = "/actuator";

            StringBuilder sb = new StringBuilder();

            sb.append("<h2>Sprig Boot Actuator</h2>");
            sb.append("<ul>");

            // http://localhost:8090/actuator
            String url = "http://" + host + ":8090" + contextPath + endpointBasePath;

            sb.append("<li><a href='" + url + "'>" + url + "</a></li>");

            sb.append("</ul>");

            return sb.toString();
        }

    }

---
Configuration de actuator
boostrap.yml

    management:
        server:
            port: 0
        endpoints:
            web:
                exposure:
                    include: "*"
            shutdown:
                enabled: true

---

### Registartion-service ###

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

Une fois démarrer il faut lancer la console qui va afficher des informations essentiels.
Celles qui nous intéreressent ici sont les instances enregistrées. 

![console-eureka](/blog/img/6/eureka-1.png)

---

### Proxy-service ###

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
Zuul sensitiveHeaders: Cookie,Set-Cookie permet que la requête comporte le header d'origine. Nécessaire en cas de la vérification du token de session jwt dans le header.

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

    zuul: # Pass Authorization header downstream
        sensitiveHeaders: Cookie,Set-Cookie

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

Sur le projet user-service, nous allons déployer la même instance sur 2 vm différentes

    deploy-app-instance-1:
        stage: deploy-app-instance-1
        image: ubuntu:18.04
        before_script:
            - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client -y)'
            - mkdir -p ~/.ssh
            - echo "$USER_APP_KEY" | tr -d '\r' > ~/.ssh/id_rsa
            - chmod 700 ~/.ssh/id_rsa
            - eval "$(ssh-agent -s)"
            - ssh-add ~/.ssh/id_rsa
            - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
        script:
            - ssh-copy-id phou_jeannory@35.208.214.143
            - ssh -t phou_jeannory@35.208.214.143 << END
            - pwd
            - sudo docker stop devdockerintegrationapp_java_1 # stop and delete the image of app
            - sudo docker rm devdockerintegrationapp_java_1
            - echo "y" | sudo docker system prune -a
            - sudo docker login registry.gitlab.com -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" # build new image of app from container registry
            - cd /home/phou_jeannory/dev-docker-integration-app
            - sudo docker-compose -f docker-compose.yml up -d # name of image is the folder + java_1 ==> devdockerintegrationapp_java_1
            - sudo docker ps -a
            - sudo docker images ls
            - END
        only:
            - develop

    deploy-app-instance-2:
        stage: deploy-app-instance-2
        image: ubuntu:18.04
        before_script:
            - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client -y)'
            - mkdir -p ~/.ssh
            - echo "$USER_APP_KEY" | tr -d '\r' > ~/.ssh/id_rsa
            - chmod 700 ~/.ssh/id_rsa
            - eval "$(ssh-agent -s)"
            - ssh-add ~/.ssh/id_rsa
            - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
        script:
            - ssh-copy-id phou_jeannory@35.216.225.110
            - ssh -t phou_jeannory@35.216.225.110 << END
            - pwd
            - sudo docker stop devdockerintegrationapp_java_1 # stop and delete the image of app
            - sudo docker rm devdockerintegrationapp_java_1
            - echo "y" | sudo docker system prune -a
            - sudo docker login registry.gitlab.com -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" # build new image of app from container registry
            - cd /home/phou_jeannory/dev-docker-integration-app
            - sudo docker-compose -f docker-compose.yml up -d # name of image is the folder + java_1 ==> devdockerintegrationapp_java_1
            - sudo docker ps -a
            - sudo docker images ls
            - END
        only:
            - develop

Ainsi proxy-service va router les requêtes clients vers l'une des 2 instances de user-service. Il fait donc office d'équilibreur de charge.
Pour ce faire la requête devra comporter l'url de proxy-service + le nom du service demandé + l'api (exemple http://35.208.214.143:9999/user-service/api/users?pageNo=0&pageSize=10&sortBy=id).
_Pour la vérification du serveur keycloak ne pas oublier de rajouter dans Valid Redirect URIs l'url de proxy-service soit http://35.214.184.168:9999/*_
