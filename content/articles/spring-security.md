---
title: "Spring Security"
date: 2019-10-07T23:47:25+02:00
draft: true
author: "Jeannory"
tags: ["articles"]
categories: ["Spring Security"]
---
Il s'agit de mon premier article, les avis/critiques sont le bienvenus.

Vous pouvez retrouver les dépôts git,

* Pour le back-end : [ici](https://github.com/jeannory/SpringSecurityExampleBE)
* Pour le front-end : [ici](https://github.com/jeannory/SpringSecurityExampleFE)

Bonne lecture.

_Article du 23/08/2019 importé en markdown pour Hugo au 08/10/2019._

-----

## Protéger son application avec SpringBoot, Angular et Jwt ##

Développer une application full stack en respectant les règles en terme de cyber sécurité.

Rapide initiation à Spring Security.

Stack technique :

* Java 8
* Springboot 2
* Angular 7

-----

### Projet ###

Le projet se compose de 2 applications. Un Back-end en java et un Front-end en Angular.

Fonctions :

* Créer un compte
* Se connecter
* Modifier un rôle
* Réaliser une action en fonction de son role

Sécurité :

Utilisation d'un token de session Jwt, pour valider les appels entre le Back-end et le Front-end.

Sommaire :

* Back-end : présentation de l'architecture en services web
* Back-end : implémentation de Spring Security
* Back-end : tests de end-point
* Front-end : Présentation du projet + implémentation
* Front-end : Interceptor
* Front-end : CanActivate
* Front-end : tests de fonctionalités

-----

## Back-end : présentation de l'architecture en services web ##

* Persistance des données sur une base de données postgresql
* Management des entités par l'ORM d'hibernate
* Conversion des entités/DTO par ModelMapper
* Exposition des services REST par Springboot

### Back-end : implémentation de Spring Security ###

Au lancement de l'application le singleton produit une liste de JsonWebKey qui va servir pour générer et valider les tokens.

    @Component
    @Scope("singleton")
    public class SingletonBean {

        private final static Logger logger = Logger.getLogger(SingletonBean.class);
        private static List<JsonWebKey> jsonWebKeys;
        private static ModelMapper modelMapper;

        public SingletonBean() {
            logger.info("**********");
            logger.info("Constructor SingletonBean begin");
            List<Integer> listInt = Arrays.asList(1, 2, 3);
            jsonWebKeys = listInt.stream().map(i -> {
                try {
                    JsonWebKey jsonWebKey = RsaJwkGenerator.generateJwk(2048);
                    jsonWebKey.setKeyId(String.valueOf(i));
                    logger.info("JsonWebKeys number : " + i + " generate");
                    return jsonWebKey;
                } catch (JoseException e) {
                    e.printStackTrace();
                    return null;
                }
            }).collect(Collectors.toCollection(ArrayList::new)).stream().filter(Objects::nonNull).collect(Collectors.toList());
            modelMapper = new ModelMapper();
            logger.info("Constructor SingletonBean end");
            logger.info("**********");
        }

        public List<JsonWebKey> getJsonWebKeys() {
            return jsonWebKeys;
        }

        public ModelMapper getModelMapper() {
            return modelMapper;
        }
    }

-----

En important Spring Security, il est nécessaire d'implémenter certaines méthodes.

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

-----

Il faut d'abord créer une class JwtRequestFilter qui hérite de OncePerRequestFilter pour redéfinir doFilterInternal.

    public class JwtRequestFilter extends OncePerRequestFilter {

        private final static Logger logger = Logger.getLogger(JwtRequestFilter.class);
        @Autowired
        private UserDetailsService userDetailsService;
        @Autowired
        private AuthProvider authProvider;
        @Autowired
        private TokenUtilityProvider tokenUtilityProvider;

        @Override
        protected void doFilterInternal(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, FilterChain filterChain) throws ServletException, IOException {
            try {
                String token = validateTokenHeader(httpServletRequest);
                TokenUtility tokenUtility = tokenUtilityProvider.getTokenUtility(token);
                if (tokenUtility.isValidateToken()) {
                    UserDetails userDetails = userDetailsService.loadUserByUsername(tokenUtility.getEmail());
                    if(userDetails!=null){
                        List<SimpleGrantedAuthority> simpleGrantedAuthorities
                                = tokenUtility.getRoles().stream()
                                .map(role -> new SimpleGrantedAuthority(AUTHORITY_PREFIX + role))
                                .collect(Collectors.toCollection(ArrayList::new));
                        UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails, null, simpleGrantedAuthorities);
                        authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(httpServletRequest));
                        SecurityContextHolder.getContext().setAuthentication(authentication);
                        logger.info("Method doFilterInternal - token is valid");
                    }
                }
            } catch (CustomNoHeaderException ex) {
                /**
                * NPE or if token not found on header no status to return
                * rejected if @Secure on controller
                */
                logger.info("Method doFilterInternal : " + ex.getMessage());
            }
            filterChain.doFilter(httpServletRequest, httpServletResponse);
        }

        private String validateTokenHeader(HttpServletRequest httpServletRequest) {
            try {
                final String requestTokenHeader = httpServletRequest.getHeader(HEADER_STRING);
                String token = requestTokenHeader.replace(TOKEN_PREFIX, "");
                if (token == null) {
                    throw new CustomNoHeaderException("token not found on header");
                }
                return token;
            } catch (NullPointerException ex) {
                throw new CustomNoHeaderException("token not found on header");
            }
        }

    }

-----

Le filter va intercepter toutes les requêtes pour récupérer le token dans le header des requests. Le principe est simple, une méthode va en vérifier le contenu et tenter de valider sa signature. Si le test est positive, le(s) rôle(s) de l'utilisateur seront enregistrés dans le context de l'application, dans le cas contraire la response enverra un status 403.

-----

Observez les méthodes getTokenUtility et validateToken de la class TokenUtilityProvider, le besoin est de décoder, convertir le token, afin de récupérer des valeurs. Il faut ensuite vérifier l'authenticité des informations recueillis (utilisation des librairies de jose4j et jjwt). Une fois ces étapes réussies (If(tokenUtility.isValidateToken dans doFilterInternal), la liste des rôles est injectée dans le contexte de l'application. Il suffit qu'une exception soit levée dans la méthode validateTokenUtility pour que l'application envoie une response avec un status 403.

    @Component
    public class TokenUtilityProvider implements ITools, Serializable {

        private final static Logger logger = Logger.getLogger(TokenUtilityProvider.class);
        @Autowired
        SingletonBean singletonBean;
        private static TokenUtility tokenUtility;

        public TokenUtilityProvider() {
        }

        public TokenUtility getTokenUtility(String token){
            try {
                tokenUtility = validateToken(token);
            } catch (CustomMalformedClaimException ex) {
                logger.error(ex.getMessage());
                tokenUtility.setValidateToken(false);
            } catch (CustomInvalidJwtException ex) {
                logger.error(ex.getMessage());
                tokenUtility.setValidateToken(false);
            } catch (CustomJwtException ex) {
                /**
                * if exception in :
                * -TokenUtility.validateToken
                * -TokenUtilityProvider.validateJsonWebKey
                * -TokenUtilityProvider.getStringFromJwtNode
                */
                logger.error(ex.getMessage());
                tokenUtility.setValidateToken(false);
            }
            logger.info("Method getTokenUtility succeed");
            return tokenUtility;
        }

        private TokenUtility validateToken(String token) {
            tokenUtility = new TokenUtility();
            try {
                String kid = getStringFromJwtNode(token, 0, "kid");
                String issuer = getStringFromJwtNode(token, 1, "iss");
                /**
                * not in use
                */
                String exp = getStringFromJwtNode(token, 1, "exp");
                JsonWebKey jsonWebKey = validateJsonWebKey(kid);
                tokenUtility.setKid(Integer.valueOf(kid));
                JwtConsumer jwtConsumer = new JwtConsumerBuilder()
                        .setRequireExpirationTime()
                        .setAllowedClockSkewInSeconds(120)
                        .setRequireSubject()
                        .setExpectedIssuer(issuer)
                        .setVerificationKey(jsonWebKey.getKey())
                        .build();
                JwtClaims jwtClaims = jwtConsumer.processToClaims(token);
                if (jwtClaims == null) {
                    throw new CustomJwtException("token is invalid");
                }
                else{
                    tokenUtility.setEmail(jwtClaims.getSubject());
                    List<String> roleStrings = jwtClaims.getStringListClaimValue(AUTHORITIES_KEY);
                    tokenUtility.setRoles(roleStrings);
                    /**
                    * jwt validation succeeded
                    */
                    tokenUtility.setValidateToken(true);
                    return tokenUtility;
                }
            } catch (MalformedClaimException ex) {
                throw new CustomMalformedClaimException("claim is malFormed");
            } catch (InvalidJwtException ex) {
                throw new CustomInvalidJwtException("jwt is invalid");
            } catch (NullPointerException ex) {
                throw new CustomJwtException("token is invalid");
            }
        }

        /**
        * to calculate expiration of jwt
        */
        private Long millisecondsLeft(String exp) {
            NumericDate jwtNumericDate = NumericDate.fromSeconds(Long.valueOf(exp));
            NumericDate numericDateNow = NumericDate.now();
            Long millisecondsLeft = jwtNumericDate.getValueInMillis() - numericDateNow.getValueInMillis();
            return millisecondsLeft;
        }

        private String getStringFromJwtNode(String token, int indice, String nodeName) {
            try {
                String[] tokenTab = token.split("\\.");
                String headerEncoded = tokenTab[indice];
                byte[] decodeBytesHeader = Base64.getUrlDecoder().decode(headerEncoded);
                String decodeHeader = new String(decodeBytesHeader);
                ObjectMapper objectMapper = new ObjectMapper();
                JsonNode rootNode;
                rootNode = objectMapper.readValue(decodeHeader, JsonNode.class);
                JsonNode node = rootNode.path(nodeName);
                String kid = node.asText();
                return kid;
            } catch (IOException ex) {
                throw new CustomJwtException("failed to get infos from the token");
            } catch (NullPointerException ex) {
                throw new CustomJwtException("failed to get infos from the token");
            }
        }

        private JsonWebKey validateJsonWebKey(String kid) {
            try {
                /**
                * get jsonWebKey by kid
                */
                JsonWebKeySet jsonWebKeySet = new JsonWebKeySet(singletonBean.getJsonWebKeys());
                JsonWebKey jsonWebKey = jsonWebKeySet.findJsonWebKey(kid, null, null, null);
                return jsonWebKey;
            } catch (NullPointerException ex) {
                throw new CustomJwtException("failed to created JsonWebKeySet");
            }
        }
    }

-----

Il faut également créer une classe qui implémente UserDetailsService (soit UserServiceImpl) et redéfinir la méthode loadUserByUsername.

    /**
     * with Spring security
     * @Secure needs to redefine loadUserByUsername(String username)
     * abstract method of UserDetailsService
     */
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        try {
            final User user = manageSelectMyUserByEmailException(username);
            Set<SimpleGrantedAuthority> simpleGrantedAuthorities = roleService.findByUsersEmail(user.getEmail()).stream().map(role -> {
                return new SimpleGrantedAuthority(AUTHORITY_PREFIX + role.getName());
            }).collect(Collectors.toCollection(HashSet::new));
            logger.info("Method loadUserByUsername succeed");
            return new org.springframework.security.core.userdetails.User(user.getEmail(), user.getPassword(), simpleGrantedAuthorities);
        } catch (UsernameNotFoundException ex) {
            logger.error(ex.getMessage());
            return null;
        }
    }

    private User manageSelectMyUserByEmailException(String username) {
        final User user = userRepository.selectMyUserByEmail(username);
        if (
                user == null ||
                        user.getId() == null ||
                        user.getEmail() == null ||
                        user.getPassword() == null ||
                        user.getRoles() == null
        ) {
            throw new UsernameNotFoundException("Invalid user");
        }
        return user;
    }

-----

Enfin il faut créer une class qui hérite de WebSecurityConfigurerAdapter et redéfinir la méthode configure.

    @Configuration
    @EnableWebSecurity
    @EnableGlobalMethodSecurity(securedEnabled = true)
    public class SecurityConfig extends WebSecurityConfigurerAdapter {

        @Bean
        public JwtRequestFilter jwtRequestFilter() {
            return new JwtRequestFilter();
        }

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            /**
            * return status 403 if not allowed
            */
            http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and().
                    authorizeRequests()
                    .antMatchers(HttpMethod.GET, "/api", "/api/*").permitAll()
                    .antMatchers(HttpMethod.GET, "/roles", "/roles/*").hasRole(ADMIN)
                    .antMatchers(HttpMethod.GET, "/users", "/users/*").hasRole(ADMIN)
                    .antMatchers(HttpMethod.POST, "/api/*").permitAll()
                    .antMatchers(HttpMethod.PUT, "/api/*").permitAll()
                    .antMatchers(HttpMethod.DELETE, "/api/*").permitAll().and()
                    .requestCache().requestCache(new NullRequestCache()).and()
                    .cors().and()
                    .csrf().disable();

            /**
            * add custom JWT security filter,if not allowing by @Secure return status 403
            */
            http.addFilterBefore(jwtRequestFilter(), UsernamePasswordAuthenticationFilter.class);
        }
    }

-----

Dans notre controlleur il est possible de limiter les accès aux end-points en fonction des rôles utilisateurs, en utilisant l'annotaion @Secure.

Le end-point getUsers lui est exclusivement réservé aux utilisateurs disposant du ADMIN.

    //http://localhost:8080/api/UserWebController/getUsers
    @Secured({AUTHORITY_PREFIX + ADMIN})
    @RequestMapping(path = "/getUsers", method = RequestMethod.GET)
    public List<UserDTO> getUsers() throws CustomConverterException{
        List<UserDTO> userDTOS = userService.getUsers();
        if(userDTOS.isEmpty()||userDTOS==null){
            logger.info("end-point getUsers failed");
            throw new ResponseStatusException(
                    HttpStatus.INTERNAL_SERVER_ERROR, "Internal server error"
            );
        }
        userDTOS.forEach(u -> {
            u.setPassword(null);
        });
        logger.info("end-point getUsers succeed");
        return userDTOS;
    }

-----

L'accessibilité du end-point setUser est réservée aux utilisateurs disposant du ou des rôles USER, MANAGER et ADMIN.

    //http://localhost:8080/api/UserWebController/setUser
    @Secured({AUTHORITY_PREFIX + USER, AUTHORITY_PREFIX + MANAGER, AUTHORITY_PREFIX + ADMIN})
    @RequestMapping(path = "/setUser", method = RequestMethod.PUT)
    public UserDTO setUser(@RequestBody UserDTO userDTOEntry) {
            validateThisUser(userDTOEntry.getEmail());
        final UserDTO userDTO = userService.setUser(userDTOEntry);
        if(userDTO==null){
            logger.info("end-point setUser failed");
            throw new ResponseStatusException(
                    HttpStatus.INTERNAL_SERVER_ERROR, "Internal server error"
            );
        }
        logger.info("end-point setUser succeed");
        return userDTO;
    }

-----

Nous souhaitons ajouter une condition supplémentaire : un administrateur peut accéder à ce endpoint quelque soit l'utilisateur à modifier, à contrario les managers et users ne peuvent modifier qu'eux même.

Il est possible d'ajouter des contraintes supplémentaires en utilisant les informations de l'utilisateur enregistré dans le contexte de l'application.

Cette condition a été rajoutée dans la super class de tous les contrôlleurs.

    public class SuperController{

        private final static Logger logger = Logger.getLogger(SuperController.class);
        private static UserDetails userDetails;

        private void getSecurityContextHolder() {
            UsernamePasswordAuthenticationToken authentication = (UsernamePasswordAuthenticationToken) SecurityContextHolder.getContext().getAuthentication();
            userDetails = (UserDetails) authentication.getPrincipal();
        }

        /**
        * for each @Secure ROLE_USER && ROLE_MANAGER only themself can access
        * for all @Secure ROLE_ADMIN can access
        */
        public void validateThisUser(String emailEntry) {
            try {
                getSecurityContextHolder();
                boolean authorization = false;
                if(userDetails.getAuthorities().contains(new SimpleGrantedAuthority("ROLE_ADMIN"))){
                    authorization = true;
                }else {
                    if (userDetails.getUsername().equals(emailEntry)) {
                        authorization = true;
                    }
                }
                    if(authorization==false){
                        throw new ResponseStatusException(
                                HttpStatus.FORBIDDEN, "Forbidden"
                        );
                    }
            } catch (NullPointerException ex) {
                throw new ResponseStatusException(
                        HttpStatus.FORBIDDEN, "Forbidden"
                );
            }
        }

    }

-----

### Back-end : tests de end-point ###

### Connection ###

Dans le cas ou l'utilisateur résussit à s'authentifier, la méthode va retourner le token de session Jwt.

    public class AuthProvider implements ITools {

        private final static Logger logger = Logger.getLogger(AuthProvider.class);
        @Autowired
        private UserRepository userRepository;
        @Autowired
        private RoleRepository roleRepository;
        @Autowired
        SingletonBean singletonBean;

        public Token validateConnection(Credential credential) {
            String credentialSha3 = getStringSha3(credential.getPassword());
            User user = userRepository.selectMyUserByEmail(credential.getEmail());
            if (user == null) {
                return null;
            } else
            if(credentialSha3.equals(user.getPassword())) {
                try {
                    String jwt = generateJwt(credential.getEmail());
                    Token token = new Token();
                    token.setToken(jwt);
                    logger.info("Method validateConnection succeed");
                    return token;
                } catch (CustomJoseException ex) {
                    logger.error(ex.getMessage());
                    return null;
                } catch(CustomTokenException ex){
                    logger.error(ex.getMessage());
                    return null;
                }
            }
            return null;
        }

        private String generateJwt(String email){
            try {
                List<Role> roles = roleRepository.findByUsersEmail(email);
                if(roles.isEmpty()||roles==null){
                    throw new CustomTokenException("token must contain at least 1 role");
                }
                List<String> rolesString = roles.stream().map(
                        role->{
                            return role.getName();
                        }).collect(Collectors.toCollection(ArrayList::new));
                int kidRandom = generateRandmoKid();
                RsaJsonWebKey rsaJsonWebKey = (RsaJsonWebKey) singletonBean.getJsonWebKeys().get(kidRandom);
                /**
                * Create the Claims, which will be the content of the jwt
                */
                JwtClaims jwtClaims = new JwtClaims();
                jwtClaims.setIssuer(DOMAIN);
                jwtClaims.setExpirationTimeMinutesInTheFuture(120);
                jwtClaims.setGeneratedJwtId();
                jwtClaims.setIssuedAtToNow();
                jwtClaims.setNotBeforeMinutesInThePast(2);
                jwtClaims.setSubject(email);
                jwtClaims.setStringListClaim(AUTHORITIES_KEY, rolesString);
                JsonWebSignature jsonWebSignature = new JsonWebSignature();
                jsonWebSignature.setPayload(jwtClaims.toJson());
                jsonWebSignature.setKeyIdHeaderValue(rsaJsonWebKey.getKeyId());
                jsonWebSignature.setKey(rsaJsonWebKey.getPrivateKey());
                jsonWebSignature.setAlgorithmHeaderValue(AlgorithmIdentifiers.RSA_USING_SHA256);
                return jsonWebSignature.getCompactSerialization();

            } catch (JoseException ex) {
                throw new CustomJoseException("Failed to generate token");
            }catch (NullPointerException ex){
                throw new CustomTokenException("user roles cannot be null");
            }
        }

    }

-----

<img src = "/img/012.png">

-----

<img src = "/img/014.png">

-----

### Jeu de tests ###

Identifiants à prendre :

* jean@jean.com (ADMIN)
* jeanne@jeanne.com (MANAGER)
* john@john.com (USER)
* johny@johny.com (USER)
* password 0000

Lancer le jeu d'essai

* http://localhost:8080/api/UserWebController/getDataTest

End-points à tester : 

* http://localhost:8080/api/UserWebController/getUserDto?email=jean@jean.com
    * Avec un role User pour lui même
    * Avec un role User pour quelqu'un d'autre
    * Avec un role Admin pour quelqu'un d'autre

* http://localhost:8080/api/UserWebController/getRoles
    * Avec un role manager
    * Avec un role admin
    * Avec un role admin en modifiant la signature du token
    * Avec un role admin en modifiant le payload du token

* http://localhost:8080/api/UserWebController/getRoleDtos?email=jean@jean.com
    * Avec un role user
    * Avec un token modifié : exemple prendre la signature d'un autre token mais disposant des caractéristiques identiques (rôles et kid)
    * Avec un token périmé (en réduisant le exp du token à 1 minute par exemple)

-----

Voici ce que devrais retourner le résultat du 1er test (une response contenant le DTO avec un status 200)

-----

<img src = "/img/015.png">

-----

### SpringBoot data rest - spécificité ###

Par défaut SpringSecurity laisse un accès libre aux end points générés automatiquement par SpringBoot JPA Rest.

Les entités de la base de données peuvent être lus par n'importe qui!

Les conditions d'accès à ces end-points sont définis dans la class SecurityConfig.

    @Configuration
    @EnableWebSecurity
    @EnableGlobalMethodSecurity(securedEnabled = true)
    public class SecurityConfig extends WebSecurityConfigurerAdapter {

        @Bean
        public JwtRequestFilter jwtRequestFilter() {
            return new JwtRequestFilter();
        }

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            /**
            * return status 403 if not allowed
            */
            http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and().
                    authorizeRequests()
                    .antMatchers(HttpMethod.GET, "/api", "/api/*").permitAll()
                    .antMatchers(HttpMethod.GET, "/roles", "/roles/*").hasRole(ADMIN)
                    .antMatchers(HttpMethod.GET, "/users", "/users/*").hasRole(ADMIN)
                    .antMatchers(HttpMethod.POST, "/api/*").permitAll()
                    .antMatchers(HttpMethod.PUT, "/api/*").permitAll()
                    .antMatchers(HttpMethod.DELETE, "/api/*").permitAll().and()
                    .requestCache().requestCache(new NullRequestCache()).and()
                    .cors().and()
                    .csrf().disable();

            /**
            * add custom JWT security filter,if not allowing by @Secure return status 403
            */
            http.addFilterBefore(jwtRequestFilter(), UsernamePasswordAuthenticationFilter.class);
        }
    }

-----

Les accès des end-points pour les entités Users et Roles sont valables uniquement pour les utilisateurs disposant du rôle ADMIN.

Pour l'entitée Space, aucune condition n'a été établiee (pour l'exemple).

Ainsi la requête http://localhost:8080/spaces sur un navigateur permet à n'importe qui de récupérer des informations contenus dans la base de données.

A tester :

* http://localhost:8080/spaces
    * Sans token

* http://localhost:8080/users
    * Sans token, avec un role user, puis avec un role admin

-----

## Front-end : Présentation du projet + implémentation ##

Le projet dispose de fonctions rudimentaires.

Les fonctions basiques d'angular ne seront pas présentées.

Nous nous intéressons ici qu'aux fonctions de sécurités.

* Se connecter
* S'inscrire
* Modifier son profil (USER, MANAGER et ADMIN)
* Allez sur sa page (MANAGER et ADMIN)
* Modifier les roles des utilisateur (ADMIN)

<img src = "/img/018.png">

-----

### Front-end : Interceptor ###

A chaque connexion, le token de session est enregistré en localStorage (mémoire du pc). Un interceptor va récupérer ce token et le renvoyer dans le header de chaques requêtes clientes.

    @Injectable({
    providedIn: 'root'
    })
    export class TokenInterceptorService implements HttpInterceptor{

    constructor(
        private router:Router,
        ) { }

        intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
        request = request.clone({
            setHeaders: {
            Authorization: `Bearer ${localStorage.getItem('token')}`
            }
        });
        return next.handle(request).pipe(
            catchError((error: HttpErrorResponse) => {
            if (error.error instanceof ErrorEvent) {
            } else {
            if (error.status === 403) { 
                this.router.navigate( ["/error"] );
            }
            if (error.status === 500) { 
                this.router.navigate( ["/error"] );
            }
            }
            return throwError(error);
        })
        );
        }
    }

-----

Ce même interceptor a pour mission de router vers la page d'error s'il détecte que le status de la response serveur est 403 ou 500.

-----

### Front-end : CanActivate ###

Angular permet aussi de gérer les rôles du token.

Il faut pour cela redéfinir la méthode canActivate.

Ici nous récupérons le token pour en extraire les roles utilisateur.
Vous observez que l'utilisateur est redirigé vers une page d'erreur si le role est null.

Il a également été rajoutée la fonction pour déterminer si le token est périmé.
Ainsi le client déconnecte automatiquement l'utilisateur et le route vers la page de re-authentification, et cela sans action du serveur d'application.

    @Injectable({
    providedIn: 'root'
    })
    export class AuthGuardService implements CanActivate{

    roles:string[];
    role:string;
    email:string;
    exp:string;

    constructor(
        private router:Router,
        private apiService:ApiService,
        private subscriptionService: SubscriptionService
        ) { }

    canActivate(
        route: ActivatedRouteSnapshot,
        state: RouterStateSnapshot): Observable<boolean> | Promise<boolean> | boolean {
        const expectedRole = route.data.expectedRole;
        this.role = null;
        if (localStorage.getItem('token')){

            this.getDecodedAccessToken(localStorage.getItem('token'));
            if(expectedRole!=null){
                this.roles.forEach(element => {
                if(element===expectedRole){
                    this.role = element;
                }
                });
                if(this.role==null){
                    this.router.navigate( ["/error"] );
                    return false
                }
            }
            var current_time = new Date().getTime() / 1000;
            if(current_time > parseInt(this.exp)){
                // Destroy local session; redirect to /login
                localStorage.removeItem('token');
                localStorage.removeItem('email');
                this.subscriptionService.emitTokenSubject();
                alert("Your session has expired, please provide Login!")
                this.router.navigate( ["/login"] );
            }
            return true;
        }
            else  {
            this.subscriptionService.emitTokenSubject();
            alert("You are currently not logged in, please provide Login!")
            this.router.navigate( ["/login"] );
            return false
        }
    }

    validateConnection(credential : Credential){
    this.apiService.validateConnection(credential)
    .subscribe((resp: any) => {
        localStorage.setItem('token', resp.token);
        this.subscriptionService.emitTokenSubject();
        this.getDecodedAccessToken(resp.token);
        this.router.navigate(['/space']);
    }, err => {
        console.log(err);
        alert(err.message);
    })
    }

    getDecodedAccessToken(token: string): void {
    try{
        let jwtData = token.split('.')[1];
        let decodedJwtJsonData = window.atob(jwtData)
        let decodedJwtData = JSON.parse(decodedJwtJsonData)
        let roles = decodedJwtData.roles+ '';
        this.exp = decodedJwtData.exp;
        this.roles= roles.split(",");
        this.email=decodedJwtData.sub+'';
        localStorage.setItem('email', this.email);
    }
    catch(Error){
        console.log(Error);
    }
    }
    }

-----    

Enfin c'est le module app-routing.module.ts qui définis les pages accessible en fonctions des roles, avec une redirection vers un page d'erreur si nécessaire.

    const routes: Routes = [
    {
        path:'',
        component:HomeComponent
    },
    {
        path:'home', 
        component:HomeComponent
    },
    {
        path:'user', 
        component:SpaceComponent,
        canActivate: [AuthGuardService],
        data:{
        expectedRole:'USER'
        }   
    },
    {
        path:'manager', 
        component:ManagerComponent,
        canActivate: [AuthGuardService],
        data:{
        expectedRole:'MANAGER'
        }   
    },
    {
        path:'admin', 
        component:AdminComponent,
        canActivate: [AuthGuardService],
        data:{
        expectedRole:'ADMIN'
        }
    },
    {
        path:'login',
        component:LoginComponent
    },
    {
        path:'register',
        component:RegisterComponent
    },
    {
        path:'error',
        component:ErrorComponent
    },
    { path: '**', redirectTo: '' }
    ];

    @NgModule({
    imports: [RouterModule.forRoot(routes)],
    exports: [RouterModule]
    })
    export class AppRoutingModule { }

-----

### Front-end : tests de fonctionalités ###

Comme pour le test des end-points, je vous propose de vous connecter/déconnecter avec des utilisateurs disposant d'un role différents (USER/MANAGER/ADMIN), et de tester les différentes pages du site.

Vous pourrez observer les redirections réalisées par Angular en fonction des conditions définis (interception du status de la response du serveur, expiration du token, rôle non approprié, utilisateur non connecté, ...).

<img src="/img/021.png">

-----

Cette rapide initiation est terminée.

Il s'agit d'une première version qui sera améliorée.

Pour tous commentaires et remarques, vous pouvez me contacter par mail : phou.jeannory@gmail.com
ou par message : [linkedin](https://www.linkedin.com/in/jeannory-phou-69380715a/)