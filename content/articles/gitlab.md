---
title: "Gitlab"
date: 2020-04-26T16:20:53+02:00
draft: false
author: "Jeannory"
tags: ["articles"]
categories: ["Gitlab"]
---
## Introduction ##

Gitlab fournit un ensemble de services complet pour l'intégration et le déploiement continue.
Il est nécessaire de créer et de renseigner le fichier .gitlab-ci.yml à la racine du projet, pour qu'à chaque commit sur une branche cible, Gitlab exécute les tâches (stages). 

Exemple d'utilisation avec le projet Back-end de <a href="/blog/articles/keycloak/">Keycloak</a> pour la réalisation des différentes étapes CI/CD.

---

### Dépôts Gitlab ###

* [Back-end](https://gitlab.com/phou.jeannory/keycloak-back-end.git)

---

## Gitlab-CI ##

L'intégration continue permet à chaque commit de tester le build du projet, l'exécution des tests, les affichages de différents rapports (qualité de code et couverture des tests) et l'archivage des différents artifacts.
Les pipelines correspond à un cycle complet, qui contiennent des jobs (stages).

![pipelines-jobs](/blog/img/gitlab-07.png)

---

### Build du projet ###

L'image maven:3.3.9-jdk-8 sera utilisée dans différents stages. 
Le runner va mettre en cache les dépendances maven et le contenu du target ce qui va engendrer un gain de temps, ressources et de connexion.

    image: maven:3.3.9-jdk-8

    variables:
        MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version"
        MAVEN_OPTS: "-Dmaven.repo.local=${CI_PROJECT_DIR}/.m2"

    cache:
        key: ${CI_COMMIT_REF_SLUG} # cache is for per branch
        paths:
            - ${CI_PROJECT_DIR}/.m2
            - target/

    stages:
        - build

    build_job:
        stage: build
        script:
            - 'mvn $MAVEN_CLI_OPTS package -Pdev -DskipTests'
        only:
            - dev-docker

---

Au prochain commit sur la branche dev-docker, les dépendances et le build du projet sont enregistrés dans le cache.

    Running after_script
    00:02
    Saving cache
    00:14
    Creating cache dev-docker...
    /builds/phou.jeannory/keycloak-back-end/.m2: found 3366 matching files 
    target/: found 200 matching files                  
    Uploading cache.zip to https://storage.googleapis.com/gitlab-com-runners-cache/project/18369362/dev-docker 
    Created cache
    Uploading artifacts for successful job
    00:01
    Job succeeded

---

### Les tests unitaires ###

Lancement des tests unitaires en utilisant le profil dev-docker.

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

    junit-tests:
        stage: junit-tests
        script:
            - 'mvn $MAVEN_CLI_OPTS test -Pdev-docker'
        only:
            - dev-docker

Les logs affichent une opération réussie.

    [INFO] Results:
    [INFO] 
    [INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0

---

### Couverture de code ###

Utilisation du pluglin Jacoco pour afficher le coverage dans Gitlab.

    <plugin>
        <groupId>org.jacoco</groupId>
        <artifactId>jacoco-maven-plugin</artifactId>
        <version>0.8.5</version>
        <executions>
            <execution>
                <goals>
                    <goal>prepare-agent</goal>
                </goals>
            </execution>
            <!-- attached to Maven test phase -->
            <execution>
                <id>report</id>
                <phase>test</phase>
                <goals>
                    <goal>report</goal>
                </goals>
            </execution>
        </executions>
    </plugin>

---

Jacoco édite un rapport complet dans le target du projet. Le runner va le récupérer du cache pour le copier dans un autre répertoire (junit-tests), accessible depuis l'IHM de Gitlab.

    junit-tests-coverage:
        stage: junit-tests-coverage
        script:
            - mkdir public
            - mv target/site/* junit-tests
        artifacts:
            paths:
            - junit-tests
        only:
            - dev-docker

---

Affichage du rapport : cliquez sur Browse et naviguez jusqu'à index.html.

![chemin](/blog/img/gitlab-02.png)

---

Les différents coverages sont détaillés dans un dashboard.

![rapport tests unitaires](/blog/img/gitlab-03.png)

---

### Qualité du code ###

La qualité du code, une aide non négligeable pour tous développeurs.

    code_quality_job:
        stage: quality
        image: docker:stable
        allow_failure: true
        services:
            - docker:stable-dind
        script:
            - mkdir codequality-results
            - docker run
            --env CODECLIMATE_CODE="$PWD"
            --volume "$PWD":/code
            --volume /var/run/docker.sock:/var/run/docker.sock
            --volume /tmp/cc:/tmp/cc
            codeclimate/codeclimate analyze -f html > ./codequality-results/index.html
        artifacts:
            paths:
            - codequality-results/
        only:
            - dev-docker

---

Affichage du rapport.

![rapport qualité de code](/blog/img/gitlab-04.png)

---

### Les tests d'intégration ###

Les tests d'intégration nécessitent une connexion vers un serveur distant et vers le serveur d'authentification Keycloak.

* Le serveur applicatif : Machine avec Ubuntu Server 18, ip dynamique et DNS, i5 8Go de ram.
* Le serveur d'authentification : VM de Google Cloud Plateform avec Ubuntu Server 18, 1 vCPU, 1,7 Go de mémoire.

Google offre un crédit de 268€70 ce qui laisse une certaine marge pour utiliser ses serveurs.

---

Connexion sftp (transfert de fichier sécurisé) à la VM de GCP.

Générer une paire clef privée/publique pour l'user enregistré dans la VM (exemple: phou_jeannory).
Je vous recommande de choisir un répertoire différent de celui de .ssh.
Il n'est pas nécessaire d'attribuer un passphrase.

    ssh-keygen -t rsa -b 4096 -C "phou_jeannory"

Filezilla, ouvrir edition/paramètre/sftp/ajouter un fichier de clé. Ajouter la clef privée, soit id_rsa.

![filezilla](/blog/img/gitlab-10.png)

Dans la console d'admin de GCP, ouvrir Compute engine/Métadonnées, sélectionner ajouter un élément et saisir la clef public (id_rsa.pub).

![Gcp consol](/blog/img/gitlab-11.png)

Déposer le fichier contenant les images de Keycloak et de sa base de données (répertoires dev-docker-integration-keycloak et keycloak-config)

---

Depuis la console d'admin de GCP, ouvrir une connexion ssh et installer docker.

    sudo apt-get update
    sudo apt-get install docker-compose
    sudo apt-get install docker.io

Builder et Runner les images.

    cd /home/phou_jeannory/dev-docker-integration-keycloak
    sudo docker-compose -f docker-compose.yml up -d

Restaurer la base de données.

    sudo cd /home/phou_jeannory/keycloak-config
    sudo cat backup.sql | docker exec -i mysql-dev-docker /usr/bin/mysql -u keycloak -ppassword keycloak

Le service keycloak est accessible sur le port 8099. Il faut modifier les règles de pare-feu pour que ce service soit accessible au protocole http (se référer à la [documentation](https://cloud.google.com/vpc/docs/using-firewalls?hl=fr)).

---

Prérecquis pour le serveur d'applications:

* Définir des règles de pare-feu pour la base de données.
* Installer Docker.
* Générer une paire clé privée/publique pour partage avec Gitlab.
* Transférer le contenu du répertoire dev-docker-integration-app-environment.
* Générer et exécuter l'image du docker-compose.yml.

---

Configuration de Gitlab pour la connexion vers le serveur d'applications.

Ouvrir le menu Settings/CI/CD/Variables et affecter la clé privée à une variable.

![private key](/blog/img/gitlab-01.png)

---

Le runner doit ouvrir une connexion distante vers la VM, et ouvrir le port du serveur Postgresql le temps du test. Il s'agit d'une solution provisoire en attendant de créer un tunnel ssh...
Il est nécessaire pour cette opération de lancer la commande en super utilisateur, le mot de passe est enregistré dans une variable et non affiché dans les logs.


    enable-remote-bdd:
        stage: enable-remote-bdd
        image: ubuntu:latest
        before_script:
            - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client -y)'
            - mkdir -p ~/.ssh
            - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa
            - chmod 700 ~/.ssh/id_rsa
            - eval "$(ssh-agent -s)"
            - ssh-add ~/.ssh/id_rsa
            - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
        script:
            - ssh-copy-id digital@$X230_SERVER_1 -p "18380"
            - ssh -t digital@$X230_SERVER_1 -p "18380" << END
            - pwd
            - exec 3>&1 1>/dev/null 2>&1
            - echo "$X230_PASSWORD_1" | sudo -S ufw allow 5432
            - sudo -S ufw status >&3
            - END
        only:
            - dev-docker

---

Utilisation du profil maven dev-docker-integration-testing et du profil Springboot dev-docker-integration pour lancer les cycles clean verify.

Le clean est dans cette étape nécessaire pour effacer le rapport des tests unitaires du cache.

    integration-tests:
        stage: integration-tests
        script:
            - 'mvn $MAVEN_CLI_OPTS clean verify -Pdev-docker-integration-testing -Dspring.profiles.active=dev-docker-integration'
        only:
            - dev-docker

---

Fichier pom.xml.

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

Fichier application.yml.

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
            ...

---

Résultat des tests.

    [INFO] Results:
    [INFO] 
    [INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0

---

Fermeture du port de la Base de données au niveau de la VM.

    deny-remote-bdd:
        stage: deny-remote-bdd
        image: ubuntu:latest
        before_script:
            - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client -y)'
            - mkdir -p ~/.ssh
            - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa
            - chmod 700 ~/.ssh/id_rsa
            - eval "$(ssh-agent -s)"
            - ssh-add ~/.ssh/id_rsa
            - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
        script:
            - ssh-copy-id digital@$X230_SERVER_1 -p "18380"
            - ssh -t digital@$X230_SERVER_1 -p "18380" << END
            - pwd
            - exec 3>&1 1>/dev/null 2>&1
            - echo "$X230_PASSWORD_1" | sudo -S ufw deny 5432
            - sudo -S ufw status >&3
            - END
        only:
            - dev-docker

---

Port 5432 en DENY.

    $ pwd
    /home/digital
    $ exec 3>&1 1>/dev/null 2>&1
    Status: active
    To                         Action      From
    --                         ------      ----
    18317                      DENY        Anywhere                  
    8080                       DENY        Anywhere                  
    18380                      ALLOW       Anywhere                  
    8099                       ALLOW       Anywhere                  
    8090                       DENY        Anywhere                  
    8081                       ALLOW       Anywhere                  
    5432                       DENY        Anywhere                  
    18317 (v6)                 DENY        Anywhere (v6)             
    8080 (v6)                  DENY        Anywhere (v6)             
    18380 (v6)                 ALLOW       Anywhere (v6)             
    8099 (v6)                  ALLOW       Anywhere (v6)             
    8090 (v6)                  DENY        Anywhere (v6)             
    8081 (v6)                  ALLOW       Anywhere (v6)             
    5432 (v6)                  DENY        Anywhere (v6)

---

Affichage du nouveau rapport de coverage de tests qui ne contient que ceux des tests d'intégration.

    integration-tests-coverage:
        stage: integration-tests-coverage
        script:
            - mkdir public
            - mv target/site/* integration-tests
        artifacts:
            paths:
            - integration-tests
        only:
            - dev-docker

---

![rapport tests d'intégration](/blog/img/gitlab-05.png)

---

### Archivage d'artifacts ###

toDo

---

## Gitlab-CD ##

À partir du package généré, le runner va build une image de l'application et le déployer sur un serveur distant.

---

### Container registry ###

Le dépôt Gitlab comprend un container registry, qui permet de déposer sa propre image.
Pour cela le runner va récupérer le jar généré (keycloakApplication.jar), et build l'image en utilisant les instructions du Dockerfile.

Fichier pom.xml.

	<build>
		<finalName>keycloakApplication</finalName>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
			<!-- create executable jar -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-jar-plugin</artifactId>
				<version>3.2.0</version>
				<configuration>
					<archive>
						<manifest>
							<addClasspath>true</addClasspath>
							<mainClass>example.KeycloakApplication</mainClass>
							<classpathPrefix>lib/</classpathPrefix>
						</manifest>
					</archive>
				</configuration>
			</plugin>
			<!-- copy dependencies / jars to ${project.build.directory}/lib/ -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-dependency-plugin</artifactId>
				<version>3.1.1</version>
				<executions>
					<execution>
						<id>copy-dependencies</id>
						<phase>package</phase>
						<goals>
							<goal>copy-dependencies</goal>
						</goals>
						<configuration>
							<outputDirectory>
								${project.build.directory}/lib/
							</outputDirectory>
						</configuration>
					</execution>
				</executions>
			</plugin>
    ...

Build de l'image.

Fichier Dockerfile.

    FROM java:8
    ARG JAR_FILE=target/keycloakApplication.jar
    ARG JAR_LIB_FILE=target/lib/

    # cd /usr/local/runme
    WORKDIR /usr/local/runme

    # cp target/keycloakApplication.jar /usr/local/runme/keycloakApplication.jar
    COPY ${JAR_FILE} keycloakApplication.jar

    # cp -rf target/lib/  /usr/local/runme/lib
    ADD ${JAR_LIB_FILE} lib/

    # java -jar /usr/local/runme/keycloakApplication.jar
    ENTRYPOINT ["java","-jar","keycloakApplication.jar"]

---

Le runner va build l'image et le déposer dans le container registry du dépôt gitlab.

    build-app-image:
        stage: build-app-image
        image: docker:latest
        services:
            - docker:dind
        before_script:
            - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
        script:
            - docker build --pull -t "$CI_REGISTRY_IMAGE" .
            - docker push "$CI_REGISTRY_IMAGE"
        only:
            - dev-docker

---

Logs de gitlab.

    Step 6/7 : ADD ${JAR_LIB_FILE} lib/
    ---> 4b121b6cf9d4
    Step 7/7 : ENTRYPOINT ["java","-jar","keycloakApplication.jar"]
    ---> Running in 2750afc0520d
    Removing intermediate container 2750afc0520d
    ---> 7d038e792089
    Successfully built 7d038e792089
    Successfully tagged registry.gitlab.com/phou.jeannory/keycloak-back-end:latest
    $ docker push "$CI_REGISTRY_IMAGE"
    The push refers to repository [registry.gitlab.com/phou.jeannory/keycloak-back-end]

---

L'image a été déposée dans le container registry.

![container registry](/blog/img/gitlab-08.png)

---

### Déploiement de l'application back-end ###

Différentes étapes du stage deploy-app:

* Connexion sur la VM distante.
* Arrêt de l'application déployé.
* Connexion au container registry de Gitlab.
* Exécution du docker-compose.yml qui va redéployer la dernière version de l'app.

---

    deploy-app:
        stage: deploy-app
        image: ubuntu:latest
        before_script:
            - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client -y)'
            - mkdir -p ~/.ssh
            - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa
            - chmod 700 ~/.ssh/id_rsa
            - eval "$(ssh-agent -s)"
            - ssh-add ~/.ssh/id_rsa
            - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
        script:
            - ssh-copy-id digital@$X230_SERVER_1 -p "18380"
            - ssh -t digital@$X230_SERVER_1 -p "18380" << END
            - pwd
            - docker stop devdockerintegrationapp_java_1 # stop and delete the image of app
            - docker rm devdockerintegrationapp_java_1
            - echo "y" | docker system prune -a
            - docker login registry.gitlab.com -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" # build new image of app from container registry
            - cd /home/digital/keycloak-project/dev-docker-integration-app
            - docker-compose -f docker-compose.yml up -d # name of image is the folder + java_1 ==> devdockerintegrationapp_java_1
            - docker ps -a
            - docker images ls
            - END
        only:
            - dev-docker

---

docker-compose.yml du répertoire /home/digital/keycloak-project/dev-docker-integration-app.

Il est important de spécifier dans le docker-compose d'ajouter la ligne 'network_mode: "host"', cela évite des plantages, notamment à cause d'un échec de connexion avec la base de données PostgreSql.
Le profil Springboot pour le déploiement est dev-docker-integration.

    version: '3.1'
        services:
        java:
            image: registry.gitlab.com/phou.jeannory/keycloak-back-end:latest
            restart: always
            network_mode: "host"
            environment:
            - "SPRING_PROFILES_ACTIVE=dev-docker-integration-deploy"

application.yml.

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
        auth-server-url: http://jeannory.ovh/auth/
        resource: sc-user
        public-client: false
        bearer-only: true
        cors: true
        ssl-required: none

    sc-properties:
        authServerUrl: http://jeannory.ovh/auth/
        authServerParameters: realms/Sc-project/protocol/openid-connect/token

---

Les logs de Gitlab indiquent que l'opération a été un succès.
L'application java est correctement déployée. Son appelation est devdockerintegrationapp_java_1. La commande docker ps -a ne nous donne pas d'indication concernant son exposition (colonne port).

Ce service utilise le port spécifié dans le fichier de configuration application.yml, soit le port 8081 en localhost.

    $ docker ps -a
    CONTAINER ID        IMAGE                                                        COMMAND                  CREATED             STATUS                  PORTS                    NAMES
    d4fbbbdf0933        registry.gitlab.com/phou.jeannory/keycloak-back-end:latest   "java -jar keycloakA…"   8 seconds ago       Up Less than a second                            devdockerintegrationapp_java_1
    b2eab32d7a21        postgres                                                     "docker-entrypoint.s…"   2 hours ago         Up 2 hours              0.0.0.0:5432->5432/tcp   postgresql-dev-docker
    $ docker images ls
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    $ END
    Running after_script
    00:02
    Saving cache
    00:13
    Creating cache dev-docker-05...
    /builds/phou.jeannory/keycloak-back-end/.m2: found 3681 matching files 
    target/: found 199 matching files                  
    Uploading cache.zip to https://storage.googleapis.com/gitlab-com-runners-cache/project/18369362/dev-docker-05 
    Created cache
    Uploading artifacts for successful job
    00:02
    Job succeeded

### Test du projet déployé ###

Test à faire en attendant de déployer le front-end sur un serveur, exécuter le client sur le poste du développeur avec le profil prod.

    ng serve --prod

L'UI va se connecter au serveur d'application jeannory.dynamic-dns.net, et vers le serveur d'authentification jeannory.ovh.

    export const environment = {
        SC_USER_BASE_URL: 'jeannory.dynamic-dns.net',
        SC_USER_PORT: '8081',
        AUTH_BASE_URL: 'jeannory.ovh',
        AUTH_PORT: '8099',
        // APP_VERSION: version,
        production: true,
    };

---

L'utilisateur s'authentifie via le serveur Keycloak et le serveur d'application répond avec un statut code 200.
Le test est un succès.

![keycloak-ui](/blog/img/gitlab-14.png)

---

Merci!

