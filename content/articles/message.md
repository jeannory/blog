---
title: "Message"
date: 2020-07-30T23:14:42+02:00
draft: false
author: "Jeannory"
tags: ["articles"]
categories: ["Message"]
---
## Introduction ##

*MAJ au 10/10/2020.*

Dans ce chapitre nous allons aborder la communication asynchrone entre micro-services à travers le développement de composants.

Stack technique

* VM Google Cloud Plateform
* Message oriented middleware activeMQ
* Back-end Java/Springboot
* Module Java/SpringBoot
* Serveur d'authentification keycloak
* Front-end Angular Material

### User storie 1###

Given:

* Je peux cliquer sur le bouton "spam les managers"

When:

* L'utilisateur n'est pas connecté

Then:

* Incrémentation du nombre de spams des utilisateurs non connectés

And:

* Tous les 10 spams, le serveur envoi un message à l'ensemble des managers connectés, ou il est indiqué le nombre de spam recus ainsi que la date et l'heure du 1er spam envoyé

### User storie 2###

Given:

* Je peux cliquer sur le bouton "spam les managers"

When:

* L'utilisateur est connecté

Then:

* Incrémentation du nombre de spams des utilisateurs connectés

And:

* Tous les 10 spams, le serveur envoi un message à l'ensemble des managers connectés, ou il est indiqué le nombre de spam recus ainsi que la date et l'heure du 1er spam envoyé

---

### Spécificité technique###

L'IHM envoie une requête http vers le serveur user-app-server, qui transmet un java message service (JMS) vers un autre micro-service (spam-app-server). Toutes les 10 incrémentations spam-app envoi un message push (web-Socket) vers l'IHM.
Le serveur d'authentification va permettre à l'utilisateur de se connecter, et contrôler le header des requêtes lorsque les api sont protégés.

---

### Schéma ###

Infrastructure de l'application

![Infrastructure](/blog/img/Message-app-img-03.png)

---

Parcours de la fonction

![Message-app-diag1](/blog/img/Message-app-img-04.png)

* 1/Envoi de la requête http vers le back-end user-app
* 2/Envoi du message vers le serveur activeMq
* 3/Récupération du message par le module spam-app
* 4/Envoi d'une notification vers le client

---

## Requête client-server ##

### Page Front-end  ###

Nous allons créer la page qui va permettre cette action

    ng g c Spam

---

    <div class="main-content">
        <div class="card mg-card">
            <img lass="card-img-top"
            src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcS52y5aInsxSm31CvHOFHWujqUx_wWTS9iM6s7BAm21oEN_RiGoog"
            alt="Card image" />
            <div class="card-body">
                <h4 class="card-title">Thread services</h4>
                <p class="card-text">Spam your manager</p>
                <a href="#" class="btn btn-danger">Here</a>
            </div>
        </div>
    </div>

![keycloak-clients](/blog/img/message-01.png)

sidebar.component.ts

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
        { path: '/spam', title: 'Spam',  icon: 'spam', class: '' },
    ];

Ajouter à la fin de sidebar.component.html

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
            <a class="nav-link" [routerLink]="[menuItem.path]"
            *ngIf="!securityService.kc.authenticated && menuItem.title == 'Spam'">
            <i class="material-icons">{{menuItem.icon}}</i>
            <p>{{menuItem.title}}</p>
        </a>
        </li>
    </ul>

Ajouter au module admin-layout.module.ts

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
        SpamComponent
    ]

Autoriser tous les utilisateur à accéder à cette nouvelle page.
Modifier admin-layout.routing.ts

    {
        path: 'home',
        component: HomeComponent
    },
    {
        path: 'spam',
        component: SpamComponent
    },

---

Un utilisateur connecté ou non pourra ainsi accéder à cette page

---

![keycloak-clients](/blog/img/message-03.png)

---

![keycloak-clients](/blog/img/message-02.png)

---

### Service http ###

Coté back-end, une api va récupérer les réquêtes http du client et renvoyer un SpamDto. Il va renvoyer l'instant présent avec un boolean : un true pour un utlisateur connecté et un false pour un utilisateur non connecté.

    @Data
    public class SpamDto {

        private boolean isConnected;
        private String now;

        public SpamDto() {
            Date date = new Date();
            SimpleDateFormat formatter = new SimpleDateFormat("dd-MM-yyyy HH:mm:ss");
            now = formatter.format(date);
        }

    }

---

Pour cela dans la class KeycloakSecurityContext nous allons rajouter la méthode qui va vérifier si l'utilisateur est connecté.

    @Bean
    @Scope(scopeName = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
    private KeycloakAuthenticationToken getKeycloakAuthenticationToken() {
        KeycloakAuthenticationToken keycloakAuthenticationToken = null;
        try {
            keycloakAuthenticationToken = (KeycloakAuthenticationToken) ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest().getUserPrincipal();
        } catch (NullPointerException ex) {
            //user is not connected
        }
        return keycloakAuthenticationToken;
    }

    public boolean isConnected() {
        boolean isConnected = true;
        if (getKeycloakAuthenticationToken() == null) {
            isConnected = false;
        }
        return isConnected;
    }

Le service SpamServiceImpl va instancier et retourner le SpamDto au controlleur

    @Autowired
    private KeycloakSecurityContext keycloakSecurityContext;

    @Override
    public SpamDto getSpamDto(){
        final SpamDto spamDto = new SpamDto();
        spamDto.setConnected(keycloakSecurityContext.isConnected());
        return spamDto;
    }

---

La simple requête http://localhost:8081/api/spam/getSpam va retourner le résultat demandé

    @CrossOrigin("*")
    @RestController
    @RequestMapping("api/spam")
    public class SpamController {

        @Autowired
        private SpamService spamService;

        @RequestMapping(path = "/getSpam", method = RequestMethod.GET)
        public ResponseEntity<SpamDto> getSpam(){
            return new ResponseEntity<SpamDto>(spamService.getSpamDto(), new HttpHeaders(), HttpStatus.OK);
        }
    }

---

Pour cette fonction nous n'allons pas ajouter de test-unitaire par simplicité (les dépendances de keycloakAuthenticationToken sont difficiles à gérer). Nous allons ajouter 2 tests d'intégrations :

    /**
    * Initialize spring context before the class
    */
    @RunWith(SpringRunner.class)
    @SpringBootTest
    @AutoConfigureMockMvc
    /**
    * ActiveProfiles must be launch by maven
    * mvn test -Pdev-local -Dspring.profiles.active=dev-docker
    */
    //@ActiveProfiles("dev-docker")
    public class SpamControllerTest extends BuilderUtilsResultAction {
        @Autowired
        private BuilderUtilsKeycloak builderUtilsKeycloak;

        @Test
        public void getSpamShouldReturnTrueWhenUserIsConnected() throws Exception {
            final String accessToken = builderUtilsKeycloak.getAccessToken("test", "1234");
            final MvcResult mvcResult = invokeGet("/api/spam/getSpam", accessToken)
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("connected", is(true)))
                    .andReturn();
        }

        @Test
        public void getSpamShouldReturnFalseWhenUserIsNotConnected() throws Exception {
            final MvcResult mvcResult = invokeGet("/api/spam/getSpam")
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("connected", is(false)))
                    .andReturn();
        }
    }

---

Coté front-end nous allons rajouter la requête qui ira consommer cette api

Ajouter les constantes dans la class ApiUrl

    private static readonly SPAM = '/spam/getSpam';

---

    static get GET_SPAM(): string {
        return `${this.BASE_SC_USER_URL}${this.BASE_API}${this.SPAM}`;
    }

---

spam.component.html:

      <a class="btn btn-danger" (click)="spamManager()">Here</a>

---

    export class SpamComponent implements OnInit {

    spam: any;

    constructor(private apiService: ApiService) { }

    ngOnInit(): void {
    }

    spamManager() {
        this.apiService.getSpam()
        .subscribe(data => {
            this.spam = data;
            console.log('Now : ' + this.spam.now + 'is connected : ' + this.spam.connected);
        }, err => {
            console.log('err.error.message : ' + err.error.message);
        })
    }

    }

---

La console affichera un résultat différent en fonction de l'état de connection de l'utilisateur

![spam-connected](/blog/img/message-05.png)

---

![spam-not-connected](/blog/img/message-06.png)

---

## Message server-server avec activeMQ ##

Comme indiqué dans le diagramme de l'infrastructure du projet, le module spam-app sera dans une vm distincte qui devra contenir un "message-oriented middleware" (MOM). Pour ce micro projet j'ai choisit ActiveMQ de la fondation apache, plus adapté que les 2 autres géants qui sont Kafka et RabbitMq.

Il faut se rendre sur [le site de activeMq](http://activemq.apache.org/), télécharger le serveur et le dézipper. 
Pour le lancer il faut tapper la commande

    ./apache-activemq-5.16.0/bin/activemq start

---

Dépendances maven à ajouter aux projets qui se connecte au serveur

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-activemq</artifactId>
		</dependency>
		<dependency>
			<groupId>org.apache.activemq</groupId>
			<artifactId>activemq-broker</artifactId>
		</dependency>

---

Configuration vers le MOM

    spring:
        profiles: dev-docker #local mode

        activemq:
            broker-url: "tcp://localhost:61616"
            user: "admin"
            password: "admin"

    ---

    spring:
        profiles: dev-docker-integration #integration testing mode

        activemq:
            broker-url: "tcp://35.210.119.52:61616"
            user: "admin"
            password: "admin"

---

    @EnableJms
    @SpringBootApplication
    public class SpamAppApplication {
        public static Connected connected = new Connected();
        public static NotConnected notConnected = new NotConnected();

        @Bean
        public JmsListenerContainerFactory<?> myFactory(ConnectionFactory connectionFactory,
                                                        DefaultJmsListenerContainerFactoryConfigurer configurer) {
            DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
            configurer.configure(factory, connectionFactory);
            return factory;
        }

        @Bean
        public MessageConverter jacksonJmsMessageConverter() {
            MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
            converter.setTargetType(MessageType.TEXT);
            converter.setTypeIdPropertyName("_type");
            return converter;
        }

---

user-app va envoyer un message au module spam-app

Expédition (class SpamServiceImpl)

        jmsTemplate.convertAndSend("spamIncrement", spamDto);
        jmsTemplate.setReceiveTimeout(10_000);


Destination (class SpamMessage)

        @JmsListener(destination = "spamIncrement", containerFactory = "myFactory")

---

Une fois le message consommé, une action pourra être enclenchée par le serveur. Ici il va envoyer un message au client (web-socket).

## Web-socket ##

Je joins ici le lien de l'implémentaton en Springboot/Angular [[lien]](https://medium.com/@haseeamarathunga/create-a-spring-boot-angular-websocket-using-sockjs-and-stomp-cb339f766a98)

---

### Web-socket Springboot ###

Dépendance coté serveur

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-websocket</artifactId>
			<version>1.5.2.RELEASE</version>
		</dependency>

---

Configuration pour l'envoi du message

    @Configuration
    @EnableWebSocketMessageBroker
    public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {
        @Override
        public void registerStompEndpoints(StompEndpointRegistry registry) {
            registry.addEndpoint("/spam-app")
                    .setAllowedOrigins("*")
                    .withSockJS();
        }
        @Override
        public void configureMessageBroker(MessageBrokerRegistry registry) {
            registry.enableSimpleBroker("/message");
        }
    }

---

Envoi du message vers le client

        if (spamDto.isConnected()) {
            SpamAppApplication.connected.increment();
            SpamAppApplication.connected.updateSince(spamDto.getNow());
            if (isSendNotification(SpamAppApplication.connected.getCount())) {
                this.template.convertAndSend("/message", SpamAppApplication.connected.message());
            }
        } else {
            SpamAppApplication.notConnected.increment();
            SpamAppApplication.notConnected.updateSince(spamDto.getNow());
            if (isSendNotification(SpamAppApplication.notConnected.getCount())) {
                this.template.convertAndSend("/message", SpamAppApplication.notConnected.message());
            }
        }

### Web-socket Angular ###

Enregistrer les scripts dans assets et charger les dans index.html

    <script src="assets/sockjs.min.js"></script>
    <script src="assets/stomp.min.js"></script>

Le service

    export class WebSocketService {

        public stompClient;
        public msg = [];

        constructor() {
            this.initializeWebSocketConnection();
        }

        initializeWebSocketConnection() {
            const serverUrl = 'http://localhost:8080/spam-app';
            const ws = new SockJS(serverUrl);
            this.stompClient = Stomp.over(ws);
            const that = this;
            // tslint:disable-next-line:only-arrow-functions
                this.stompClient.connect({}, function (frame) {
                    that.stompClient.subscribe('/message', (message) => {
                        if (message.body) {
                        that.msg.push(message.body);
                        }
                    });
                });
        }
    }

---

Injection de dépendance vers le composant qui va afficher les notifications (NavbarComponent)

    private webSocketService: WebSocketService

---

Affichage des notifications pour le manager connecté (page navbar.component.html)

            <ul class="navbar-nav" *ngIf="securityService.kc.authenticated">
                <li class="nav-item dropdown">
                    <a class="nav-link" href="javascript:void(0)" id="navbarDropdownMenuLink" data-toggle="dropdown"
                        aria-haspopup="true" aria-expanded="false">
                        <i class="material-icons">notifications</i>
                        <span *ngIf="webSocketService.msg.length > 0" class="notification">{{webSocketService.msg.length}}</span>
                    </a>
                    <div class="dropdown-menu dropdown-menu-right" aria-labelledby="navbarDropdownMenuLink">
                        <a *ngFor="let msg of webSocketService.msg" class="dropdown-item">{{msg}}</a>
                    </div>
                </li>
                ...

---

Notification recu coté UI

![notification](/blog/img/Message-app-img-01.png)

---

## Intégration et livraison continue ##

### Spécificité technique###

Dans le cadre de l'intégration et de la livraison continue, voici le schéma de l'infrastructure mit en place, pour le build, test and deploy des projets user-app et de spam-app.

---

![Gitlab-ci-cd](/blog/img/Message-app-img-05.png)
L'offre gratuite proposé par gitlab permet une utilisation poussé du ci/cd à travers les dépôts officiels.
De ce fait et parce que la gestion des certificats est assez couteuse, le serveur gitlab n'a pas été embarqué dans le cloud. 
Ne faisant pas parti du sous réseau, certains ports vont être ouvert le temps des tests et déployement (chose à bannir lors de vrais application d'entreprises).

---

### Configuration des VM avec Google cloud plateform ###

![gcp](/blog/img/Message-app-img-02.png)

### Stage des pipelines de user-app ###

### Stage des pipelines de spam-app ###

### Stage des pipelines du front-end ###

## Conclusion ##
