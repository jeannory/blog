---
title: "Gitlab"
date: 2020-04-26T16:20:53+02:00
draft: false
author: "Jeannory"
tags: ["articles"]
categories: ["Gitlab"]
---
## Introduction ##

Gitlab fournit un ensemble de service complet pour l'intégration et le déploiement continue.
Exemple d'utilisation avec le projet Back-end de <a href="/blog/articles/keycloak/">Keycloak</a>.

### Dépôts Gitlab ###

* [Back-end](https://gitlab.com/phou.jeannory/keycloak-back-end.git)
* [Front-end] (https://gitlab.com/phou.jeannory/keycloak-front-end.git)

## Gitlab-CI ##

A chaque commit, Gitlab va lancer les tâches (stages) à partir du fichier .gitlab-ci.yml enregistré à la racine du projet.

### Build du projet ###

L'image maven:3.3.9-jdk-8 sera utilisé dans différents stages. 
Gitlab va mettre en cache les dépendances maven et le contenu du target ce qui va engendrer un gain de temps, ressources et de connexion.

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

Au prochain commit sur la branche dev-docker, les dépendances et le build du projet ont bien été mis dans le cache.

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

### Les tests unitaires ###

Lancement des tests unitaires en utilisant le profile dev-docker

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

Les logs affichent une opération réussis

    [INFO] Results:
    [INFO] 
    [INFO] Tests run: 20, Failures: 0, Errors: 0, Skipped: 0

### Affichage de couverture de code ###

Utilisation du pluglin jacoco pour afficher le coverage dans Gitlab

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

Récupération du site static généré par jacoco dans le target pour l'afficher dans le répertoire junit-tests de Gitlab

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

Affichage du rapport : cliquez sur Browse et naviguez jusqu'à index.html

![chemin](/blog/img/gitlab-02.png)

Les différents coverages sont détaillés dans un dashboard.

![rapport tests unitaires](/blog/img/gitlab-03.png)

### Qualité du code ###

La qualité du code, une aide non négligeable pour tous développeur.

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

Affichage du rapport

![rapport qualité de code](/blog/img/gitlab-04.png)

### Les tests d'intégration ###

Pour les tests d'intégration, il est nécessaire de se connecter à un serveur. Pour cela nous allons utiliser une clef privée.
Ouvrir le menu Settings/CI/CD/Variables et enregistrer la clef

![private key](/blog/img/gitlab-01.png)

Pour les tests gitlab va ouvrir le port du serveur postgresql, puis le refermer. Il s'agit d'une solution provisoire en attendant de créer un tunnel ssh...

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

Les tests d'intégration nécessitent une connexion vers base de données et vers le serveur d'authentification Keycloak (utilisation du profil maven dev-docker-integration-testing et du profil Springboot dev-docker-integration-testing)


    integration-tests:
        stage: integration-tests
        script:
            - 'mvn $MAVEN_CLI_OPTS clean verify -Pdev-docker-integration-testing -Dspring.profiles.active=dev-docker-integration-testing'
        only:
            - dev-docker

pom.xml

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

fichier application.yml

    spring:
        profiles: dev-docker-integration-testing

        jpa:
            database: POSTGRESQL
            show-sql: true
            hibernate:
            ddl-auto: create-drop

        datasource:
            platform: postgres
            url: jdbc:postgresql://jeannory.dynamic-dns.net:5432/keycloak_back_end_db
            username: user1

Résultat des tests

    [INFO] Results:
    [INFO] 
    [INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0

Affichage du rapport

![rapport tests d'intégration](/blog/img/gitlab-05.png)


## Gitlab-CD ##

toDo

### Build de l'application back-end ###

toDo

### Déploiement de l'application back-end ###

toDo
