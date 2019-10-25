---
title: "Craftsmanship"
date: 2019-10-25T15:02:42+02:00
draft: false
author: "Jeannory"
tags: ["articles"]
categories: ["Craftsmanship"]
---
### Introduction ###

Pour présentation.

Retrouver le dépôt git : [ici](https://github.com/jeannory/SpringSecurityExampleBE)

### Sommaire ###

toDo

### Fonctionnalités ###

toDo

### Stack technique ###

toDo

## Tests unitaires - Junits & Mockito ##

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

Déclarer la class à tester et ses dépendances (ici un cas simple avec une seule dépendance):

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

        private boolean isValidEmail(String email) {
            String regex = "^[\\w-_\\.+]*[\\w-_\\.]\\@([\\w]+\\.)+[\\w]+[\\w]$";
            return email.matches(regex);
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


## Tests persistence - DataJpaTest ##

Test des repository avec une base de données embarquée

Dépendance à ajouter

        <!-- https://mvnrepository.com/artifact/com.h2database/h2 -->
                <dependency>
                    <groupId>com.h2database</groupId>
                    <artifactId>h2</artifactId>
                    <version>1.4.200</version>
                    <scope>test</scope>
                </dependency>

---

Repo à tester

    @CrossOrigin("*")
    @Repository
    public interface UserRepository extends JpaRepository<User, Long> {
        User findByEmail(String email);
        @Query(value = "select * from cuisine_user u where u.email ilike :paramEmail", nativeQuery=true)
        User selectMyUserByEmail(@Param("paramEmail") String email);
    }

---

jeu de test

* test_findByEmail_when_all_parameters_valid_should_return_result
* test_findByEmail_when_user_no_have_roles_should_return_user
* test_findByEmail_when_no_user_found_should_return_null
* test_selectMyUserByEmail_when_all_parameters_valid_should_return_result
* test_selectMyUserByEmail_when_user_no_have_roles_should_return_user
* test_selectMyUserByEmail_when__no_user_found_should_return_null

La bdd se réinitialise à chaque test

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

## Tests de controleurs - In-Container ##

toDo

### Test de end-point - statut de retour ###

toDo

### Test de end-point - rollback du transactional ###

toDo

## Conclusion ##

