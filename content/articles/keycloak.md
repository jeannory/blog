---
title: "Keycloak"
date: 2020-04-23T19:40:34+02:00
draft: false
author: "Jeannory"
tags: ["3/Keycloak"]
categories: ["3/Keycloak"]
---
## Introduction ##

*MAJ au 16/07/2020*

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

Exemple d'implémentation, avec des tests et une application entièrement portable grâce aux containers.

### Dépôts Gitlab ###

* [Back-end](https://gitlab.com/phou.jeannory/keycloak-back-end.git)
* [Front-end] (https://gitlab.com/phou.jeannory/keycloak-front-end.git)

---

### Projet : spécificités techniques ###


* Serveur d'authentification
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

## Serveur d'authentification 100 % portable avec docker ##

Dans une application en micro service, le serveur d'authentification est décorélé des autres micro services. Le serveur Keycloak est déployé dans une VM.
Keycloak et ses dépendances sont buildés dans une même image docker. Ils sont donc portable d'un environnement à un autre.
Cette image comporte le serveur standalone de Keycloak, et sa base de données MySql (répertoire dev-docker-integration-keycloak):

docker-compose.yml

    version: '3.1'
    services:
        lab-mysql:
            container_name: mysql-dev-docker
            restart: always
            image: mysql/mysql-server
            environment:
                MYSQL_ROOT_PASSWORD: password
                MYSQL_ROOT_HOST: "%"
                MYSQL_DATABASE: keycloak
                MYSQL_USER: keycloak
                MYSQL_PASSWORD: password
            volumes:
                - ./../target/mysql-dev-docker:/var/lib/mysql
            ports:
                - 3306:3306
        keycloak:
            container_name: keycloak-dev-docker
            restart: always
            image: jboss/keycloak
            depends_on:
                - lab-mysql
            environment:
                DB_VENDOR: MYSQL
                DB_ADDR: lab-mysql
                DB_PORT: 3306
                DB_DATABASE: keycloak
                DB_USER: keycloak
                DB_PASSWORD: password
                KEYCLOAK_USER: admin
                KEYCLOAK_PASSWORD: 1234
            ports:
                - 8099:8080
                - 8443:8443
                - 9990:9990
            volumes:
                - ./../target/keycloak-dev-docker:/data
            links:
                - lab-mysql

---

### Déploiement de Keycloak dans une VM GCP ###

Il s'agit d'une petite VM sous Ubuntu Serveur (g1-small :1 vCPU, 1,7 Go de mémoire).
Il faut déployer l'image et restaurer la base de données:

    docker-compose -f docker-compose.yml up -d

---

Si nécessaire, récupérer l'ip de l'image mysql-dev-docker.

    docker inspect mysql-dev-docker

---

    "NetworkID": "909b56f4c2caa1c594f8d691e0204cc236849fb25ddb2302bb75fcba516a8698",
    "EndpointID": "21d1c92defc16198b971ecf87be244bcff13c3e8bb5750753c8d2c428a85685e",
    "Gateway": "172.25.0.1",
    "IPAddress": "172.25.0.3",

---

Restaurer la base de données Mysql.

    cd keycloak-config
    cat backup.sql | docker exec -i mysql-dev-docker /usr/bin/mysql -h '172.25.0.3' -u keycloak -ppassword keycloak

ou

    cd keycloak-config
    cat backup.sql | docker exec -i mysql-dev-docker /usr/bin/mysql -u keycloak -ppassword keycloak

*Cette configuration a un niveau de sécurité permettant l'exposition de l'Ihm avec le protocol http.*

---

### L'IHM de Keycloak ###

Assurez-vous que le port de la VM soit ouverte en entré sur le port définis pour keycloak (8089)
Se connecter à l'IHM :

    http://35.209.231.158:8099/auth/

Les crédentials figurent sur le docker-compose.yml

    KEYCLOAK_USER: admin
    KEYCLOAK_PASSWORD: 1234


---

![keycloak-realm](/blog/img/gitlab-15.png)

Le Realm correspond au royaume ou à l'entretprise.

---

![keycloak-clients](/blog/img/gitlab-16.png)

Les clients sont les applications de l'entreprise. Dans notre exemple il y a 2 clients qui vont être utilisés:

* sc-ui (Front-end)
* sc-user (Back-end)

---

![keycloak-roles](/blog/img/gitlab-17.png)

Les rôles peuvent être définis de manière globale ou pour une ou plusieurs applications (clients).

---

![keycloak-users](/blog/img/gitlab-18.png)

Les utilisateurs sont visibles dans l'IHM. Leurs rôles peuvent ainsi être définis.

---

![keycloak-installations](/blog/img/gitlab-19.png)

Ces éléments vont être utiles pour créer les connecteurs des applications

---

## Back-end en java avec Springboot ##

Les infos récupérés vont servir à renseigner le fichier de configuration de Springboot.

    {
    "realm": "Sc-project",
    "auth-server-url": "http://35.209.231.158:8099/auth/",
    "ssl-required": "none",
    "resource": "sc-user",
    "public-client": true,
    "confidential-port": 0
    }


Le nom de domaine http://jeannory.ovh pointe bien évidement vers l'ip externe http://35.209.231.158

Le profil dev-docker-integration va servir pour les étapes de CI/CD. Gitlab va se connecter à cette VM afin de réaliser tous les tests d'intégration.

    ---

    spring:
    profiles: dev-docker-integration

    jpa:
        database: POSTGRESQL
        show-sql: true
        hibernate:
        ddl-auto: create-drop

    datasource:
        platform: postgres
        url: jdbc:postgresql://jeannory.dynamic-dns.net:5432/keycloak_back_end_db
        username: user1
        password: 1234!*MyJea@99
        driverClassName: org.postgresql.Driver
        initialization-mode: always

    keycloak:
    realm: Sc-project
    auth-server-url: http://jeannory.ovh:8099/auth/
    resource: sc-user
    public-client: false
    bearer-only: true
    cors: true


    sc-properties:
    authServerUrl: http://jeannory.ovh:8099/auth/
    authServerParameters: realms/Sc-project/protocol/openid-connect/token

    ---

---

La redéfinition de la méthode *configure* permet une protection des end-points en fonction des rôles

    @KeycloakConfiguration
    public class KeycloakSpringSecurityConfig extends KeycloakWebSecurityConfigurerAdapter {

        @Override
        protected SessionAuthenticationStrategy sessionAuthenticationStrategy() {
            return new RegisterSessionAuthenticationStrategy(new SessionRegistryImpl());
        }

        @Override
        protected void configure(AuthenticationManagerBuilder auth) throws Exception{
            auth.authenticationProvider(new KeycloakAuthenticationProvider());
        }

        @Override
        protected void configure(HttpSecurity http)throws Exception{
            super.configure(http);
            http.authorizeRequests().antMatchers("/api/users/connected-user").hasAuthority(("user"));
            http.authorizeRequests().antMatchers("/api/users").hasAuthority(("manager"));
            http.authorizeRequests().antMatchers("/api/spaces/**").hasAuthority(("manager"));
        }

    }

---

Ces méthodes permettent de récupérer les infos de l'utilisateurs dans le security context de l'application.

    @Component
    public class KeycloakSecurityContext {

        /**
        *
        * @return roles
        */
        public OidcKeycloakAccount getOidcKeycloakAccount(){
            return getKeycloakAuthenticationToken().getAccount();
        }

        /**
        *
        * @return preferredUsername, email, givenName, familyName
        */
        public AccessToken getAccessToken(){
            return getOidcKeycloakAccount().getKeycloakSecurityContext().getToken();
        }

        @Bean
        @Scope(scopeName = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
        private KeycloakAuthenticationToken getKeycloakAuthenticationToken(){
            return (KeycloakAuthenticationToken) ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest().getUserPrincipal();
        }

    }

---

La méthode getAccessToken() permet ainsi de récupérer le username (accessToken.getPreferredUsername()), pour s'en servir par la suite.

    @Service
    public class UserServiceImpl implements UserService {

        @Autowired
        private KeycloakSecurityContext keycloakSecurityContext;

        @Autowired
        private UserRepository userRepository;

        @Autowired
        private SpaceRepository spaceRepository;

        @Autowired
        @Qualifier("UserUserDtoConverter")
        private SuperConverter<User, UserDto> converter;

        @Override
        public UserDto getConnectedUserDto() {
            try {
                return converter.toDto(getConnectedUser());
            } catch (CustomServiceException ex) {
                return null;
            }
        }

        @Override
        public List<UserDto> getAllUsers(final Integer pageNo, final Integer pageSize, final String sortBy) {
            final Pageable paging = PageRequest.of(pageNo, pageSize, Sort.by(sortBy));
            final Page<User> pagedResult = userRepository.findAll(paging);
            if(pagedResult.hasContent()) {
                return converter.toDtos(pagedResult.getContent());
            } else {
                return new ArrayList<>();
            }
        }



        //catch CustomTransactionalException in this method
        private User getConnectedUser() {
            final AccessToken accessToken = keycloakSecurityContext.getAccessToken();
            if (accessToken == null) {
                throw new CustomServiceException("error getAccessToken failed");
            } else if (accessToken.getPreferredUsername() == null || accessToken.getPreferredUsername().isEmpty()) {
                throw new CustomServiceException("error preferredUsername cannot be null or empty");
            }
            final String username = accessToken.getPreferredUsername();
            User user = userRepository.findByUsernameIgnoreCase(username);
            if (user == null) {
                try {
                    user = testCreateUser(accessToken);
                } catch (CustomTransactionalException ex) {
                    return null;
                }
            }
            return user;
        }

        // Do not catch CustomTransactionalException in this method
        @Transactional(propagation = Propagation.REQUIRES_NEW,
                rollbackFor = CustomTransactionalException.class)
        private User testCreateUser(final AccessToken accessToken) {
            return validateCreateUser(accessToken);
        }

        // MANDATORY: Transaction must be created before.
        @Transactional(propagation = Propagation.MANDATORY)
        private User validateCreateUser(final AccessToken accessToken) {
            try {
                final Space space = new Space(null, accessToken.getPreferredUsername() + "_space", null);
                spaceRepository.save(space);
                final User user = new User();
                user.setUsername(accessToken.getPreferredUsername());
                user.setEmail(accessToken.getEmail());
                user.setFirstName(accessToken.getGivenName());
                user.setLastName(accessToken.getFamilyName());
                user.setSpace(space);
                userRepository.save(user);
                return userRepository.findByUsernameIgnoreCase(accessToken.getPreferredUsername());
            } catch (Exception ex) {
                throw new CustomTransactionalException("persistence failed : " + ex.getMessage());
            }
        }

    }

---

Ces quelques classes/fichier de configuation permettent une très rapide implémentation d'un projet back-end avec une sécurisation optimale.


### Déploiement du Back-end et serveur Keycloak 100% sur son local ###

Build les images de Postgresql, Keycloak et sa bdd MySql (répertoire dev-docker).

    cd dev-docker
    docker-compose -f docker-compose.yml up -d

docker-compose.yml.

    version: '3.1'
    services:
        db:
            image: postgres
            container_name: postgresql-dev-docker
            restart: always
            environment:
                POSTGRES_DB: keycloak_back_end_db
                POSTGRES_PASSWORD: 1234!*MyJea@99
                POSTGRES_USER: user1
            volumes:
                - ./../target/postgresql-dev-docker:/var/lib/postgresql/data
            ports:
                - 5432:5432
        lab-mysql:
            container_name: mysql-dev-docker
            restart: always
            image: mysql/mysql-server
            environment:
                MYSQL_ROOT_PASSWORD: password
                MYSQL_ROOT_HOST: "%"
                MYSQL_DATABASE: keycloak
                MYSQL_USER: keycloak
                MYSQL_PASSWORD: password
            volumes:
                - ./../target/mysql-dev-docker:/var/lib/mysql
            ports:
                - 3306:3306
        keycloak:
            container_name: keycloak-dev-docker
            restart: always
            image: jboss/keycloak
            depends_on:
                - lab-mysql
            environment:
                DB_VENDOR: MYSQL
                DB_ADDR: lab-mysql
                DB_PORT: 3306
                DB_DATABASE: keycloak
                DB_USER: keycloak
                DB_PASSWORD: password
                KEYCLOAK_USER: admin
                KEYCLOAK_PASSWORD: 1234
            ports:
                - 8099:8080
                - 8443:8443
                - 9990:9990
            volumes:
                - ./../target/keycloak-dev-docker:/data
            links:
                - lab-mysql

---

Vérifier que les images ont bien été déployées.

    docker ps -a

---

    CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS                             PORTS                                                                    NAMES
    de2face102da        jboss/keycloak       "/opt/jboss/tools/do…"   22 seconds ago      Up 16 seconds                      0.0.0.0:8443->8443/tcp, 0.0.0.0:9990->9990/tcp, 0.0.0.0:8099->8080/tcp   keycloak-dev-docker
    2909874a449a        postgres             "docker-entrypoint.s…"   33 seconds ago      Up 24 seconds                      0.0.0.0:5432->5432/tcp                                                   postgresql-dev-docker
    1cef41fa3f93        mysql/mysql-server   "/entrypoint.sh mysq…"   33 seconds ago      Up 23 seconds (health: starting)   0.0.0.0:3306->3306/tcp, 33060/tcp                                        mysql-dev-docker

---

Si nécessaire, récupérer l'ip de l'image mysql-dev-docker.

    docker inspect mysql-dev-docker

---

    "NetworkID": "909b56f4c2caa1c594f8d691e0204cc236849fb25ddb2302bb75fcba516a8698",
    "EndpointID": "21d1c92defc16198b971ecf87be244bcff13c3e8bb5750753c8d2c428a85685e",
    "Gateway": "172.25.0.1",
    "IPAddress": "172.25.0.3",

---

Restaurer la base de données Mysql.

    cd keycloak-config
    cat backup.sql | docker exec -i mysql-dev-docker /usr/bin/mysql -h '172.25.0.3' -u keycloak -ppassword keycloak

ou

    cd keycloak-config
    cat backup.sql | docker exec -i mysql-dev-docker /usr/bin/mysql -u keycloak -ppassword keycloak

---

Un warning indique qu'il n'est pas recommandé de renseigner le password dans une ligne de commande...

    mysql: [Warning] Using a password on the command line interface can be insecure.

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

Rebuild et déployer les images.

    cd dev-docker
    docker-compose -f docker-compose.yml up -d

---

En relancant la commande docker ps -a nous constatons que tout est rentré dans l'ordre.

    CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS                             PORTS                                                                    NAMES
    97b1d64bf536        jboss/keycloak       "/opt/jboss/tools/do…"   25 seconds ago      Up 18 seconds                      0.0.0.0:8443->8443/tcp, 0.0.0.0:9990->9990/tcp, 0.0.0.0:8099->8080/tcp   keycloak-dev-docker
    3b9e6f4a82e3        postgres             "docker-entrypoint.s…"   38 seconds ago      Up 23 seconds                      0.0.0.0:5432->5432/tcp                                                   postgresql-dev-docker
    2bd8051a1850        mysql/mysql-server   "/entrypoint.sh mysq…"   38 seconds ago      Up 24 seconds (health: starting)   0.0.0.0:3306->3306/tcp, 33060/tcp                                        mysql-dev-docker


---


### Les profils maven et Springboot ###

Un profil maven a été définis par défaut. Il permet de lancer l'application sans exécuter de test.

		<!--profile by default								-->
		<!--no junit or integration tests in this profile	-->
		<profile>
			<id>dev</id>
			<activation>
				<activeByDefault>true</activeByDefault>
			</activation>
			<dependencies>
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
		</profile>

---

Le profil dev-docker permet de lacer exclusivement les tests unitaires.

		<!--profile for gitlab-ci										-->
		<!--all junit tests in this profile								-->
		<!--command														-->
		<!--mvn test -Pdev-docker -Dspring.profiles.active=dev-docker	-->
		<profile>
			<id>dev-docker</id>
			<activation>
				<activeByDefault>false</activeByDefault>
			</activation>
			<dependencies>
				<dependency>
					<groupId>org.springframework.boot</groupId>
					<artifactId>spring-boot-starter-test</artifactId>
					<scope>test</scope>
				</dependency>
			</dependencies>
			<build>
				<plugins>
					<!-- exclude integration tests-->
					<plugin>
						<groupId>org.apache.maven.plugins</groupId>
						<artifactId>maven-surefire-plugin</artifactId>
						<version>3.0.0-M1</version>
						<configuration>
							<excludes>
								<exclude>**/KeycloakApplicationTests*.java</exclude>
								<exclude>**/UserControllerContextTest*.java</exclude>
								<exclude>**/SpaceControllerContextTest*.java</exclude>
							</excludes>
						</configuration>
					</plugin>
				</plugins>
			</build>
		</profile>

---

Le profil dev-docker-integration-testing permet de ne lancer que les tests d'intégration

		<!--profile for gitlab-ci										-->
		<!--only integration tests in this profile						-->
		<!--command														-->
		<!--mvn test -Pdev-docker-integration-testing -Dspring.profiles.active=dev-docker	-->
		<profile>
			<id>dev-docker-integration-testing</id>
			<activation>
				<activeByDefault>false</activeByDefault>
			</activation>
			<dependencies>
				<dependency>
					<groupId>org.springframework.boot</groupId>
					<artifactId>spring-boot-starter-test</artifactId>
					<scope>test</scope>
				</dependency>
			</dependencies>
			<build>
				<plugins>
					<!-- only integration tests-->
					<plugin>
						<groupId>org.apache.maven.plugins</groupId>
						<artifactId>maven-surefire-plugin</artifactId>
						<version>3.0.0-M1</version>
						<configuration>
							<excludes>
								<exclude>**/SpaceSpaceDtoConverterTest*.java</exclude>
								<exclude>**/UserUserDtoConverterTest*.java</exclude>
								<exclude>**/UserServiceImplTest*.java</exclude>
								<exclude>**/UserServiceImplTest*.java</exclude>
							</excludes>
						</configuration>
					</plugin>
				</plugins>
			</build>
		</profile>

---

La combinaison de ces 3 profiles permettra l'exécution de plusiseurs étapes d'intégration et de déployement continue (voir le chapitre sur gitlab ci/cd).

---

Pour un déploiement en local ou pour compléter les tâches de ci/cd, la combinaison avec les profils Springboot va permettre de charger des variables sur un projet déjà compilés.

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

    keycloak:
        realm: Sc-project
        auth-server-url: http://localhost:8099/auth/
        resource: sc-user
        public-client: false
        bearer-only: true
        cors: true

    sc-properties:
        authServerUrl: http://localhost:8099/auth/
        authServerParameters: realms/Sc-project/protocol/openid-connect/token

Le profil dev-docker permet un déploiement en local et les tests unitaires.

---

    spring:
        profiles: dev-docker-integration

        jpa:
            database: POSTGRESQL
            show-sql: true
            hibernate:
                ddl-auto: create-drop

        datasource:
            platform: postgres
            url: jdbc:postgresql://jeannory.dynamic-dns.net:5432/keycloak_back_end_db
            username: user1
            password: 1234!*MyJea@99
            driverClassName: org.postgresql.Driver
            initialization-mode: always

    keycloak:
        realm: Sc-project
        auth-server-url: http://jeannory.ovh:8099/auth/
        resource: sc-user
        public-client: false
        bearer-only: true
        cors: true


    sc-properties:
        authServerUrl: http://jeannory.ovh:8099/auth/
        authServerParameters: realms/Sc-project/protocol/openid-connect/token

Le profil dev-docker-integration va être utilisé pour les tests d'intégration.

---

    spring:
        profiles: dev-docker-integration-deploy

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

    keycloak:
        realm: Sc-project
        auth-server-url: http://jeannory.ovh:8099/auth/
        resource: sc-user
        public-client: false
        bearer-only: true
        cors: true
        ssl-required: none

    sc-properties:
        authServerUrl: http://jeannory.ovh:8099/auth/
        authServerParameters: realms/Sc-project/protocol/openid-connect/token

Enfin le profil dev-docker-integration-deploy va être utilisé pour le déploiement du projet.

---

### Les tests unitaires ###

Ce projet comporte des tests unitaires. La méthode *getConnectedUserDto* a été contruite par les tests (principe du test first).

Méthode :

    @Service
    public class UserServiceImpl implements UserService {

        @Autowired
        private KeycloakSecurityContext keycloakSecurityContext;

        @Autowired
        private UserRepository userRepository;

        @Autowired
        private SpaceRepository spaceRepository;

        @Autowired
        @Qualifier("UserUserDtoConverter")
        private SuperConverter<User, UserDto> converter;

        @Override
        public UserDto getConnectedUserDto() {
            try {
                return converter.toDto(getConnectedUser());
            } catch (CustomServiceException ex) {
                return null;
            }
        }

        //catch CustomTransactionalException in this method
        private User getConnectedUser() {
            final AccessToken accessToken = keycloakSecurityContext.getAccessToken();
            if (accessToken == null) {
                throw new CustomServiceException("error getAccessToken failed");
            } else if (accessToken.getPreferredUsername() == null || accessToken.getPreferredUsername().isEmpty()) {
                throw new CustomServiceException("error preferredUsername cannot be null or empty");
            }
            final String username = accessToken.getPreferredUsername();
            User user = userRepository.findByUsernameIgnoreCase(username);
            if (user == null) {
                try {
                    user = testCreateUser(accessToken);
                } catch (CustomTransactionalException ex) {
                    return null;
                }
            }
            return user;
        }

        // Do not catch CustomTransactionalException in this method
        @Transactional(propagation = Propagation.REQUIRES_NEW,
                rollbackFor = CustomTransactionalException.class)
        private User testCreateUser(final AccessToken accessToken) {
            return validateCreateUser(accessToken);
        }

        // MANDATORY: Transaction must be created before.
        @Transactional(propagation = Propagation.MANDATORY)
        private User validateCreateUser(final AccessToken accessToken) {
            try {
                final Space space = new Space(null, accessToken.getPreferredUsername() + "_space", null);
                spaceRepository.save(space);
                final User user = new User();
                user.setUsername(accessToken.getPreferredUsername());
                user.setEmail(accessToken.getEmail());
                user.setFirstName(accessToken.getGivenName());
                user.setLastName(accessToken.getFamilyName());
                user.setSpace(space);
                userRepository.save(user);
                return userRepository.findByUsernameIgnoreCase(accessToken.getPreferredUsername());
            } catch (Exception ex) {
                throw new CustomTransactionalException("persistence failed : " + ex.getMessage());
            }
        }

    }

---


Class de test :

    public class UserServiceImplTest {

        @InjectMocks
        private UserServiceImpl userService;

        @Mock
        private KeycloakSecurityContext keycloakSecurityContext;

        @Mock
        private UserRepository userRepository;

        @Mock
        private SpaceRepository spaceRepository;

        @Mock
        private SuperConverter<User, UserDto> converter;

        @Before
        public void setUp() throws Exception {
            MockitoAnnotations.initMocks(this);
        }

        @Test
        public void testGetConnectedUserDtoWhenUserExistsReturnDto() {
            //given
            final AccessToken accessToken = Mockito.spy(new AccessToken());
            accessToken.setPreferredUsername("XLD");
            Mockito.when(keycloakSecurityContext.getAccessToken()).thenReturn(accessToken);
            final User user = new User();
            user.setUsername("XLD");
            Mockito.when(userRepository.findByUsernameIgnoreCase(Mockito.anyString())).thenReturn(user);
            final UserDto userDto = new UserDto();
            userDto.setUsername("XLD");
            Mockito.when(converter.toDto(user)).thenReturn(userDto);

            //when
            final UserDto result = userService.getConnectedUserDto();

            //then
            assertEquals("XLD", result.getUsername());
        }

        @Test
        public void testGetConnectedUserDtoWhenUserExistsReturnDtoBis() {
            //given
            Mockito.when(keycloakSecurityContext.getAccessToken()).thenReturn(BuilderUtils.getAccessToken());
            Mockito.when(userRepository.findByUsernameIgnoreCase(Mockito.anyString())).thenReturn(BuilderUtils.getUser());
            Mockito.when(converter.toDto(BuilderUtils.getUser())).thenReturn(BuilderUtils.getUserDto());

            //when
            final UserDto result = userService.getConnectedUserDto();

            //then
            assertEquals(1L, (long) result.getId());
            assertEquals("user1",result.getUsername());
            assertEquals("user1@gmail.com",result.getEmail());
            assertEquals(Gender.Mister,result.getGender());
            assertEquals("firstName1",result.getFirstName());
            assertEquals("name1",result.getLastName());
            assertEquals(Status.ACTIVE,result.getStatus());
            assertEquals("user1_space", result.getSpace().getName());
        }

        @Test
        public void testGetConnectedUserDtoWhenUserNoExistsReturnDto() {
            //given
            final AccessToken accessToken = Mockito.spy(new AccessToken());
            accessToken.setPreferredUsername("XLD");
            Mockito.when(keycloakSecurityContext.getAccessToken()).thenReturn(accessToken);
            Mockito.when(userRepository.findByUsernameIgnoreCase(Mockito.anyString())).thenReturn(null);
            final Space space = new Space();
            space.setId(1L);
            space.setName("XLD_space");

            Mockito.when(spaceRepository.save(Mockito.any(Space.class))).thenReturn(space);
            final User user = new User();
            user.setId(1L);
            user.setUsername("XLD");
            user.setSpace(space);
            Mockito.when(userRepository.save(Mockito.any(User.class))).thenReturn(user);
            Mockito.when(userRepository.findByUsernameIgnoreCase(Mockito.anyString())).thenReturn(user);

            final SpaceDto spaceDto = new SpaceDto();
            spaceDto.setName("XLD_space");

            final UserDto userDto = new UserDto();
            userDto.setUsername("XLD");
            userDto.setSpace(spaceDto);
            Mockito.when(converter.toDto(user)).thenReturn(userDto);

            //when
            final UserDto result = userService.getConnectedUserDto();
            result.toString();

            //then
            assertEquals("XLD", result.getUsername());
        }

        @Test
        public void testGetConnectedUserWhenGetAccessTokenIsNullThenThrowExceptionReturnNull(){
            //given
            Mockito.when(keycloakSecurityContext.getAccessToken()).thenReturn(null);

            //when
            final UserDto result = userService.getConnectedUserDto();

            //then
            assertNull("return null", result);
        }

        @Test
        public void testGetConnectedUserWhenPreferredUsernameIsNullThenThrowExceptionReturnNull() {
            //given
            final AccessToken accessToken = Mockito.spy(new AccessToken());
            Mockito.when(keycloakSecurityContext.getAccessToken()).thenReturn(accessToken);

            //when
            final UserDto result = userService.getConnectedUserDto();

            //then
            assertNull("return null", result);
        }

        @Test
        public void testGetConnectedUserWhenPreferredUsernameIsEmptyThenThrowExceptionReturnNull() {
            //given
            final AccessToken accessToken = Mockito.spy(new AccessToken());
            accessToken.setPreferredUsername("");
            Mockito.when(keycloakSecurityContext.getAccessToken()).thenReturn(accessToken);

            //when
            final UserDto result = userService.getConnectedUserDto();

            //then
            assertNull("return null", result);
        }

        @Test
        public void testGetConnectedUserWhenSpaceSaveFailedThenThrowExceptionReturnNull() {
            //given
            final AccessToken accessToken = Mockito.spy(new AccessToken());
            accessToken.setPreferredUsername("XLD");
            Mockito.when(keycloakSecurityContext.getAccessToken()).thenReturn(accessToken);
            Mockito.when(userRepository.findByUsernameIgnoreCase(Mockito.anyString())).thenReturn(null);
            final Space space = new Space();
            space.setName("XLD_space");
            Mockito.when(spaceRepository.save(Mockito.any(Space.class))).thenThrow(new CustomTransactionalException());

            //when
            final UserDto result = userService.getConnectedUserDto();

            //then
            assertNull("return null", result);
        }

        @Test
        public void testGetConnectedUserWhenUserSaveFailedThenThrowExceptionReturnNull() {
            //given
            final AccessToken accessToken = Mockito.spy(new AccessToken());
            accessToken.setPreferredUsername("XLD");
            Mockito.when(keycloakSecurityContext.getAccessToken()).thenReturn(accessToken);
            Mockito.when(userRepository.findByUsernameIgnoreCase(Mockito.anyString())).thenReturn(null);
            final User user = new User();
            user.setUsername("XLD");
            Mockito.when(userRepository.save(Mockito.any(User.class))).thenThrow(new CustomTransactionalException());
            final Space space = new Space();
            space.setName("XLD_space");
            space.setUser(user);
            Mockito.when(spaceRepository.save(Mockito.any(Space.class))).thenReturn(space);

            //when
            final UserDto result = userService.getConnectedUserDto();

            //then
            assertNull("return null", result);
        }
    }

---

### Les tests d'intégration ###

Ce projet dispose de tests d'intégration.
Il est nécessaire de diposer d'un environnement complet, un serveur d'authentification et la base de donnée du projet.
Ces tests pourront être lancés en local ou depuis gitlab. Il faut pour cela charger le profil maven et Springboot adéquat.

    /**
    * Initialize spring context before the class
    */
    @RunWith(SpringRunner.class)
    @SpringBootTest
    @AutoConfigureMockMvc
    /**
    * ActiveProfiles must be launch by maven
    * mvn test -Pdev-local -Dspring.profiles.active=dev-local
    */
    //@ActiveProfiles("dev-docker")
    @DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_CLASS)
    public class UserControllerContextTest extends BuilderUtilsResultAction {
        @Autowired
        private BuilderUtilsKeycloak builderUtilsKeycloak;

        @Test
        public void getGetUsersShouldReturnForbiddenWhenRoleIsUser() throws Exception {
            final String accessToken = builderUtilsKeycloak.getAccessToken("test", "1234");
            final MvcResult mvcResult = invokeGet("/api/users", accessToken)
                    .andExpect(status().isForbidden())
                    .andReturn();
        }

        @Test
        public void testGetUsersWhenRoleIsManagerShouldReturn9() throws Exception {
            final String accessToken = builderUtilsKeycloak.getAccessToken("phou.jeannory@gmail.com", "1234");
            final MultiValueMap<String, String> parameters = new LinkedMultiValueMap<>();
            parameters.put("pageNo", Collections.singletonList("0"));
            parameters.put("pageSize", Collections.singletonList("9"));
            parameters.put("sortBy", Collections.singletonList("id"));
            final MvcResult mvcResult = invokeGet("/api/users", accessToken, parameters)
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$",hasSize(9)))
                    .andExpect(jsonPath("$[0].username", is("user1")))
                    .andReturn();
        }

        @Test
        public void testGetConnectedUser() throws Exception {
            final String accessToken = builderUtilsKeycloak.getAccessToken("test", "1234");
            final MvcResult mvcResult = invokeGet("/api/users/connected-user", accessToken)
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("email", is("test@gmail.com")))
                    .andExpect(jsonPath("space.name", is("test_space")))
                    .andReturn();
        }

    }

Pour le mode local la commande est :

    mvn test -Pdev-local -Dspring.profiles.active=dev-docker

---

## Le front-end avec Angular Material ##

### Le projet et ses dépendances ###

Le front-end a été réalisé depuis un projet sous licence MIT : Material Dashboard Angular.
Récupérer le à l'addresse suivante : https://www.creative-tim.com/product/material-dashboard-angular2 (format zip).

![Angular Material](/blog/img/gitlab-20.png)

Une fois le projet dézippé dans un répertoire local, télécharger toutes les dépendances.

    npm i

---

Rendez-vous sur le site de keycloak-js et installer la librairie

![keycloak js](/blog/img/gitlab-21.png)

    npm i keycloak-js --save

---

Il faut rajouter dans le fichier angular.json *"node_modules/keycloak-js/dist/keycloak.min.js"*

    "scripts": [
        "node_modules/jquery/dist/jquery.js",
        "node_modules/popper.js/dist/umd/popper.js",
        "node_modules/bootstrap-material-design/dist/js/bootstrap-material-design.min.js",
        "node_modules/arrive/src/arrive.js",
        "node_modules/moment/moment.js",
        "node_modules/perfect-scrollbar/dist/perfect-scrollbar.min.js",
        "node_modules/bootstrap-notify/bootstrap-notify.js",
        "node_modules/chartist/dist/chartist.js",
        "node_modules/keycloak-js/dist/keycloak.min.js"
    ]

### Les variables d'environnement ###

Pour commencer il faut setter les variables à l'application

environment.ts

    export const environment = {
    SC_USER_BASE_URL: 'localhost',
    SC_USER_PORT: '8081',
    AUTH_BASE_URL: 'localhost',
    AUTH_PORT: '8099',
    // APP_VERSION: version,
    production: false,
    };

---

environment.prod.ts

    export const environment = {
    SC_USER_BASE_URL: 'jeannory.dynamic-dns.net',
    SC_USER_PORT: '8081',
    AUTH_BASE_URL: 'jeannory.ovh',
    AUTH_PORT: '8099',
    // APP_VERSION: version,
    production: true,
    };

---

Toutes les urls sont regroupées dans des variables statics

Générer la class

    ng g class utils/ApiUrl

---

api-url.ts

        import { environment } from '../../environments/environment';

    export class ApiUrl {
        private static readonly BASE_SC_USER_URL = `http://${environment.SC_USER_BASE_URL}:${environment.SC_USER_PORT}`;
        private static readonly BASE_API = '/api';
        private static readonly USERS = '/users';
        private static readonly CONNECTED_USER = '/connected-user'
        private static readonly AUTH = '/auth/'
        private static readonly AUTH_SERVER = `http://${environment.AUTH_BASE_URL}:${environment.AUTH_PORT}`;

        static get GET_CONNECTED_USER(): string {
            return `${this.BASE_SC_USER_URL}${this.BASE_API}${this.USERS}${this.CONNECTED_USER}`;
        }

        static get GET_USERS(): string {
            return `${this.BASE_SC_USER_URL}${this.BASE_API}${this.USERS}`;
        }

        static get GET_AUTH_SERVER(): string {
            return `${this.AUTH_SERVER}${this.AUTH}`;
        }
    }

### Implémentation de keycloak ###

Voici les étapes nécessaires

Créer le service KeycloakSecurity

    ng g service services/KeycloakSecurity

---

    import { Injectable } from '@angular/core';
    import { ApiUrl } from '../utils/api-url';

    declare var Keycloak: any;
    @Injectable({
    providedIn: 'root'
    })
    export class KeycloakSecurityService {

    public kc;
    apiUrl = ApiUrl;

    constructor() { }
    init() {
        return new Promise((resolve, reject) => {
        console.log('security Initialisation ...');
        this.kc = new Keycloak({
            url: this.apiUrl.GET_AUTH_SERVER,
            realm: 'Sc-project',
            clientId: 'sc-ui'
        });
        this.kc.init({
            onLoad: 'check-sso',
            promiseType: 'native'
        }).then((authenticated) => {
            console.log(this.kc.token);
            resolve({ auth: authenticated, token: this.kc.token });
        }).catch(err => {
            reject(err);
        });
        });
    }

    }

...

Puis créer le service RequestInterceptor qui implements HttpInterceptor.
La redéfinition des méthodes de HttpInterceptor permet d'ajouter un header à chaque requête et récupérer tous les status codes des responses serveurs (cette fonction ne sera pas implémenter ici)

    ng g service services/RequestInterceptor

---

    import { Injectable } from '@angular/core';
    import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent } from '@angular/common/http';
    import { KeycloakSecurityService } from './keycloak-security.service';
    import { Observable } from 'rxjs';

    @Injectable({
    providedIn: 'root'
    })
    export class RequestInterceptorService implements HttpInterceptor{

    constructor(private securityService: KeycloakSecurityService) { }

    intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
        console.log('Request Http Interceptor ...');
        if (!this.securityService.kc.authenticated) {
        return next.handle(req);
        } else {
        const request = req.clone({
            setHeaders: {
            Authorization: 'Bearer ' + this.securityService.kc.token
            }
        });
        return next.handle(request);
        }
    }
    }

Modifier app.module.ts comme tel

    import { BrowserAnimationsModule } from '@angular/platform-browser/animations';
    import { NgModule, DoBootstrap, ApplicationRef } from '@angular/core';
    import { FormsModule, ReactiveFormsModule } from '@angular/forms';
    import { HttpModule } from '@angular/http';
    import { RouterModule } from '@angular/router';
    import { AppRoutingModule } from './app.routing';
    import { ComponentsModule } from './components/components.module';
    import { AppComponent } from './app.component';
    import {
    AgmCoreModule
    } from '@agm/core';
    import { AdminLayoutComponent } from './layouts/admin-layout/admin-layout.component';
    import { KeycloakSecurityService } from './services/keycloak-security.service';
    import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http';
    import { RequestInterceptorService } from './services/request-interceptor.service';
    import { MatSnackBarModule } from '@angular/material/snack-bar';

    const securityService = new KeycloakSecurityService();

    @NgModule({
    imports: [
        BrowserAnimationsModule,
        FormsModule,
        ReactiveFormsModule,
        HttpModule,
        ComponentsModule,
        RouterModule,
        AppRoutingModule,
        AgmCoreModule.forRoot({
        apiKey: 'YOUR_GOOGLE_MAPS_API_KEY'
        }),
        HttpClientModule,
        MatSnackBarModule
    ],
    declarations: [
        AppComponent,
        AdminLayoutComponent,
    ],
    providers: [
        { provide: KeycloakSecurityService, useValue: securityService },
        { provide: HTTP_INTERCEPTORS, useClass: RequestInterceptorService, multi: true }
    ],
    })

    export class AppModule implements DoBootstrap {

    ngDoBootstrap(appRef: ApplicationRef): void {
        securityService.init()
        .then(res => {
            console.log(res);
            appRef.bootstrap(AppComponent);
        })
        .catch((err) => {
            console.log('Keycloak error ', err);
        });
    }
    }

---

Il faut également modifier app.component.ts

    import { Component, OnInit} from '@angular/core';
    import { KeycloakSecurityService } from './services/keycloak-security.service';


    @Component({
    selector: 'app-root',
    templateUrl: './app.component.html',
    styleUrls: ['./app.component.css']
    })
    export class AppComponent implements OnInit{

    title = 'sc-ui';

    constructor(public securityService: KeycloakSecurityService){

    }

    ngOnInit(): void {
        console.log('AppComponent');
    }

    }

---

Il faut créer le service RoleGuard qui servira à protéger l'accès aux pages

    ng g service services/RoleGuard

---

    import { Injectable } from '@angular/core';
    import { CanActivate, Router, ActivatedRouteSnapshot } from '@angular/router';
    import { KeycloakSecurityService } from './keycloak-security.service';
    import { MatSnackBar } from '@angular/material/snack-bar';

    @Injectable({
    providedIn: 'root'
    })
    export class RoleGuardService implements CanActivate {

    roles = [];

    constructor(
        public securityService: KeycloakSecurityService,
        private router: Router,
        private snackBar: MatSnackBar
    ) {
        this.getAccessRole();
    }

    canActivate(route: ActivatedRouteSnapshot): boolean {
        const expectedRoleArray = route.data;
        let i = 0;
        while (i < expectedRoleArray.expectedRole.length) {
        let j = 0;
        while (j < this.roles.length) {
            if (expectedRoleArray.expectedRole[i] === this.roles[j]) {
            return true;
            }
            j++;
        }
        i++;
        }
        this.snackBar.open('You are not authorized to access this page.', 'ok', {
        panelClass: 'snackbar-error',
        verticalPosition: 'top',
        duration: 5000,
        });
        this.router.navigate(['/home']);
        return false;
    }

    private getAccessRole(): void {
        if (this.securityService.kc.hasRealmRole('user')) {
        this.roles.push('user');
        }
        if (this.securityService.kc.hasRealmRole('manager')) {
        this.roles.push('manager');
        }
    }
    }

---

Enfin nous allons créer un service qui va centraliser les requêtes vers le serveur back-end

    ng g service services/Api

---

Les méthodes sont très basiques

    import { Injectable } from '@angular/core';
    import { HttpClient } from '@angular/common/http';
    import { ApiUrl } from '../utils/api-url';

    @Injectable({
    providedIn: 'root'
    })
    export class ApiService {

    apiUrl = ApiUrl;

    constructor(
        private http: HttpClient,
    ) { }

    public getConnectedUser() {
        return this.http.get(this.apiUrl.GET_CONNECTED_USER)
    }

    public getUsers(pageNo: number, pageSize: number, sortBy: string) {
        return this.http.get(this.apiUrl.GET_USERS + '?pageNo=' + pageNo + '&pageSize=' + pageSize + '&sortBy=' + sortBy);
    }
    }

---

La page Home permettra de rediriger les utilisateurs authentifiés

    ng g c Home

---

Il faut encore modifier quelques fichiers du projet existant

Pour naviguer vers la page *Home*

sidebar.component.ts

    import { Component, OnInit } from '@angular/core';
    import { KeycloakSecurityService } from 'app/services/keycloak-security.service';

    declare const $: any;
    declare interface RouteInfo {
        path: string;
        title: string;
        icon: string;
        class: string;
    }
    export const ROUTES: RouteInfo[] = [
        { path: '/home', title: 'Home',  icon: 'home', class: '' },
        { path: '/dashboard', title: 'Dashboard',  icon: 'dashboard', class: '' },
        { path: '/user-profile', title: 'User Profile',  icon:'person', class: '' },
        { path: '/table-list', title: 'Table List',  icon:'content_paste', class: '' },
        { path: '/typography', title: 'Typography',  icon:'library_books', class: '' },
        { path: '/icons', title: 'Icons',  icon:'bubble_chart', class: '' },
        { path: '/maps', title: 'Maps',  icon:'location_on', class: '' },
        { path: '/notifications', title: 'Notifications',  icon:'notifications', class: '' },
        { path: '/upgrade', title: 'Upgrade to PRO',  icon:'unarchive', class: 'active-pro' },
    ];

    @Component({
    selector: 'app-sidebar',
    templateUrl: './sidebar.component.html',
    styleUrls: ['./sidebar.component.css']
    })
    export class SidebarComponent implements OnInit {
    menuItems: any[];

    constructor(public securityService: KeycloakSecurityService) { }

    ngOnInit() {
        this.menuItems = ROUTES.filter(menuItem => menuItem);
    }
    isMobileMenu() {
        if ($(window).width() > 991) {
            return false;
        }
        return true;
    };

    }

---

sidebar.component.ts

    <div class="logo">
        <a routerLink="/home" class="simple-text">
            <div class="logo-img">
                <img src="/assets/img/angular2-logo-red.png" />
            </div>
            Creative Tim
        </a>
    </div>
    <div class="sidebar-wrapper">
        <div *ngIf="isMobileMenu()">
            <form class="navbar-form">
                <span class="bmd-form-group">
                    <div class="input-group no-border">
                        <input type="text" value="" class="form-control" placeholder="Search...">
                        <button mat-raised-button type="submit" class="btn btn-white btn-round btn-just-icon">
                            <i class="material-icons">search</i>
                            <div class="ripple-container"></div>
                        </button>
                    </div>
                </span>
            </form>
            <ul class="nav navbar-nav nav-mobile-menu">
                <li class="nav-item">
                    <a class="nav-link" href="javascript:void(0)">
                        <i class="material-icons">dashboard</i>
                        <p>
                            <span class="d-lg-none d-md-block">Stats</span>
                        </p>
                    </a>
                </li>
                <li class="nav-item dropdown">
                    <a class="nav-link" href="javascript:void(0)" id="navbarDropdownMenuLink" data-toggle="dropdown"
                        aria-haspopup="true" aria-expanded="false">
                        <i class="material-icons">notifications</i>
                        <span class="notification">5</span>
                        <p>
                            <span class="d-lg-none d-md-block">Some Actions</span>
                        </p>
                    </a>
                    <div class="dropdown-menu dropdown-menu-right" aria-labelledby="navbarDropdownMenuLink">
                        <a class="dropdown-item" href="#">Mike John responded to your email</a>
                        <a class="dropdown-item" href="#">You have 5 new tasks</a>
                        <a class="dropdown-item" href="#">You're now friend with Andrew</a>
                        <a class="dropdown-item" href="#">Another Notification</a>
                        <a class="dropdown-item" href="#">Another One</a>
                    </div>
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="javascript:void(0)">
                        <i class="material-icons">person</i>
                        <p>
                            <span class="d-lg-none d-md-block">Account</span>
                        </p>
                    </a>
                </li>
            </ul>
        </div>
        <ul class="nav">
            <li routerLinkActive="active" *ngFor="let menuItem of menuItems" class="{{menuItem.class}} nav-item">
                <a class="nav-link" [routerLink]="[menuItem.path]"
                    *ngIf="!securityService.kc.authenticated && menuItem.title == 'Home'">
                    <i class="material-icons">{{menuItem.icon}}</i>
                    <p>{{menuItem.title}}</p>
                </a>
                <a class="nav-link" [routerLink]="[menuItem.path]" *ngIf="securityService.kc.authenticated">
                    <i class="material-icons">{{menuItem.icon}}</i>
                    <p>{{menuItem.title}}</p>
                </a>
            </li>
        </ul>
    </div>

---

Quelques méthodes de la librairies de keycloak seront acessible depuis la navbar

navbar.component.ts

    import { Component, OnInit, ElementRef } from '@angular/core';
    import { ROUTES } from '../sidebar/sidebar.component';
    import { Location, LocationStrategy, PathLocationStrategy } from '@angular/common';
    import { Router } from '@angular/router';
    import { KeycloakSecurityService } from 'app/services/keycloak-security.service';

    @Component({
        selector: 'app-navbar',
        templateUrl: './navbar.component.html',
        styleUrls: ['./navbar.component.css']
    })
    export class NavbarComponent implements OnInit {
        private listTitles: any[];
        location: Location;
        mobile_menu_visible: any = 0;
        private toggleButton: any;
        private sidebarVisible: boolean;

        constructor(
            location: Location,
            private element: ElementRef,
            private router: Router,
            public securityService: KeycloakSecurityService) {
            this.location = location;
            this.sidebarVisible = false;
        }

        ngOnInit() {
            this.listTitles = ROUTES.filter(listTitle => listTitle);
            const navbar: HTMLElement = this.element.nativeElement;
            this.toggleButton = navbar.getElementsByClassName('navbar-toggler')[0];
            this.router.events.subscribe((event) => {
                this.sidebarClose();
                var $layer: any = document.getElementsByClassName('close-layer')[0];
                if ($layer) {
                    $layer.remove();
                    this.mobile_menu_visible = 0;
                }
            });
        }

        sidebarOpen() {
            const toggleButton = this.toggleButton;
            const body = document.getElementsByTagName('body')[0];
            setTimeout(function () {
                toggleButton.classList.add('toggled');
            }, 500);

            body.classList.add('nav-open');

            this.sidebarVisible = true;
        };
        sidebarClose() {
            const body = document.getElementsByTagName('body')[0];
            this.toggleButton.classList.remove('toggled');
            this.sidebarVisible = false;
            body.classList.remove('nav-open');
        };
        sidebarToggle() {
            // const toggleButton = this.toggleButton;
            // const body = document.getElementsByTagName('body')[0];
            var $toggle = document.getElementsByClassName('navbar-toggler')[0];

            if (this.sidebarVisible === false) {
                this.sidebarOpen();
            } else {
                this.sidebarClose();
            }
            const body = document.getElementsByTagName('body')[0];

            if (this.mobile_menu_visible == 1) {
                // $('html').removeClass('nav-open');
                body.classList.remove('nav-open');
                if ($layer) {
                    $layer.remove();
                }
                setTimeout(function () {
                    $toggle.classList.remove('toggled');
                }, 400);

                this.mobile_menu_visible = 0;
            } else {
                setTimeout(function () {
                    $toggle.classList.add('toggled');
                }, 430);

                var $layer = document.createElement('div');
                $layer.setAttribute('class', 'close-layer');


                if (body.querySelectorAll('.main-panel')) {
                    document.getElementsByClassName('main-panel')[0].appendChild($layer);
                } else if (body.classList.contains('off-canvas-sidebar')) {
                    document.getElementsByClassName('wrapper-full-page')[0].appendChild($layer);
                }

                setTimeout(function () {
                    $layer.classList.add('visible');
                }, 100);

                $layer.onclick = function () { //asign a function
                    body.classList.remove('nav-open');
                    this.mobile_menu_visible = 0;
                    $layer.classList.remove('visible');
                    setTimeout(function () {
                        $layer.remove();
                        $toggle.classList.remove('toggled');
                    }, 400);
                }.bind(this);

                body.classList.add('nav-open');
                this.mobile_menu_visible = 1;

            }
        };

        getTitle() {
            let titlee = this.location.prepareExternalUrl(this.location.path());
            if (titlee.charAt(0) === '#') {
                titlee = titlee.slice(1);
            }

            for (let item = 0; item < this.listTitles.length; item++) {
                if (this.listTitles[item].path === titlee) {
                    return this.listTitles[item].title;
                }
            }
            return 'Dashboard';
        }

        onLogin() {
            this.securityService.kc.login();
        }

        onLogout() {
            this.securityService.kc.logout();
        }

        onEditAccount() {
            this.securityService.kc.accountManagement();
        }
    }

---

navbar.component.html

    <nav class="navbar navbar-expand-lg navbar-transparent  navbar-absolute fixed-top">
        <div class="container-fluid">
            <div class="navbar-wrapper">
                <a class="navbar-brand" href="javascript:void(0)">{{getTitle()}}</a>
            </div>
            <button mat-raised-button class="navbar-toggler" type="button" (click)="sidebarToggle()">
                <span class="sr-only">Toggle navigation</span>
                <span class="navbar-toggler-icon icon-bar"></span>
                <span class="navbar-toggler-icon icon-bar"></span>
                <span class="navbar-toggler-icon icon-bar"></span>
            </button>
            <div class="collapse navbar-collapse justify-content-end" id="navigation"
                *ngIf="securityService.kc.authenticated">
                <form class="navbar-form">
                    <div class="input-group no-border">
                        <input type="text" value="" class="form-control" placeholder="Search...">
                        <button mat-raised-button type="submit" class="btn btn-white btn-round btn-just-icon">
                            <i class="material-icons">search</i>
                            <div class="ripple-container"></div>
                        </button>
                    </div>
                </form>
                <ul class="navbar-nav" *ngIf="securityService.kc.authenticated">
                    <li class="nav-item dropdown">
                        <a class="nav-link" href="javascript:void(0)" id="navbarDropdownMenuLink" data-toggle="dropdown"
                            aria-haspopup="true" aria-expanded="false">
                            <i class="material-icons">notifications</i>
                            <span class="notification">5</span>
                            <p>
                                <span class="d-lg-none d-md-block">Some Actions</span>
                            </p>
                        </a>
                        <div class="dropdown-menu dropdown-menu-right" aria-labelledby="navbarDropdownMenuLink">
                            <a class="dropdown-item" href="javascript:void(0)">Mike John responded to your email</a>
                            <a class="dropdown-item" href="javascript:void(0)">You have 5 new tasks</a>
                            <a class="dropdown-item" href="javascript:void(0)">You're now friend with Andrew</a>
                            <a class="dropdown-item" href="javascript:void(0)">Another Notification</a>
                            <a class="dropdown-item" href="javascript:void(0)">Another One</a>
                        </div>
                    </li>
                    <li class="nav-item">
                        <a class="clickable nav-link" (click)="onEditAccount()">
                            <i class="material-icons">person</i>
                            <p>
                                <span class="d-lg-none d-md-block">account</span>
                            </p>
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="clickable nav-link" (click)="onLogout()">
                            <i class="material-icons">exit_to_app</i>
                            <p>
                                <span class="d-lg-none d-md-block">logout</span>
                            </p>
                        </a>
                    </li>
                </ul>
            </div>
            <div class="collapse navbar-collapse justify-content-end" id="navigation"
                *ngIf="!securityService.kc.authenticated">
                <ul class="navbar-nav">
                    <li class="nav-item">
                        <a class="clickable nav-link" (click)="onLogin()">
                            <i class="material-icons">exit_to_app</i>
                            <p>
                                <span class="d-lg-none d-md-block">login</span>
                            </p>
                        </a>
                    </li>
                </ul>
            </div>
        </div>
    </nav>

---

navbar.component.css

    .clickable{
        cursor: pointer
    }

---

Il est important de protéger certaines pages en fonction des rôles de l'utilisateur

admin-layout.routing.ts

    import { Routes } from '@angular/router';

    import { DashboardComponent } from '../../dashboard/dashboard.component';
    import { UserProfileComponent } from '../../user-profile/user-profile.component';
    import { TableListComponent } from '../../table-list/table-list.component';
    import { TypographyComponent } from '../../typography/typography.component';
    import { IconsComponent } from '../../icons/icons.component';
    import { MapsComponent } from '../../maps/maps.component';
    import { NotificationsComponent } from '../../notifications/notifications.component';
    import { UpgradeComponent } from '../../upgrade/upgrade.component';
    import { RoleGuardService } from 'app/services/role-guard.service';
    import { HomeComponent } from 'app/home/home.component';

    export const AdminLayoutRoutes: Routes = [
        {
            path: 'home',
            component: HomeComponent
        },
        {
            path: 'dashboard',
            component: DashboardComponent,
            canActivate: [RoleGuardService],
            data: {
                expectedRole: ['user', 'manager']
            }
        },
        {
            path: 'user-profile',
            component: UserProfileComponent,
            canActivate: [RoleGuardService],
            data: {
                expectedRole: ['user', 'manager']
            }
        },
        {
            path: 'table-list',
            component: TableListComponent,
            canActivate: [RoleGuardService],
            data: {
                expectedRole: ['manager']
            }
        },
        {
            path: 'typography',
            component: TypographyComponent,
            canActivate: [RoleGuardService],
            data: {
                expectedRole: ['user', 'manager']
            }
        },
        {
            path: 'icons',
            component: IconsComponent,
            canActivate: [RoleGuardService],
            data: {
                expectedRole: ['user', 'manager']
            }
        },
        {
            path: 'maps',
            component: MapsComponent,
            canActivate: [RoleGuardService],
            data: {
                expectedRole: ['user', 'manager']
            }
        },
        {
            path: 'notifications',
            component: NotificationsComponent,
            canActivate: [RoleGuardService],
            data: {
                expectedRole: ['user', 'manager']
            }
        },
        {
            path: 'upgrade',
            component: UpgradeComponent,
            canActivate: [RoleGuardService],
            data: {
                expectedRole: ['user', 'manager']
            }
        },
    ];

---

admin-layout.module.ts

    import { NgModule } from '@angular/core';
    import { RouterModule } from '@angular/router';
    import { CommonModule } from '@angular/common';
    import { FormsModule, ReactiveFormsModule } from '@angular/forms';
    import { AdminLayoutRoutes } from './admin-layout.routing';
    import { DashboardComponent } from '../../dashboard/dashboard.component';
    import { UserProfileComponent } from '../../user-profile/user-profile.component';
    import { TableListComponent } from '../../table-list/table-list.component';
    import { TypographyComponent } from '../../typography/typography.component';
    import { IconsComponent } from '../../icons/icons.component';
    import { MapsComponent } from '../../maps/maps.component';
    import { NotificationsComponent } from '../../notifications/notifications.component';
    import { UpgradeComponent } from '../../upgrade/upgrade.component';
    import {MatButtonModule} from '@angular/material/button';
    import {MatInputModule} from '@angular/material/input';
    import {MatRippleModule} from '@angular/material/core';
    import {MatFormFieldModule} from '@angular/material/form-field';
    import {MatTooltipModule} from '@angular/material/tooltip';
    import {MatSelectModule} from '@angular/material/select';
    import { HomeComponent } from 'app/home/home.component';

    @NgModule({
    imports: [
        CommonModule,
        RouterModule.forChild(AdminLayoutRoutes),
        FormsModule,
        ReactiveFormsModule,
        MatButtonModule,
        MatRippleModule,
        MatFormFieldModule,
        MatInputModule,
        MatSelectModule,
        MatTooltipModule,
    ],
    declarations: [
        HomeComponent,
        DashboardComponent,
        UserProfileComponent,
        TableListComponent,
        TypographyComponent,
        IconsComponent,
        MapsComponent,
        NotificationsComponent,
        UpgradeComponent,
    ]
    })

    export class AdminLayoutModule {}

---

### Affichage personnalisée ###

Une amélioration de la page UserProfile

user-profile.component.ts

    import { Component, OnInit } from '@angular/core';
    import { ApiService } from 'app/services/api.service';

    @Component({
    selector: 'app-user-profile',
    templateUrl: './user-profile.component.html',
    styleUrls: ['./user-profile.component.css']
    })
    export class UserProfileComponent implements OnInit {

    user: any;
    public errorMessage: any;

    constructor(private apiService: ApiService) { }

    ngOnInit() {
        this.apiService.getConnectedUser()
        .subscribe(data => {
        this.user = data;
        }, err => {
        console.log('err.error.message : ' + err.error.message);
        this.errorMessage = err.error.message;
        });
    }

    }

---

user-profile.component.html

    <div class="main-content">
    <div class="container-fluid">
        <div class="row">
        <div class="col-md-8">
            <div class="card">
            <div class="card-header card-header-danger">
                <h4 class="card-title">Your Profile</h4>
                <p class="card-category">data from keycloak</p>
            </div>
            <div class="card-body">
                <div class="row">
                <div class="col-md-3">
                    <mat-form-field class="example-full-width">
                    Username
                    </mat-form-field>
                </div>
                <div class="col-md-9">
                    <mat-form-field class="example-full-width">
                    {{user?.username}}
                    </mat-form-field>
                </div>
                </div>
                <div class="row">
                <div class="col-md-3">
                    <mat-form-field class="example-full-width">
                    email
                    </mat-form-field>
                </div>
                <div class="col-md-9">
                    <mat-form-field class="example-full-width">
                    {{user?.email}}
                    </mat-form-field>
                </div>
                </div>
                <div class="row">
                <div class="col-md-3">
                    <mat-form-field class="example-full-width">
                    firstName
                    </mat-form-field>
                </div>
                <div class="col-md-9">
                    <mat-form-field class="example-full-width">
                    {{user?.firstName}}
                    </mat-form-field>
                </div>
                </div>
                <div class="row">
                <div class="col-md-3">
                    <mat-form-field class="example-full-width">
                    lastName
                    </mat-form-field>
                </div>
                <div class="col-md-9">
                    <mat-form-field class="example-full-width">
                    {{user?.lastName}}
                    </mat-form-field>
                </div>
                </div>
                <div class="row">
                <div class="col-md-3">
                    <mat-form-field class="example-full-width">
                    space
                    </mat-form-field>
                </div>
                <div class="col-md-9">
                    <mat-form-field class="example-full-width">
                    {{user?.space?.name}}
                    </mat-form-field>
                </div>
                </div>
                <div class="clearfix"></div>
            </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="card card-profile">
            <div class="card-avatar">
                <a href="javascript:void(0)">
                <img class="img" src="./assets/img/faces/marc.jpg" />
                </a>
            </div>
            <div class="card-body">
                <h6 class="card-category text-gray">CEO / Co-Founder</h6>
                <h4 class="card-title">Alec Thompson</h4>
                <p class="card-description">
                Don't be scared of the truth because we need to restart the human foundation in truth And I love you like
                Kanye loves Kanye I love Rick Owens’ bed design but the back is...
                </p>
                <a href="javascript:void(0)" class="btn btn-danger btn-round">Follow</a>
            </div>
            </div>
        </div>
        </div>
    </div>
    </div>

---

Enfin une récupération de tous les utilisateurs du back-end avec une pagination customisée

table-list.component.ts

    import { Component, OnInit } from '@angular/core';
    import { ApiService } from 'app/services/api.service';

    @Component({
    selector: 'app-table-list',
    templateUrl: './table-list.component.html',
    styleUrls: ['./table-list.component.css']
    })
    export class TableListComponent implements OnInit {

    users: any;
    public errorMessage: any;
    pageNo: number;
    pageSize: number;
    sortBy;

    constructor(private apiService: ApiService) { }

    ngOnInit() {
        this.pageNo = 0;
        this.pageSize = 10;
        this.sortBy = 'id';
        this.getUsers(this.pageNo, this.sortBy);
    }

    getUsers(pageNo: number, sortBy: string) {
        this.apiService.getUsers(pageNo, this.pageSize, this.sortBy)
        .subscribe(data => {
            this.users = data;
            this.cancelNext(this.users.length);
        }, err => {
            console.log('err.error.message : ' + err.error.message);
            this.errorMessage = err.error.message;
        });
    }

    navigateBefore() {
        if (this.pageNo !== 0) {
        this.pageNo -= 1;
        this.getUsers(this.pageNo, this.sortBy);
        }
    }

    navigateNext() {
        this.pageNo += 1;
        this.getUsers(this.pageNo, this.sortBy);
    }

    cancelNext(size: number) {
        console.log('size : ' + + size);
        if (size === 0) {
        this.pageNo -= 1;
        this.getUsers(this.pageNo, this.sortBy);
        }
    }

    sortById() {
        this.sortBy = 'id';
        this.getUsers(this.pageNo, this.sortBy);
    }

    sortByGender() {
        this.sortBy = 'gender';
        this.getUsers(this.pageNo, this.sortBy);
    }

    sortByUsername() {
        this.sortBy = 'username';
        this.getUsers(this.pageNo, this.sortBy);
    }

    sortByFirstName() {
        this.sortBy = 'firstName';
        this.getUsers(this.pageNo, this.sortBy);
    }

    sortByLastName() {
        this.sortBy = 'lastName';
        this.getUsers(this.pageNo, this.sortBy);
    }

    sortByEmail() {
        this.sortBy = 'email';
        this.getUsers(this.pageNo, this.sortBy);
    }

    sortByStatus() {
        this.sortBy = 'status';
        this.getUsers(this.pageNo, this.sortBy);
    }

    }

---

table-list.component.html

    <div class="main-content">
        <div class="container-fluid">
            <div class="row">
                <div class="col-md-12">
                    <div class="card card-plain">
                        <div class="card-header card-header-danger">
                            <h4 class="card-title mt-0"> Table on Plain Background</h4>
                            <p class="card-category"> Here is a subtitle for this table</p>
                        </div>
                        <div class="card-body">
                            <div class="table-responsive">
                                <table class="table table-hover">
                                    <thead class="">
                                        <th>
                                            <a (click)="sortById()">Id</a>
                                        </th>
                                        <th>
                                            <a (click)="sortByGender()">Gender</a>
                                        </th>
                                        <th>
                                            <a (click)="sortByUsername()">Username</a>
                                        </th>
                                        <th>
                                            <a (click)="sortByFirstName()">First Name</a>
                                        </th>
                                        <th>
                                            <a (click)="sortByLastName()">Last Name</a>
                                        </th>
                                        <th>
                                            <a (click)="sortByEmail()">Email</a>
                                        </th>
                                        <th>
                                            <a (click)="sortByStatus()">Status</a>
                                        </th>
                                        <th>
                                            Space
                                        </th>
                                    </thead>
                                    <tbody>
                                        <tr *ngFor="let user of users">
                                            <td>
                                                {{user.id}}
                                            </td>
                                            <td>
                                                {{user.gender}}
                                            </td>
                                            <td>
                                                {{user.username}}
                                            </td>
                                            <td>
                                                {{user.firstName}}
                                            </td>
                                            <td>
                                                {{user.lastName}}
                                            </td>
                                            <td>
                                                {{user.email}}
                                            </td>
                                            <td>
                                                {{user.status}}
                                            </td>
                                            <td>
                                                {{user.space.name}}
                                            </td>
                                        </tr>
                                        <tr>
                                            <td>
                                            </td>
                                            <td>
                                            </td>
                                            <td>
                                            </td>
                                            <td>
                                            </td>
                                            <td>
                                                <a class="clickable nav-link" (click)="navigateBefore()">
                                                    <i class="material-icons">navigate_before</i>
                                                    <p>
                                                        <span class="d-lg-none d-md-block">Before</span>
                                                    </p>
                                                </a>
                                            </td>
                                            <td>
                                                <a class="clickable nav-link" (click)="navigateNext()">
                                                    <i class="material-icons">navigate_next</i>
                                                    <p>
                                                        <span class="d-lg-none d-md-block">Next</span>
                                                    </p>
                                                </a>
                                            </td>
                                        </tr>
                                    </tbody>
                                </table>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

Vous pouvez maintenant lancer l'application en mode local

    ng serve --open

## Conclusion ##

Vous disposer d'une application en micro service comprenant un serveur d'authentification, d'un back-end en java et d'un Front-end avec Angular. L'application est protégé de manière optimale avec un token de session jwt. Tout est géré par Keycloak. L'implémentation est relativement simple. Dans la rubrique Gitlab vous verrez comment tester et déployer le projet back-end. Je compte si le temps me le permet de faire de l'intégration et du déploiment continue avec le front-end sous docker.

PS: Cet article a été écrit relativement vite, ne vous offusquez pas des fautes d'orthographes ou erreurs de syntaxes. Je compte l'améliorer au fur et à mesure!