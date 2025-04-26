
# Questions

## 1 – What are Authentication and Authorization ?

- Authentication (Kimlik Doğrulama): Kullanıcının kim olduğunu doğrulama sürecidir.
- Authorization (Yetkilendirme): Doğrulanmış kullanıcının hangi kaynaklara erişebileceğini belirler.

Bir banka uygulamasında kullanıcı adı ve parola girerek sisteme giriş (Authentication) yaptıktan sonra, kullanıcının sadece kendi hesap bilgilerine erişmesine izin verilmesi, başka hesaplara erişimin engellenmesi (Authorization).

### Örnek

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // ——— HTTP zinciri ———
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http, JwtDecoder decoder) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.POST, "/auth/login").permitAll()   // login serbest
                .requestMatchers("/admin/**").hasRole("ADMIN")                 // yetkilendirme
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2   // Bearer token doğrulama
                .jwt(jwt -> jwt.decoder(decoder))
            );
        return http.build();
    }

    // ——— Demo kullanıcıları (authentication) ———
    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder pe) {
        UserDetails admin = User.withUsername("admin").password(pe.encode("123")).roles("ADMIN").build();
        UserDetails user  = User.withUsername("user").password(pe.encode("123")).roles("USER").build();
        return new InMemoryUserDetailsManager(admin, user);
    }

    @Bean               // BCrypt tavsiye edilir
    public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }

    // ——— JWT altyapısı (HMAC secret) ———
    private static final String SECRET = "my-secret-key-should-be-256-bit-long!!!";
    @Bean
    public JwtEncoder jwtEncoder() {
        SecretKey key = Keys.hmacShaKeyFor(SECRET.getBytes(StandardCharsets.UTF_8));
        return new NimbusJwtEncoder(new ImmutableSecret<>(key));
    }

    @Bean
    public JwtDecoder jwtDecoder() {
        SecretKey key = Keys.hmacShaKeyFor(SECRET.getBytes(StandardCharsets.UTF_8));
        return NimbusJwtDecoder.withSecretKey(key).build();
    }
}
```

## 2 – What is Hashing in Spring Security ?

Hashing, şifre gibi hassas verilerin tek yönlü algoritmalar kullanılarak şifrelenmesidir. Girilen veriden aynı hash değerini üretir fakat hash’ten orijinal veriye geri dönmek mümkün değildir. Kullanıcı şifresi açık olarak veritabanında tutulmaz, Spring Security tarafından hashlenerek güvenli hale getirilir.

```java
// 1) SecurityConfig — Parolaları nasıl kodlayacağımızı bildiriyoruz
@Configuration
public class SecurityConfig {

    @Bean                                 
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(10);
    }
}

// 2) UserService — Yeni kullanıcı kaydederken parolayı hash’le
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepo;
    private final PasswordEncoder encoder;   

    public void register(RegisterDto dto) {
        UserEntity user = new UserEntity();
        user.setUsername(dto.username());
        user.setPassword(encoder.encode(dto.password())); 
        userRepo.save(user);                              
    }
}

// 3) Otomatik doğrulama — Form login / JWT vb. akışta
boolean ok = passwordEncoder.matches(rawPasswordFromLoginForm, hashedPasswordFromDb);
```

## 3 –  What is Salting and why do we use the process of Salting ?
Salting, şifre hashlenirken rastgele bir değer ekleyerek şifre güvenliğini artırma yöntemidir. Böylece aynı şifreye sahip kullanıcıların hash değerleri farklı olur. Bir web sitesinde iki farklı kullanıcı aynı şifreyi seçtiğinde, salting sayesinde hash değerleri birbirinden farklı olur ve saldırganların şifreleri çözmesi zorlaşır.

### Örnek

```java
byte[] salt = new byte[16];
SecureRandom rand = new SecureRandom();
rand.nextBytes(salt);                                  // rastgele salt

MessageDigest md = MessageDigest.getInstance("SHA-256");
md.update(salt);                                       // salt + parola
byte[] hash = md.digest(password.getBytes(StandardCharsets.UTF_8));

String saltHex = HexFormat.of().formatHex(salt);       // ikili → hex
String hashHex = HexFormat.of().formatHex(hash);

user.setSalt(saltHex);                                 // DB’ye hem salt hem hash yaz
user.setPasswordHash(hashHex);
```

## What is “intercept-url” pattern ?

Intercept-url, Spring Security’de belirli URL’lere erişimi kısıtlamak veya izin vermek için kullanılır. Örneğin Yönetici paneline (/admin/**) sadece admin yetkisi olan kullanıcıların erişmesine izin verilmesi için kullanılır.

### Örnek Kod

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
          .authorizeHttpRequests(auth -> auth
              .requestMatchers("/admin/**").hasRole("ADMIN")
              .requestMatchers(HttpMethod.GET, "/docs/**").permitAll()
              .anyRequest().authenticated()
          )
          .formLogin();   // veya JWT, OAuth2 Resource Server
        return http.build();
    }
}
```

## 5 – What is @Query used for ?

Session management; kimliği doğrulanmış bir kullanıcının oturumunun
- Ne zaman ve nasıl oluşturulacağını,
- Sunucu belleğinde nerede tutulacağını,
- Kaç eş-zamanlı oturuma izin verileceğini,
- Ne kadar süre etkin kalacağını,
- Oturum hırsızlığı / fixation gibi saldırılara karşı nasıl korunacağını yöneten politika ve mekanizmaların tamamıdır.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
          // 1) Oturum politikası: REST API'ysek STATELESS seçeriz
          .sessionManagement(sm -> sm
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED) // varsayılan
                .sessionFixation().migrateSession()                      // JSESSIONID yenile
                .invalidSessionUrl("/login?expired")                     // süre dolarsa
          )

          // 2) Aynı kullanıcı en fazla bir yerde açık kalsın
          .sessionManagement(sm -> sm
                .maximumSessions(1)               // >1 giriş olursa…
                .maxSessionsPreventsLogin(false)  // yenisi girerse eskisini düşür
          )

          // 3) Kalan normal ayarlar
          .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
          )
          .formLogin();        // stateful form login
        return http.build();
    }
}
```

## 6 – Why we need Exception Handling ?
Hataları yakalayıp kontrollü biçimde kullanıcıya veya API istemcisine güvenli, standart bir yanıt döndürme süreci.

- AuthenticationEntryPoint → Kimliği doğrulanmamış isteklerde 401/redirect
- AccessDeniedHandler → Yetki yetersizse 403/redirect

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    // Genel bir Exception yakalayıcı
    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleAllExceptions(Exception ex) {
        return new ResponseEntity<>("Beklenmeyen bir hata oluştu: " + ex.getMessage(), HttpStatus.INTERNAL_SERVER_ERROR);
    }

    // Özel bir exception yakalayıcı (örnek: IllegalArgumentException)
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<String> handleIllegalArgumentException(IllegalArgumentException ex) {
        return new ResponseEntity<>("Geçersiz bir argüman girdiniz: " + ex.getMessage(), HttpStatus.BAD_REQUEST);
    }
}
```

## 7 – Explain what is AuthenticationManager in Spring security ?
Çeşitli AuthenticationProvider bileşenlerini orkestre ederek kimlik doğrulama yapan merkezi servis.

Uygulamanız hem form-login (parola) hem de JWT ile çalışıyorsa:
- İlk istekte DAO provider parola kontrolü yapar.
- Sonraki isteklerde JWT provider token’ı doğrular.


## 8 – What is Spring Security Filter Chain ?
web uygulamasına gelen her HTTP isteğini sırayla kontrol eden, güvenlik filtrelerinden oluşmuş bir zincirdir.
Her istek controller’a ulaşmadan önce bu filtrelerin hepsinden geçmek zorundadır.

İstek → güvenlik filtresi → diğer güvenlik filtresi → … → en son controller’a geçiş izni

```java
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<User> query = cb.createQuery(User.class);
Root<User> root = query.from(User.class);

List<Predicate> predicates = new ArrayList<>();
if (name != null) {
    predicates.add(cb.equal(root.get("name"), name));
}
if (email != null) {
    predicates.add(cb.equal(root.get("email"), email));
}
query.where(predicates.toArray(new Predicate[0]));
List<User> users = entityManager.createQuery(query).getResultList();
```

## 9 – What are the differences between OAuth2 and  JWT ?

### OAuth2 bir yetkilendirme protokolüdür.
Bir kullanıcının, başka bir uygulamaya (örneğin mobil app’e veya üçüncü parti siteye) sınırlı erişim vermesini sağlar. “Ben Google hesabımla giriş yapıyorum” diyorsan bu bir OAuth2 sürecidir.

### JWT (JSON Web Token) bir token formatıdır.
Kullanıcı bilgilerini veya yetkilendirme bilgisini taşıyan, imzalanmış ve bazen şifrelenmiş küçük bir veri paketidir. Genellikle OAuth2 içinde veya bağımsız sistemlerde kullanılır.

### Kullanım şekilleri farklıdır.
- OAuth2, bir “Authorization Server” ile çalışır (mesela: Google, GitHub, Facebook). Kullanıcıyı doğrular, ardından uygulamana bir Access Token verir. Bu Access Token bazen JWT formatında olur, bazen farklı olabilir.
  
- JWT ise doğrudan API isteklerinde taşınır. Genellikle HTTP Header’ın Authorization: Bearer <token> kısmında gönderilir. OAuth2, JWT kullanabilir ama kullanmak zorunda değildir.
  

### JWT tamamen stateless çalışır.
-	JWT token’ı sunucuya her istekle birlikte gönderirsin.
- Sunucu bu token’ın içeriğini okuyarak kim olduğunu ve yetkilerini anlar.
- Sunucu hafızasında session bilgisi tutmaz, her şey token’ın içindedir.
  
### OAuth2 ise hem stateful hem de stateless çalışabilir.
- Refresh Token gibi mekanizmalar kullanıyorsa, sunucunun bir noktada durum (state) tutması gerekir.
- Mesela bir Refresh Token sunucuda kara listeye alınabilir.


## 10 - What is method security and why do we need it ?
Java veya Spring uygulamanda, sadece URL’leri değil, doğrudan method seviyesinde güvenlik önlemi alman anlamına gelir. 

Yani sadece /admin/** gibi endpointlere erişimi kısıtlamakla kalmazsın, aynı zamanda servis içindeki belirli methodlara da kimlerin erişebileceğini kontrol edersin.

Spring Security bunu sağlayan bazı anotasyonlar sunar:
- @PreAuthorize
- @PostAuthorize
- @Secured
- @RolesAllowed

```java
PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long userId) {
    // Sadece ADMIN rolüne sahip kullanıcılar bu metodu çalıştırabilir
}
```

## 11 – What Proxy means and how and where can be used ?

Bir işlem doğrudan gerçek nesneye gitmez önce Proxy’ye gider, Proxy araya girer ve isterse isteği işler, isterse iletir.

Proxy farklı şekillerde kullanılabilir:
### Access Control (Erişim Kontrolü)
Proxy, istek yapan kişinin yetkili olup olmadığını kontrol edebilir.
Örneğin: Bir nesneye sadece belirli kullanıcıların ulaşabilmesini sağlamak.
### Lazy Initialization (Tembel Yükleme)
Proxy nesnesi, gerçek nesneyi gerektiğinde oluşturur. Büyük bir veri nesnesi, ancak gerçekten erişildiğinde belleğe alınır.
### Logging (Kayıt Tutma)
Proxy, bir method çağrılmadan önce ve sonra log kaydı alabilir.
Hangi methodlar çalıştırıldı, ne kadar sürede çalıştı gibi bilgileri toplamak için.
### Remote Access (Uzak Erişim)
Proxy, uzak bir sunucudaki servisi yerel gibi gösterir.
Örneğin: Bir Java uygulaması uzaktaki bir servisle haberleşirken sanki yerelmiş gibi proxy kullanır (RMI, gRPC gibi).
### Caching (Önbellekleme)
Proxy, bir methodun sonucunu önbelleğe alıp aynı istek tekrar geldiğinde hızlıca dönebilir.
Böylece gereksiz yere gerçek nesneye gitmekten tasarruf edilir.

## 12 – Waht is Wrapper Class and where can be used ?

Java’daki ilkel tipleri (primitive types) sınıf (class) yapısına sarar, nesne gibi kullanılmasını sağlar.

Java koleksiyonları (List, Set, Map) primitive veri tiplerini doğrudan kabul etmez. Bu yüzden bir List içine int koyamazsın; Integer kullanman gerekir.

```java
public class IntWrapper {
    private int value;  // primitive değerimizi tutacağız

    // Constructor
    public IntWrapper(int value) {
        this.value = value;
    }

    // Getter
    public int getValue() {
        return value;
    }

    // Setter
    public void setValue(int value) {
        this.value = value;
    }

    // İsteğe bağlı: değeri artıran yardımcı method
    public void increment() {
        this.value++;
    }

    // İsteğe bağlı: toString override
    @Override
    public String toString() {
        return "IntWrapper{value=" + value + "}";
    }
}
```
## 13 - What are the properties of an entity ?

SSL (Secure Sockets Layer), internet üzerinden veri aktarımını güvenli hale getiren eski bir şifreleme protokolüdür. Sunucu ile tarayıcı (veya iki sunucu) arasındaki verileri şifreler. Böylece üçüncü kişiler bu verileri dinleyip anlayamaz.

TLS (Transport Layer Security),SSL’in daha gelişmiş, daha güvenli ve modern versiyonudur. SSL protokolünün yerini almak için tasarlanmıştır. Şu anda internette kullanılan HTTPS bağlantılarının çoğu TLS kullanır.

Örneğin, Spring Boot’ta şöyle yaparsın:
```yaml
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: yourpassword
    key-store-type: PKCS12
```

## 14 - Why do you need the intercept-url ?

intercept-url kullanmamızın sebebi, web uygulamasında belirli URL’lere (yollara) kimlerin erişebileceğini merkezi bir yerden kontrol etmek içindir.

Belirli yolları belirli rollere göre koruyabilirsin:

- /admin/** sadece ADMIN rolü olanlar görsün.
- /profile/** sadece giriş yapmış kullanıcılar görsün.
- /public/** herkes görebilsin.

 ```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")        
    .requestMatchers("/profile/**").authenticated()       
    .requestMatchers("/public/**").permitAll()            
    .anyRequest().denyAll()                               
);
```
