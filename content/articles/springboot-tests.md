---
title: "Springboot-tests"
date: 2019-10-25T15:02:42+02:00
draft: false
author: "Jeannory"
tags: ["articles"]
categories: ["2/Springboot-tests"]
---
## Introduction ##

Springboot-tests : Type de tests avec Springboot

* Tests de persistence - DataJpaTest 
* Tests unitaires - Mockito
* Tests de controleurs - In-Container

---

Fonctionalitées du projet :

* Enregistrement/Modification utilisateur
* Authentification
* Gestion des rôles
* implémentation de Spring Security/Jwt.

Présentation de l'implémentation de Spring Security/Jwt : [ici](/blog/articles/spring-security/)

---

Stack technique 

* Java 8
* Springboot 2
* Spring Security
* Jwt
* Postgresql
* Junit 4
* Mockito 1.9
* H2

Dépôt git du projet back-end : [ici](https://github.com/jeannory/SpringSecurityExampleBE)

## Tests de persistence - DataJpaTest ##

### Test des repositories avec bdd embarquée H2 ###

Etapes :

* Tester pour comprendre le type de retour
* Tester les méthodes génériques de JpaRepository
* Créer et tester les requêtes personnalisées (par noms de méthodes)
* Créer et tester les nativesQuery

---

Avantages :

* Aider au développements des services
* Tester les requetes personnalisées sans besoin d'interface
* Tester les nativesQuery sans besoin d'interface

---

Dépendance à ajouter

        <!-- https://mvnrepository.com/artifact/com.h2database/h2 -->
                <dependency>
                    <groupId>com.h2database</groupId>
                    <artifactId>h2</artifactId>
                    <version>1.4.200</version>
                    <scope>test</scope>
                </dependency>

---

Repository à tester

    @CrossOrigin("*")
    @Repository
    public interface UserRepository extends JpaRepository<User, Long> {
        User findByEmail(String email);
        @Query(value = "select * from cuisine_user u where u.email ilike :paramEmail", nativeQuery=true)
        User selectMyUserByEmail(@Param("paramEmail") String email);
    }

---

jeu de test

* test_findById_when_user_exist_should_return_user
* test_findById_when_user_no_exist_should_return_optional_empty
* test_findAll_when_users_exist_should_return_list
* test_findAll_when_users_no_exist_should_return_emptyList
* test_findByEmail_when_all_parameters_valid_should_return_result
* test_findByEmail_when_user_no_have_roles_should_return_user
* test_findByEmail_when_no_user_found_should_return_null
* test_selectMyUserByEmail_when_all_parameters_valid_should_return_result
* test_selectMyUserByEmail_when_user_no_have_roles_should_return_user
* test_selectMyUserByEmail_when_no_user_found_should_return_null

Réinitialisation non automatique de la bdd après chaque test (réinitialisation des données mais pas de l'incrémentation, utilisation de @DirtiesContext si besoin).

---

    @RunWith(SpringRunner.class)
    @DataJpaTest
    public class UserRepositoryTest {

        @Autowired
        private UserRepository userRepository;
        @Autowired
        private RoleRepository roleRepository;

        @Before
        public void setUp() throws Exception {
        }

        @DirtiesContext(methodMode = DirtiesContext.MethodMode.BEFORE_METHOD)
        @Test
        public void test_findById_when_user_exist_should_return_user(){
            //given
            final User user = BuilderUtils1.buildUser("jean@jean.com", "1234", Gender.Monsieur, "Jean", "Leroy",
                    "0101010101", "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE);
            userRepository.save(user);

            //when
            final Optional<User> result = userRepository.findById(1L);

            //then
            //incrementing start at 1
            Assert.assertTrue(1L == result.get().getId());
            Assert.assertEquals("jean@jean.com", result.get().getEmail());
        }

        @DirtiesContext(methodMode = DirtiesContext.MethodMode.BEFORE_METHOD)
        @Test
        public void test_findById_when_user_no_exist_should_return_optional_empty(){
            //given & when
            final Optional<User> result = userRepository.findById(1L);

            //then
            Assert.assertEquals(Optional.empty(), result);
        }

        @Test
        public void test_findAll_when_users_exist_should_return_list(){
            //given
            final User user1 = BuilderUtils1.buildUser("1-jean@gmail.com", "1234", Gender.Monsieur, "Jean", "Leroy",
                    "0101010101", "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE);
            final User user2 = BuilderUtils1.buildUser("2-jean@gmail.com", "1234", Gender.Monsieur, "Jean", "Leroy",
                    "0101010101", "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE);
            final User user3 = BuilderUtils1.buildUser("3-jean@gmail.com", "1234", Gender.Monsieur, "Jean", "Leroy",
                    "0101010101", "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE);
            userRepository.save(user1);
            userRepository.save(user2);
            userRepository.save(user3);

            //when
            final List<User> results = userRepository.findAll();

            //then
            Assert.assertEquals(3,results.size());
            /**
            * all emails have been founded
            */
            AtomicInteger emailsFound = new AtomicInteger();
            results.forEach(
                    user->{
                        if(user.getEmail().equals("1-jean@gmail.com")){
                            emailsFound.addAndGet(1);
                        }
                        else if(user.getEmail().equals("2-jean@gmail.com")){
                            emailsFound.addAndGet(1);
                        }
                        else if(user.getEmail().equals("3-jean@gmail.com")){
                            emailsFound.addAndGet(1);
                        }
                    }
            );
            Assert.assertEquals(3, emailsFound.get());
        }

        @Test
        public void test_findAll_when_users_no_exist_should_return_emptyList(){
            //given & when
            final List<User> results = userRepository.findAll();

            //then
            Assert.assertEquals(0, results.size());
            Assert.assertEquals(Collections.emptyList(), results);
        }

        @Test
        public void test_findByEmail_when_all_parameters_valid_should_return_result() {
            //given
            final Role role1 = new Role();
            role1.setName("USER");
            final Role role2 = new Role();
            role2.setName("MANAGER");
            final Role role3 = new Role();
            role3.setName("ADMIN");
            final Set<Role> roles  = new HashSet<>(Arrays.asList(role1, role2, role3));
            roles.forEach(role -> {
                roleRepository.save(role);
            });
            final User user = BuilderUtils1.buildUser("jean@jean.com", "1234", Gender.Monsieur, "Jean", "Leroy",
                    "0101010101", "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE);
            user.setRoles(roles);
            userRepository.save(user);

            //when
            final User result = userRepository.findByEmail("jean@jean.com");

            //then
            Assert.assertEquals("jean@jean.com", result.getEmail());
            Assert.assertEquals(3, result.getRoles().size());
        }

        @Test
        public void test_findByEmail_when_user_no_have_roles_should_return_user() {
            //given
            final User user = BuilderUtils1.buildUser("jean@jean.com", "1234", Gender.Monsieur, "Jean", "Leroy",
                    "0101010101", "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE);
            userRepository.save(user);

            //when
            final User result = userRepository.findByEmail("jean@jean.com");

            //then
            Assert.assertEquals("jean@jean.com", result.getEmail());
            Assert.assertEquals(null, result.getRoles());
        }

        @Test
        public void test_findByEmail_when_no_user_found_should_return_null() {
            //given && when
            final User result = userRepository.findByEmail("jean@jean.com");

            //then
            Assert.assertNull("return null", result);
        }

        @Test
        public void test_selectMyUserByEmail_when_all_parameters_valid_should_return_result() {
            //given
            final Role role1 = new Role();
            role1.setName("USER");
            final Role role2 = new Role();
            role2.setName("MANAGER");
            final Role role3 = new Role();
            role3.setName("ADMIN");
            final Set<Role> roles  = new HashSet<>(Arrays.asList(role1, role2, role3));
            roles.forEach(role -> {
                roleRepository.save(role);
            });
            final User user = BuilderUtils1.buildUser("jean@jean.com", "1234", Gender.Monsieur, "Jean", "Leroy", "0101010101",
                    "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE);
            user.setRoles(roles);
            userRepository.save(user);

            //when
            final User result = userRepository.selectMyUserByEmail("jean@jean.com");

            //then
            Assert.assertEquals("jean@jean.com", result.getEmail());
            Assert.assertEquals(3, result.getRoles().size());
        }

        @Test
        public void test_selectMyUserByEmail_when_user_no_have_roles_should_return_user() {
            //given
            final User user = BuilderUtils1.buildUser("jean@jean.com", "1234", Gender.Monsieur, "Jean", "Leroy", "0101010101",
                    "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE);
            userRepository.save(user);

            //when
            final User result = userRepository.selectMyUserByEmail("jean@jean.com");

            //then
            Assert.assertEquals("jean@jean.com", result.getEmail());
            Assert.assertEquals(null, result.getRoles());
        }

        @Test
        public void test_selectMyUserByEmail_when__no_user_found_should_return_null() {
            //given && when
            final User result = userRepository.selectMyUserByEmail("jean@jean.com");

            //then
            Assert.assertNull("return null", result);
        }
    }

---

## Tests unitaires - Mockito ##

Principes :

* Jeu de tests la plus exhaustive possible
* Créer des exceptions personnalisées en fonction du jeu de tests
* Disposer d'un taux de couverture de 100 % sur les méthodes testés
* Utiliser Mockito

### Test d'une méthode simple ###

Class SuperModelMapper, Méthode générique simple à tester (conversion d'un dto en entité) :

    public Optional<E> convertToEntity(final D dto) {
        logger.info("Method convertToEntity");
        if (dto == null) {
            return Optional.empty();
        }
        E entity = singletonBean.getModelMapper().map(dto, (Type) dto.getEntityClass());
        if(dto.getClass().equals(UserDTO.class)){
            entity = convertToUser(entity);
        }
        return Optional.of(entity);
    }

    //hash password for persistence or authentication
    private E convertToUser(final E entity){
        logger.info("Method convertToUser");
        final User user = (User) entity;
        user.setPassword(getStringSha3(user.getPassword()));
        return (E) user;
    }

---

Déclarer la class à tester et ses dépendances

---

    public class SuperModelMapperTest1 {

        private SuperModelMapper superModelMapper;
        private SingletonBean singletonBean;

        @Before
        public void setUp() throws Exception {
            this.superModelMapper = Mockito.spy(new SuperModelMapper());
            singletonBean = Mockito.mock(SingletonBean.class);
            Whitebox.setInternalState(superModelMapper, "singletonBean", singletonBean);
        }

---

Jeu de tests ou tous les DTOS sont testés

* test_convertToEntity_when_dto_is_RoleDTO_should_return_Role
* test_convertToEntity_when_RoleDTO_is_null_should_return_optional_empty
* test_convertToEntity_when_dto_is_SpaceDTO_should_return_Space
* test_convertToEntity_when_SpaceDTO_is_null_should_return_optional_empty
* test_convertToEntity_when_dto_is_UserDTO_should_return_User_with_password_is_null
* test_convertToEntity_when_UserDTO_is_null_should_return_optional_empty

---

        @Test
        public void test_convertToEntity_when_dto_is_RoleDTO_should_return_Role(){
            //given
            final RoleDTO roleDTO = new RoleDTO();
            roleDTO.setId(1L);
            roleDTO.setName("ADMIN");
            Mockito.when(singletonBean.getModelMapper()).thenReturn(new ModelMapper());

            //when
            final Optional<Role> result = superModelMapper.convertToEntity(roleDTO);

            //then
            Assert.assertEquals("ADMIN", result.get().getName());
            Assert.assertEquals(Role.class, result.get().getClass());
        }

        @Test
        public void test_convertToEntity_when_RoleDTO_is_null_should_return_optional_empty(){
            //given
            final RoleDTO roleDTO = null;
            Mockito.when(singletonBean.getModelMapper()).thenReturn(new ModelMapper());

            //when
            final Optional<Role> result = superModelMapper.convertToEntity(roleDTO);

            //then
            Assert.assertEquals(Optional.empty(), result);
        }

        @Test
        public void test_convertToEntity_when_dto_is_SpaceDTO_should_return_Space(){
            //given
            final SpaceDTO spaceDTO = new SpaceDTO();
            spaceDTO.setName("Espace de Jean");
            spaceDTO.setUserEmail("jean@gmail.com");
            Mockito.when(singletonBean.getModelMapper()).thenReturn(new ModelMapper());

            //when
            final Optional<Space> result = superModelMapper.convertToEntity(spaceDTO);

            //then
            Assert.assertEquals("Espace de Jean", result.get().getName());
            Assert.assertEquals("jean@gmail.com", result.get().getUser().getEmail());
            Assert.assertEquals(Space.class, result.get().getClass());

        }

        @Test
        public void test_convertToEntity_when_SpaceDTO_is_null_should_return_optional_empty(){
            //given
            final SpaceDTO spaceDTO = null;
            Mockito.when(singletonBean.getModelMapper()).thenReturn(new ModelMapper());

            //when
            final Optional<Space> result = superModelMapper.convertToEntity(spaceDTO);

            //then
            Assert.assertEquals(Optional.empty(), result);
        }

        @Test
        public void test_convertToEntity_when_dto_is_UserDTO_should_return_User_with_password_crypted(){
            //given
            final UserDTO userDTO = BuilderUtils1.buildUserDTO(
                    1L, "jean@gmail.com", "1234", Gender.Monsieur, "Jean", "Leroy",
                    "0101010101", "9 rue du roi", "75018", "Paris", "9ème étage",
                    null, "ADMIN, COOKER, USER", Status.ACTIVE);
            Mockito.when(singletonBean.getModelMapper()).thenReturn(new ModelMapper());

            //when
            final Optional<User> result = superModelMapper.convertToEntity(userDTO);

            //then
            Assert.assertEquals("jean@gmail.com", result.get().getEmail());
            Assert.assertEquals(User.class, result.get().getClass());
            final String sha3Password = getStringSha3("1234");
            Assert.assertEquals(sha3Password, result.get().getPassword());
            Assert.assertNull("roles are null", result.get().getRoles());
        }

        @Test
        public void test_convertToEntity_when_UserDTO_is_null_should_return_optional_empty(){
            //given
            final UserDTO userDTO = null;
            Mockito.when(singletonBean.getModelMapper()).thenReturn(new ModelMapper());

            //when
            final Optional<User> result = superModelMapper.convertToEntity(userDTO);

            //then
            Assert.assertEquals(Optional.empty(), result);
        }


---

### Test d'une méthode complexe AuthProvider.validateConnection ###

Comporte plusieurs dépendances


    @Component
    public class AuthProvider implements ITools {

        private final static Logger logger = Logger.getLogger(AuthProvider.class);
        @Autowired
        private UserRepository userRepository;
        @Autowired
        private RoleRepository roleRepository;
        @Autowired
        SingletonBean singletonBean;

        public Token validateConnection(Credential credential) {
            logger.info("Method validateConnection");
            try {
                final User user = validateUser(credential.getEmail());
                if (credential.getSha3Password().equals(user.getPassword())) {
                    try {
                        final String jwt = generateJwt(credential.getEmail());
                        final Token token = new Token();
                        token.setToken(jwt);
                        return token;
                    } catch (CustomJoseException ex) {
                        logger.error(ex.getMessage());
                        return null;
                    }
                }
            } catch (CustomTokenException ex) {
                logger.error(ex.getMessage());
                return null;
            }
            return null;
        }

        private User validateUser(String email) {
            final User user = userRepository.selectMyUserByEmail(email);
            if (user == null || email.isEmpty() || !isValidEmail(email)) {
                throw new CustomTokenException("email not valid or user not found");
            }
            return user;
        }

        private String generateJwt(String email) {
            logger.info("Method generateJwt");
            try {
                List<Role> roles = roleRepository.findByUsersEmail(email);
                if (roles.isEmpty()) {
                    throw new CustomTokenException("token must contain at least 1 role");
                }
                List<String> rolesString = roles.stream().map(
                        role -> {
                            return role.getName();
                        }).collect(Collectors.toCollection(ArrayList::new));
                final int kidRandom = generateRandmoKid();
                final RsaJsonWebKey rsaJsonWebKey = (RsaJsonWebKey) singletonBean.getJsonWebKeys().get(kidRandom);
                /**
                * Create the Claims, which will be the content of the jwt
                */
                final JwtClaims jwtClaims = new JwtClaims();
                jwtClaims.setIssuer(DOMAIN);
                /**
                *UI request for refresh token 30 min before its expiration
                *setExpirationTimeMinutesInTheFuture to 29 UI will always request for a new token
                *setExpirationTimeMinutesInTheFuture to 120 UI will request for a new token after 90 min
                */
                jwtClaims.setExpirationTimeMinutesInTheFuture(120);
                //jwtClaims.setExpirationTimeMinutesInTheFuture(29);
                jwtClaims.setGeneratedJwtId();
                jwtClaims.setIssuedAtToNow();
                jwtClaims.setNotBeforeMinutesInThePast(2);
                jwtClaims.setSubject(email);
                jwtClaims.setStringListClaim(AUTHORITIES_KEY, rolesString);
                final JsonWebSignature jsonWebSignature = new JsonWebSignature();
                jsonWebSignature.setPayload(jwtClaims.toJson());
                jsonWebSignature.setKeyIdHeaderValue(rsaJsonWebKey.getKeyId());
                jsonWebSignature.setKey(rsaJsonWebKey.getPrivateKey());
                jsonWebSignature.setAlgorithmHeaderValue(AlgorithmIdentifiers.RSA_USING_SHA256);
                return jsonWebSignature.getCompactSerialization();
            } catch (JoseException ex) {
                throw new CustomJoseException("CustomJoseException - Failed to generate token");
            } catch (IndexOutOfBoundsException ex) {
                throw new CustomJoseException("singletonBean.jsonWebKeys is null or empty");
            }
        }
    }

---

Ecriture de tests

Des Exceptions ont été rajoutés dans la méthode lors de l'écriture des tests

* test_validateConnection_when_all_parameters_valid
* test_validateConnection_when_credential_not_matching_should_return_null
* test_validateConnection_when_user_not_found_then_throw_CustomTokenException_and_return_null
* test_validateConnection_when_email_is_empty_then_throw_CustomTokenException_and_return_null
* test_validateConnection_when_email_is_null_then_throw_CustomTokenException_and_return_null
* test_validateConnection_when_email_is_null_then_throw_CustomTokenException_and_return_null_bis
* test_validateConnection_when_email_is_not_valid_email_then_throw_CustomTokenException_and_return_null
* test_validateConnection_when_user_Roles_is_empty_then_throw_CustomTokenException_and_return_null
* test_validateConnection_when_jsonWebKeys_isEmpty_throw_CustomJoseException_should_return_null
* test_validateConnection_when_throw_JoseException_and_return_null

---

    public class AuthProviderTest2 implements ITools {

        private AuthProvider authProvider;
        private UserRepository userRepository;
        private RoleRepository roleRepository;
        private SingletonBean singletonBean;

        @Before
        public void setUp() throws Exception {
            this.authProvider = new AuthProvider();
            userRepository = Mockito.mock(UserRepository.class);
            roleRepository = Mockito.mock(RoleRepository.class);
            singletonBean = Mockito.mock(SingletonBean.class);
            Whitebox.setInternalState(authProvider, "userRepository", userRepository);
            Whitebox.setInternalState(authProvider, "roleRepository", roleRepository);
            Whitebox.setInternalState(authProvider, "singletonBean", singletonBean);
        }

        @Test
        public void test_validateConnection_when_all_parameters_valid() throws JoseException {
            //given
            final Credential credential = Mockito.spy(new Credential());
            credential.setEmail("jean@jean.com");
            credential.setPassword("1234");
            final String hashPassword = getStringSha3("1234");
            final User user = BuilderUtils1.buildUser(1L, "jean@jean.com", hashPassword, Gender.Monsieur, "Jean", "Leroy", "0101010101",
                    "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE, Collections.singletonList(Arrays.asList("1", "USER")));
            Mockito.when(userRepository.selectMyUserByEmail(Mockito.eq(credential.getEmail()))).thenReturn(user);
            final List<Role> roles = Arrays.asList(
                    BuilderUtils1.buildRole(Arrays.asList("1", "USER")),
                    BuilderUtils1.buildRole(Arrays.asList("2", "MANAGER")),
                    BuilderUtils1.buildRole(Arrays.asList("3", "ADMIN")));
            Mockito.when(roleRepository.findByUsersEmail(Mockito.anyString())).thenReturn(roles);
            final List<JsonWebKey> jsonWebKeys = Arrays.asList(
                    BuilderUtils1.buildJsonWebKey(0),
                    BuilderUtils1.buildJsonWebKey(1),
                    BuilderUtils1.buildJsonWebKey(2)
            );
            Mockito.when(singletonBean.getJsonWebKeys()).thenReturn(jsonWebKeys);

            //when
            final Token result = authProvider.validateConnection(credential);

            //then
            Assert.assertNotNull("return not null", result.getToken());
            final String subject = BuilderUtils1.getStringFromJwtNode(result.getToken(), 1, "sub");
            Assert.assertEquals("jean@jean.com", subject);
            /**
            * the token has a new expiration which is always after now
            */
            final String expiration = BuilderUtils1.getStringFromJwtNode(result.getToken(), 1, "exp");
            final NumericDate jwtNumericDate = NumericDate.fromSeconds(Long.valueOf(expiration));
            final NumericDate numericDateNow = NumericDate.now();
            Assert.assertTrue(jwtNumericDate.isOnOrAfter(numericDateNow));
        }

        @Test
        public void test_validateConnection_when_credential_not_matching_should_return_null() throws JoseException {
            //given
            final Credential credential = new Credential();
            credential.setEmail("jean@jean.com");
            credential.setPassword("1234");
            final String hashPassword = getStringSha3(credential.getPassword());
            final User user = BuilderUtils1.buildUser(1L, "jean@jean.com", "anotherPassword", Gender.Monsieur, "Jean", "Leroy", "0101010101",
                    "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE, Collections.singletonList(Arrays.asList("1", "USER")));
            Mockito.when(userRepository.selectMyUserByEmail(Mockito.eq(credential.getEmail()))).thenReturn(user);

            //when
            final Token result = authProvider.validateConnection(credential);

            //then
            Assert.assertNull("return null", result);
        }

        @Test
        public void test_validateConnection_when_user_not_found_then_throw_CustomTokenException_and_return_null() throws JoseException {
            //given
            final Credential credential = new Credential();
            credential.setEmail("jean@jean.com");
            credential.setPassword("1234");
            final User user = null;
            Mockito.when(userRepository.selectMyUserByEmail(Mockito.eq(credential.getEmail()))).thenReturn(user);

            //when
            final Token result = authProvider.validateConnection(credential);

            //then
            Assert.assertNull("return null", result);
        }

        @Test
        public void test_validateConnection_when_email_is_empty_then_throw_CustomTokenException_and_return_null() throws JoseException {
            //given
            final Credential credential = new Credential();
            credential.setEmail("");
            credential.setPassword("1234");
            final User user = null;
            Mockito.when(userRepository.selectMyUserByEmail(Mockito.eq(credential.getEmail()))).thenReturn(user);

            //when
            final Token result = authProvider.validateConnection(credential);

            //then
            Assert.assertNull("return null", result);
        }

        @Test
        public void test_validateConnection_when_email_is_null_then_throw_CustomTokenException_and_return_null() throws JoseException {
            //given
            final Credential credential = new Credential();
            credential.setEmail(null);
            credential.setPassword("1234");
            final User user = null;
            Mockito.when(userRepository.selectMyUserByEmail(Mockito.eq(credential.getEmail()))).thenReturn(user);

            //when
            final Token result = authProvider.validateConnection(credential);

            //then
            Assert.assertNull("return null", result);
        }

        @Test
        public void test_validateConnection_when_email_is_null_then_throw_CustomTokenException_and_return_null_bis() throws JoseException {
            //given
            final Credential credential = new Credential();
            credential.setPassword("1234");
            final User user = null;
            Mockito.when(userRepository.selectMyUserByEmail(Mockito.eq(credential.getEmail()))).thenReturn(user);

            //when
            final Token result = authProvider.validateConnection(credential);

            //then
            Assert.assertNull("return null", result);
        }

        @Test
        public void test_validateConnection_when_email_is_not_valid_email_then_throw_CustomTokenException_and_return_null() throws JoseException {
            //given
            final Credential credential = new Credential();
            credential.setEmail("not an email");
            credential.setPassword("1234");
            final User user = null;
            Mockito.when(userRepository.selectMyUserByEmail(Mockito.eq(credential.getEmail()))).thenReturn(user);

            //when
            final Token result = authProvider.validateConnection(credential);

            //then
            Assert.assertNull("return null", result);
        }

        @Test
        public void test_validateConnection_when_user_Roles_is_empty_then_throw_CustomTokenException_and_return_null() throws JoseException {
            //given
            final Credential credential = new Credential();
            credential.setEmail("jean@jean.com");
            credential.setPassword("1234");
            final String hashPassword = getStringSha3(credential.getPassword());
            final User user = BuilderUtils1.buildUser(1L, "jean@jean.com", hashPassword, Gender.Monsieur, "Jean", "Leroy", "0101010101",
                    "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE, Collections.singletonList(Arrays.asList("1", "USER")));
            Mockito.when(userRepository.selectMyUserByEmail(Mockito.eq(credential.getEmail()))).thenReturn(user);
            Mockito.when(roleRepository.findByUsersEmail("jean@jean.com")).thenReturn(Collections.emptyList());

            //when
            final Token result = authProvider.validateConnection(credential);

            //then
            Assert.assertNull("return null", result);
        }

        @Test
        public void test_validateConnection_when_jsonWebKeys_isEmpty_throw_CustomJoseException_should_return_null() throws JoseException {
            //given
            final Credential credential = Mockito.spy(new Credential());
            credential.setEmail("jean@jean.com");
            credential.setPassword("1234");
            final String hashPassword = getStringSha3(credential.getPassword());
            final User user = BuilderUtils1.buildUser(1L, "jean@jean.com", hashPassword, Gender.Monsieur, "Jean", "Leroy", "0101010101",
                    "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE, Collections.singletonList(Arrays.asList("1", "USER")));
            Mockito.when(userRepository.selectMyUserByEmail(Mockito.eq(credential.getEmail()))).thenReturn(user);
            List<Role> roles = Arrays.asList(
                    BuilderUtils1.buildRole(Arrays.asList("1", "USER")),
                    BuilderUtils1.buildRole(Arrays.asList("2", "MANAGER")),
                    BuilderUtils1.buildRole(Arrays.asList("3", "ADMIN")));
            Mockito.when(roleRepository.findByUsersEmail(Mockito.anyString())).thenReturn(roles);
            Mockito.when(singletonBean.getJsonWebKeys()).thenReturn(Collections.emptyList());

            //when
            final Token result = authProvider.validateConnection(credential);

            //then
            Assert.assertNull("return null", result);
        }

---

Le test suivant comporte une exception importé JoseException.

Pour throw cet exception, il a été nécessaire de mocker les comportements de chacune des dépendances

---

    @Test
    public void test_validateConnection_when_throw_JoseException_and_return_null() throws JoseException{
        //given
        final Credential credential = Mockito.spy(new Credential());
        credential.setEmail("jean@jean.com");
        credential.setPassword("1234");
        final String hashPassword = getStringSha3("1234");
        final User user = BuilderUtils1.buildUser(1L, "jean@jean.com", hashPassword, Gender.Monsieur, "Jean", "Leroy", "0101010101",
                "9 rue du roi", "75018", "Paris", "9ème étage", Status.ACTIVE, Collections.singletonList(Arrays.asList("1", "USER")));
        Mockito.when(userRepository.selectMyUserByEmail(Mockito.eq(credential.getEmail()))).thenReturn(user);
        final List<Role> roles = Arrays.asList(
                BuilderUtils1.buildRole(Arrays.asList("1", "USER")),
                BuilderUtils1.buildRole(Arrays.asList("2", "MANAGER")),
                BuilderUtils1.buildRole(Arrays.asList("3", "ADMIN")));
        Mockito.when(roleRepository.findByUsersEmail(Mockito.anyString())).thenReturn(roles);
        final List<JsonWebKey> jsonWebKeys = Arrays.asList(
                BuilderUtils1.buildJsonWebKey(0),
                BuilderUtils1.buildJsonWebKey(1),
                BuilderUtils1.buildJsonWebKey(2)
        );
        Mockito.when(singletonBean.getJsonWebKeys()).thenReturn(jsonWebKeys);
        Mockito.when(authProvider.validateConnection(credential)).thenThrow(new CustomJoseException("CustomJoseException - Failed to generate token"));

        //when
        final Token result = authProvider.validateConnection(credential);

        //then
        Assert.assertNull("return null", result);
    }

---

## Tests de controleurs - Tests-In-Container ##

Les annotations permettent de faire des tests dans un container

* @RunWith(SpringRunner.class)
* @SpringBootTest
* @AutoConfigureMockMvc
* @ActiveProfiles("test")

@DirtiesContext permet de choisir comment réinitialiser le contexte Spring

Avant chaque test, JdbcTemplate intègre un jeu d'essai dans une embedded database.

---

    @RunWith(SpringRunner.class)
    @SpringBootTest
    @AutoConfigureMockMvc
    @ActiveProfiles("test")
    @DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_CLASS)
    public class UserWebControllerTest2 implements ITools {

        private static final String BASE_URL = PRE_PATH + USER_CONTROLLER;
        @Autowired
        private MockMvc mockMvc;
        @Autowired
        private BuilderUtils2 builderUtils2;
        @Autowired
        private UserRepository userRepository;
        @Autowired
        private JdbcTemplate jdbcTemplate;

        @Before
        public void setUp() throws Exception {
            final String sha3Password = getStringSha3("0000");
            jdbcTemplate.execute(
                    "delete from cuisine_users_roles; ");
            jdbcTemplate.execute(
                    "delete from cuisine_role; ");
            jdbcTemplate.execute(
                    "delete from cuisine_space; ");
            jdbcTemplate.execute(
                    "delete from cuisine_user; ");
            jdbcTemplate.execute(
                    "insert into cuisine_role(id, name) values (1001, 'USER'); " +
                            "insert into cuisine_role(id, name) values (1002, 'MANAGER'); " +
                            "insert into cuisine_role(id, name) values (1003, 'ADMIN'); "
            );
            jdbcTemplate.execute(
                    "insert into cuisine_user(id, email, password) values (1001, 'jean@jean.com', '" + sha3Password + "'); " +
                            "insert into cuisine_user(id, email, password) values (1002, 'johny@johny.com', '" + sha3Password + "'); " +
                            "insert into cuisine_user(id, email, password) values (1003, 'jeanne@jeanne.com', '" + sha3Password + "'); "
            );
            jdbcTemplate.execute(
                    "insert into cuisine_users_roles(cuisine_user_id, cuisine_role_id) values (1001,1001); " +
                            "insert into cuisine_users_roles(cuisine_user_id, cuisine_role_id) values (1001,1002); " +
                            "insert into cuisine_users_roles(cuisine_user_id, cuisine_role_id) values (1001,1003); " +
                            "insert into cuisine_users_roles(cuisine_user_id, cuisine_role_id) values (1002,1001); " +
                            "insert into cuisine_users_roles(cuisine_user_id, cuisine_role_id) values (1002,1002); " +
                            "insert into cuisine_users_roles(cuisine_user_id, cuisine_role_id) values (1003,1001); "
            );
        }



### Test de end-point - Test d'authentification ###

Demande classique d'authentification avec un identifiant et un mot de passe.

---

    //http://localhost:8080/api/UserWebController/validateConnection
    @RequestMapping(path = "/validateConnection", method = RequestMethod.POST)
    public Token validateConnection(@RequestBody final Credential credential) {
        logger.info("End point validateConnection");
        return validateToken(credential);
    }

---   

Jeu de tests :

* Requête ok : retourne Status 200 avec Token
* Requête ko : retourne Status Unauthorized 401
* Requête ko : retourne Status Not found 404

--- 

    @Test
    public void test_validateConnection_when_credential_valid_should_return_status_ok_with_Token() throws Exception {
        //given
        final Credential credential = new Credential();
        credential.setEmail("jean@jean.com");
        credential.setPassword("0000");

        //when && then
        final MvcResult mvcResult = invokeValidateConnection(credential)
                .andExpect(status().isOk())
                .andReturn();
        final Token result = builderUtils2.fromJsonResult(mvcResult, Token.class);
        System.out.println(result.getToken());
        Assert.assertFalse(result.getToken().isEmpty());
        final String sub = BuilderUtils1.getStringFromJwtNode(result.getToken(), 1, "sub");
        Assert.assertEquals("jean@jean.com", sub);
        final String roles = BuilderUtils1.getStringFromJwtNode(result.getToken(), 1, "roles");
        Assert.assertEquals("[\"USER\",\"MANAGER\",\"ADMIN\"]", roles);
    }

    @Test
    public void test_validateConnection_when_credential_not_valid_should_return_status_unauthorized_401() throws Exception {
        //given
        final Credential credential = new Credential();
        credential.setEmail("jean@jean.com");
        credential.setPassword("wrong password");

        //when && then
        final MvcResult mvcResult = invokeValidateConnection(credential)
                .andExpect(status().isUnauthorized())
                .andReturn();
    }

    @Test
    public void test_validateConnection_when_credential_email_no_exist_should_return_status_unauthorized_401() throws Exception {
        //given
        final Credential credential = new Credential();
        credential.setEmail("EmailNotOnDatabase@gmail.com");
        credential.setPassword("wrong password");

        //when && then
        final MvcResult mvcResult = invokeValidateConnection(credential)
                .andExpect(status().isUnauthorized())
                .andReturn();
    }

    @Test
    public void test_validateConnection_when_credential_email_is_empty_should_return_status_404() throws Exception {
        //given
        final Credential credential = new Credential();
        credential.setEmail("");
        credential.setPassword("1234");

        //when && then
        final MvcResult mvcResult = invokeValidateConnection(credential)
                .andExpect(status().isNotFound())
                .andReturn();
    }

    @Test
    public void test_validateConnection_when_credential_password_is_empty_should_return_status_404() throws Exception {
        //given
        final Credential credential = new Credential();
        credential.setEmail("jean@jean.com");
        credential.setPassword("");

        //when && then
        final MvcResult mvcResult = invokeValidateConnection(credential)
                .andExpect(status().isNotFound())
                .andReturn();
    }

    @Test
    public void test_validateConnection_when_credential_email_is_not_valid_email_should_return_status_404() throws Exception {
        //given
        final Credential credential = new Credential();
        credential.setEmail("this is not an email");
        credential.setPassword("1234");

        //when && then
        final MvcResult mvcResult = invokeValidateConnection(credential)
                .andExpect(status().isNotFound())
                .andReturn();
    }


---  

    private ResultActions invokeValidateConnection(final Credential credential) throws Exception {
        return mockMvc.perform(post(BASE_URL + "/validateConnection")
                .content(builderUtils2.asJsonString(credential))
                .accept(MediaType.APPLICATION_JSON)
                .contentType(MediaType.APPLICATION_JSON)
        );
    }

---

### Test de end-point - Authorisation via le jwt ###

Test d'un end-point dont l'accès est exclusivement réservé à un utilisateur avec le role ADMIN

---

    //http://localhost:8080/api/UserWebController/getRoles
    @Secured({AUTHORITY_PREFIX + ADMIN})
    @RequestMapping(path = "/getRoles", method = RequestMethod.GET)
    public List<RoleDTO> getRoles() {
        logger.info("End point getRoles");
        if(getRoles().isEmpty()){
            logger.error("Not found 404");
            throw new ResponseStatusException(
                    HttpStatus.NOT_FOUND, "Not found 404"
            );
        }
        return roleService.getAdminRoleDTOS();
    }

---

* Requête ok : retourne Status 200 avec List<RoleDTO>
* Requête ko : retourne Status Unauthorized 403
* Requête ko : retourne Status Not found 404
    * dernier cas non testé car difficulté pour simulter une erreur de conversion Role to RoleDTO à ce stade

---

    @Test
    public void test_getRoles_when_has_role_ADMIN_should_return_status_ok_with_roles() throws Exception {
        //given
        final Credential credential = new Credential();
        credential.setEmail("jean@jean.com");
        credential.setPassword("0000");

        //when && then
        final MvcResult mvcResult = invokeGetRoles(credential)
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(3)))
                .andExpect(jsonPath("$[0].name", is("USER")))
                .andExpect(jsonPath("$[1].name", is("MANAGER")))
                .andExpect(jsonPath("$[2].name", is("ADMIN")))
                .andReturn();
        final RoleDTO[] roles = builderUtils2.fromJsonResult(mvcResult, RoleDTO[].class);
        Assert.assertEquals("USER", roles[0].getName());
        Assert.assertEquals("MANAGER", roles[1].getName());
        Assert.assertEquals("ADMIN", roles[2].getName());
    }

    @Test
    public void test_getRoles_when_has_role_MANAGER_should_return_status_forbidden() throws Exception {
        //given
        final Credential credential = new Credential();
        credential.setEmail("johny@johny.com");
        credential.setPassword("0000");

        //when && then
        final MvcResult mvcResult = invokeGetRoles(credential)
                .andExpect(status().isForbidden())
                .andReturn();
    }

    @Test
    public void test_getRoles_when_has_role_USER_should_return_status_forbidden() throws Exception {
        //given
        final Credential credential = new Credential();
        credential.setEmail("jeanne@jeanne.com");
        credential.setPassword("0000");

        //when && then
        final MvcResult mvcResult = invokeGetRoles(credential)
                .andExpect(status().isForbidden())
                .andReturn();
    }

    @Test
    public void test_getRoles_when_no_token_on_header_should_return_forbidden() throws Exception {
        //given
        final ResultActions resultActions = mockMvc.perform(get(BASE_URL + "/getRoles")
                .accept(MediaType.APPLICATION_JSON));

        //when && then
        final MvcResult mvcResult = resultActions
                .andExpect(status().isForbidden())
                .andReturn();
    }

---

    private ResultActions invokeGetRoles(final Credential credential) throws Exception {
        final String token = getAccessToken(credential);
        return mockMvc.perform(get(BASE_URL + "/getRoles")
                .header("Authorization", "Bearer " + token)
                .accept(MediaType.APPLICATION_JSON));
    }

    private String getAccessToken(final Credential credential) throws Exception {
        final MvcResult mvcResult = invokeValidateConnection(credential)
                .andReturn();
        final Token token = builderUtils2.fromJsonResult(mvcResult, Token.class);
        return token.getToken();
    }

    private ResultActions invokeValidateConnection(final Credential credential) throws Exception {
        return mockMvc.perform(post(BASE_URL + "/validateConnection")
            .content(builderUtils2.asJsonString(credential))
            .accept(MediaType.APPLICATION_JSON)
            .contentType(MediaType.APPLICATION_JSON)
    );
    }

---

### Test de end-point - rollback du transactional ###

---

    //http://localhost:8080/api/UserWebController/getDataTest
    @RequestMapping(path = "/getDataTest", method = RequestMethod.GET)
    public String getDataTest() {
        logger.info("End point getDataTest");
        if (userService.getDataTest()) {
            return "success";
        } else {
            logger.error("Internal server error 500");
            throw new ResponseStatusException(
                    HttpStatus.INTERNAL_SERVER_ERROR, "Internal server error 500"
            );
        }
    }

---

Ce end-point sert pour persister le jeu d'essai dans la base de donnée. Elle utilise l'annotation Transactional avec gestion du rollback.

Pour notre test l'insertion de l'user userTest@test.com va permettre de véfifier le comportement du rollback.

---

    @Service(value = "userService")
    public class UserServiceImpl implements UserDetailsService, IUserService, ITools {

        private final static Logger logger = Logger.getLogger(UserServiceImpl.class);
        @Autowired
        private TokenUtilityProvider tokenUtilityProvider;
        @Autowired
        private AuthProvider authProvider;
        @Autowired
        private SuperModelMapper superModelMapper;
        @Autowired
        private UserRepository userRepository;
        @Autowired
        private RoleRepository roleRepository;
        @Autowired
        private SpaceRepository spaceRepository;
        @Autowired
        private IRoleService roleService;

        @Override
        //catch CustomTransactionalException in this method
        public boolean getDataTest(){
            logger.info("Method getDataTest");
            try {
                testGetDataTest();
            }catch (CustomTransactionalException ex){
                logger.error(ex.getMessage());
                return false;
            }
            return true;
        }

        // Do not catch CustomTransactionalException in this method
        @Transactional(propagation = Propagation.REQUIRES_NEW,
                rollbackFor = CustomTransactionalException.class)
        private void testGetDataTest() throws CustomTransactionalException{
            validateGetDataTest();
        }

        // MANDATORY: Transaction must be created before.
        @Transactional(propagation = Propagation.MANDATORY )
        private void validateGetDataTest(){
            try {
                /**
                no persistence of userTest when rollback
                */
                final User userTest = new User();
                userTest.setEmail("userTest@test.com");
                userTest.setPassword(getStringSha3("userTest"));
                userRepository.save(userTest);

                final Role userRole = new Role();
                userRole.setName(USER);
                final Role managerRole = new Role();
                managerRole.setName(MANAGER);
                final Role adminRole = new Role();
                adminRole.setName(ADMIN);

                roleRepository.save(userRole);
                roleRepository.save(managerRole);
                roleRepository.save(adminRole);

                final Set<Role> userRoles = roleService.getUserRoleSet();
                final Set<Role> managerRoles = roleService.getManagerRoleSet();
                final Set<Role> adminRoles = roleService.getAdminRoleSet();

                final User user1 = new User();
                user1.setEmail("jean@jean.com");
                user1.setGender(Gender.Monsieur);
                user1.setFirstName("Jean");
                user1.setLastName("Leroi");
                user1.setPassword(getStringSha3("0000"));
                user1.setRoles(adminRoles);
                user1.setPhoneNumber("0606060606");
                user1.setAddress("3 rue du Roi");
                user1.setZip("95000");
                user1.setCity("Cergy");
                user1.setDeliveryInformation("2ème étage");
                final User user2 = new User();
                user2.setEmail("jeanne@jeanne.com");
                user2.setPassword(getStringSha3("0000"));
                user2.setRoles(managerRoles);
                final User user3 = new User();
                user3.setEmail("john@john.com");
                user3.setPassword(getStringSha3("0000"));
                user3.setRoles(userRoles);
                final User user4 = new User();
                user4.setEmail("johny@johny.com");
                user4.setPassword(getStringSha3("0000"));
                user4.setRoles(userRoles);
                user1.setStatus(Status.ACTIVE);
                user2.setStatus(Status.ACTIVE);
                user3.setStatus(Status.ACTIVE);
                user4.setStatus(Status.INACTIVE);

                userRepository.save(user1);
                userRepository.save(user2);
                userRepository.save(user3);
                userRepository.save(user4);

                final Space space1 = new Space();
                space1.setName("Jean@jean.com space");
                space1.setUser(user1);
                spaceRepository.save(space1);

                final Space space2 = new Space();
                space2.setName("jeanne@jeanne.com space");
                space2.setUser(user2);
                spaceRepository.save(space2);

                final Space space3 = new Space();
                space3.setName("john@john.com space");
                space3.setUser(user3);
                spaceRepository.save(space3);

                final Space space4 = new Space();
                space4.setName("johny@johny.com space");
                space4.setUser(user4);
                spaceRepository.save(space4);
            }catch(Exception ex){
                throw new CustomTransactionalException("data tests failed");
            }
        }

---       

Jeu de test : 

* Requête ok : retourne Status 200 avec success
    * objets persistés
* Requête ko : retourne Status Internal server error 500
    * objets non persistés

---  

    /**
    * Initialize spring context at each tests
    */
    @RunWith(SpringRunner.class)
    @SpringBootTest
    @AutoConfigureMockMvc
    @ActiveProfiles("test")
    @DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
    public class UserWebControllerTest1 implements ITools {

        private static final String BASE_URL = PRE_PATH + USER_CONTROLLER;
        @Autowired
        private JdbcTemplate jdbcTemplate;
        @Autowired
        private MockMvc mockMvc;
        @Autowired
        private BuilderUtils2 builderUtils2;
        @Autowired
        private UserRepository userRepository;

        @Test
        public void test_getDataTest_should_return_status_200_with_success_and_with_results() throws Exception {
            //given
            jdbcTemplate.execute(
                    "delete from cuisine_users_roles; ");
            jdbcTemplate.execute(
                    "delete from cuisine_role; ");
            jdbcTemplate.execute(
                    "delete from cuisine_space; ");
            jdbcTemplate.execute(
                    "delete from cuisine_user; ");
            //when
            final ResultActions resultActions = mockMvc.perform(get(BASE_URL + "/getDataTest").accept(MediaType.APPLICATION_JSON));

            //then
            final MvcResult mvcResult = resultActions.andExpect(status().isOk())
                    .andExpect(jsonPath("$", is("success")))
                    .andReturn();
            final String result = mvcResult.getResponse().getContentAsString();
            Assert.assertEquals("success", result);
            /**
            * check persistence of objects in IUserService.getDataTest()
            */
            final Credential credential = new Credential();
            credential.setEmail("jean@jean.com");
            credential.setPassword("0000");
            /**
            * userTest cannot be retrieved with getUsers because userTest has no roles (UserUserDTOConverter condition)
            */
            final MvcResult mvcResult2 = invokeGetUsers(credential)
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$", hasSize(4)))
                    .andExpect(jsonPath("$[0].email", is("jean@jean.com")))
                    .andExpect(jsonPath("$[1].email", is("jeanne@jeanne.com")))
                    .andExpect(jsonPath("$[2].email", is("john@john.com")))
                    .andExpect(jsonPath("$[3].email", is("johny@johny.com")))
                    .andExpect(jsonPath("$[0].id", is(2)))
                    .andExpect(jsonPath("$[1].id", is(3)))
                    .andExpect(jsonPath("$[2].id", is(4)))
                    .andExpect(jsonPath("$[3].id", is(5)))
                    .andReturn();
            /**
            * userTest cannot be retrieved with getUser because userTest has no roles (UserUserDTOConverter condition)
            */
            final ResultActions resultActions3 = invokeGetUser(credential, "userTest@test.com");
            final MvcResult mvcResult3 = resultActions3
                    .andExpect(status().isNotFound())
                    .andReturn();
            /**
            * userTest exist in memory test (or database) and the entity can be retrieve with userRepository.selectMyUserByEmail
            */
            final User user = userRepository.selectMyUserByEmail("userTest@test.com");
            Assert.assertEquals("userTest@test.com", user.getEmail());
        }

        /**
        * when getDataTest then throw sql exception constraint of unique column + constraint of duplicate key
        * rollback should works
        */
        @Test
        public void test_getDataTest_transactional_failed_should_status_500_with_no_new_persistence() throws Exception {
            //given
            final String sha3Password = getStringSha3("0000");
            jdbcTemplate.execute(
                    "delete from cuisine_users_roles; ");
            jdbcTemplate.execute(
                    "delete from cuisine_role; ");
            jdbcTemplate.execute(
                    "delete from cuisine_space; ");
            jdbcTemplate.execute(
                    "delete from cuisine_user; ");
            jdbcTemplate.execute(
                    "insert into cuisine_role(id, name) values (1, 'USER'); " +
                            "insert into cuisine_role(id, name) values (2, 'MANAGER'); " +
                            "insert into cuisine_role(id, name) values (3, 'ADMIN'); "
            );
            jdbcTemplate.execute(
                    "delete from cuisine_user; " +
                            "insert into cuisine_user(id, email, password) values (1, 'jean@jean.com', '" + sha3Password + "'); ");
            jdbcTemplate.execute(
                    "insert into cuisine_users_roles(cuisine_user_id, cuisine_role_id) values (1,1); " +
                            "insert into cuisine_users_roles(cuisine_user_id, cuisine_role_id) values (1,2); " +
                            "insert into cuisine_users_roles(cuisine_user_id, cuisine_role_id) values (1,3); "
            );
            //when
            final ResultActions resultActions = mockMvc.perform(get(BASE_URL + "/getDataTest").accept(MediaType.APPLICATION_JSON));

            //then
            final MvcResult mvcResult = resultActions.andExpect(status().isInternalServerError())
                    .andReturn();
            /**
            * check persistence of objects in IUserService.getDataTest()
            *  only jean@jean.com has been persisted on db (or memory test)
            */
            final Credential credential = new Credential();
            credential.setEmail("jean@jean.com");
            credential.setPassword("0000");
            final MvcResult mvcResult2 = invokeGetUsers(credential)
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$", hasSize(1)))
                    .andExpect(jsonPath("$[0].email", is("jean@jean.com")))
                    .andReturn();
            final User user = userRepository.selectMyUserByEmail("userTest@test.com");
            Assert.assertNull("return null", user);
        }

        /**
        * Invoke end-points
        */
        private ResultActions invokeGetUser(final Credential credential, final String email) throws Exception{
            final String token = getAccessToken(credential);
            return mockMvc.perform(get(BASE_URL + "/getUser")
                    .param("email",email)
                    .header("Authorization", "Bearer " + token)
                    .accept(MediaType.APPLICATION_JSON)
            );
        }

        private ResultActions invokeGetUsers(final Credential credential) throws Exception {
            final String token = getAccessToken(credential);
            return mockMvc.perform(get(BASE_URL + "/getUsers")
                    .header("Authorization", "Bearer " + token)
                    .accept(MediaType.APPLICATION_JSON));
        }

        private String getAccessToken(final Credential credential) throws Exception {
            final MvcResult mvcResult = invokeValidateConnection(credential)
                    .andReturn();
            final Token token = builderUtils2.fromJsonResult(mvcResult, Token.class);
            return token.getToken();
        }

        private ResultActions invokeValidateConnection(final Credential credential) throws Exception {
            return mockMvc.perform(post(BASE_URL + "/validateConnection")
                    .content(builderUtils2.asJsonString(credential))
                    .accept(MediaType.APPLICATION_JSON)
                    .contentType(MediaType.APPLICATION_JSON)
            );
        }

    }

## Conclusion ##

Merci!