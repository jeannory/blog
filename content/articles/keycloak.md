---
title: "Keycloak"
date: 2020-04-23T19:40:34+02:00
draft: false
author: "Jeannory"
tags: ["articles"]
categories: ["Keycloak"]
---
## Introduction ##

*MAJ au 16/07/2020 reprise de cet article après une courte pause de 3 mois :)*

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

### Serveur d'authentification 100 % portable avec docker ###

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

toDo

### Les tests ###

toDo

---
