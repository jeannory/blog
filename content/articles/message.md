---
title: "Message"
date: 2020-07-30T23:14:42+02:00
draft: false
author: "Jeannory"
tags: ["articles"]
categories: ["Message"]
---
## Introduction ##

Dans ce chapitre nous allons aborder la communication asynchrone entre micro-services à travers le développement de composants d'une user storie.

*Ebauche en cours de développement*

### User storie ###

Quand un utilisateur clique sur "spam" un bouton apparait
*spam the managers*

Quand un utilisateur non connecté clique sur le bouton, un service incrémente un compteur 1 sur un thread (thread1) et enregistre la date et l'heure.

Quand un utilisateur connecté clique sur le bouton, un service incrémente un compteur 2 sur un thread (thread2) et enregistre la date et l'heure.

Quand thread1 compte 10 incrémentations il envoi une notification à tous les managers connectés.
La popup affiche : *10 utilisateurs non connectés spam les managers depuis le (date/heure du spam 0)*.
Puis le compteur 1 est remit a zéro.

Quand thread2 compte 10 incrémentations il envoi une notification à tous les managers connectés.
La popup affiche : *10 utilisateurs connectés spam les managers depuis le (date/heure du spam 0)*.
Puis le compteur 2 est remit a zéro.

Spécificité technique : L'IHM envoie une requête http vers le back-end sc-user, qui transmet un Java Message Service (JMS) vers un autre micro-service (SpamApp). Toutes les 10 incrémentations SpamApp envoi un message push (web-Socket) vers l'IHM.

---

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

### Spam service ###

Créer l'appliation Springboot

![keycloak-clients](/blog/img/message-04.png)

