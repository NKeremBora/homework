
# Questions

## 1 – What is the difference between manual testing and automated testing ?

- Manuel test: Test senaryoları insan tarafından çalıştırılır. Avantajı sezgisel keşif (exploratory testing) yapabilmek ve UI/UX problemlerini görmektir. Dezavantajı yavaştır, tekrarlarda hata yapma olasılığı yüksektir ve geriye dönük izlenebilirlik zayıftır.
  
- Otomasyon testi: Senaryolar kodla (Selenium, JUnit, Cypress vb.) yazılır; CI/CD içinde defalarca, hızlı ve hatasız koşar. 


## 2 – What does Assert class ?

JUnit’te org.junit.jupiter.api.Assertions (Jupiter) veya eski sürümlerde org.junit.Assert, beklenen durumla (expected) gerçek sonucu (actual) karşılaştıran statik yöntemler içerir (assertEquals, assertTrue, vb.). Koşul başarısızsa test senaryosunu fail durumuna geçirir; böylece CI pipe’ı “kırılır”. 

```java
class Calculator {

    int add(int a, int b)  { return a + b; }
    int divide(int a, int b) { return a / b; }
    boolean isPositive(int n) { return n > 0; }
}

class CalculatorTest {

    private final Calculator calc = new Calculator();

    @Test
    @DisplayName("Toplama doğru sonucu döndürmeli")
    void shouldReturnCorrectSum() {
        // Arrange + Act
        int result = calc.add(2, 3);

        // Assert
        Assertions.assertEquals(5, result, "2 + 3 sonucu 5 olmalı");
    }

    @Test
    @DisplayName("Sıfırdan büyük sayılar pozitif olmalı")
    void shouldIdentifyPositiveNumbers() {
        Assertions.assertTrue(calc.isPositive(7), "7 pozitiftir");
        Assertions.assertFalse(calc.isPositive(-1), "-1 negatif olduğu için false dönmeli");
    }

    @Test
    @DisplayName("Sıfıra bölme IllegalArgumentException fırlatmalı")
    void shouldThrowWhenDivideByZero() {
        Assertions.assertThrows(ArithmeticException.class,
            () -> calc.divide(4, 0),
            "Sıfıra bölme ArithmeticException üretmeli");
    }

    @Test
    @DisplayName("Birden fazla doğrulamayı tek seferde raporla")
    void shouldCheckMultipleAssertionsTogether() {
        Assertions.assertAll("toplu doğrulamalar",
            () -> Assertions.assertEquals(8, calc.add(5, 3)),
            () -> Assertions.assertTrue(calc.isPositive(1)),
            () -> Assertions.assertThrows(ArithmeticException.class, () -> calc.divide(1, 0))
        );
    }
}
```

## 3 –  How can be tested 'private' methods ?
private metodu kullanan public API’yı test edin; gerçek davranış zaten yansır.

### Örnek

```java

public class TaxCalculator {

    // gizli iş mantığı
    private double calculateTax(double net) {
        return net * 0.20;          // %20 KDV
    }

    /** PUBLIC API ─ test edilmesi önerilen nokta */
    public double totalWithTax(double net) {
        return net + calculateTax(net);
    }
}

class TaxCalculatorTest {

    private final TaxCalculator calc = new TaxCalculator();

    @Test
    void shouldAdd20PercentTax() {
        // Arrange & Act
        double total = calc.totalWithTax(100.0);

        // Assert
        assertEquals(120.0, total, 0.0001,
                     "100 birim net → 120 brüt dönmeli");
    }
}
```

## 4 – What is Monolithic Architecture ?
Uygulamanın tüm alan (UI, iş kuralları, veri erişimi) katmanlarının tek deployable (JAR/WAR/EXE) içinde bulunduğu, ölçeklendirmenin tüm uygulamayı çoğaltarak yapıldığı mimaridir. Tek kod tabanı - tek veritabanı - tek pipeline; değişiklikler genelde tüm sistemi etkiler.

## 5 – What are the best practices to write a Unit Test Case ?
- AAA (Arrange-Act-Assert) bloklarını net ayırın.
- Test adını “shouldDoX_whenY” şeklinde davranış odaklı verin.
- Dış bağımlılıkları Mockito/Fake ile izole edin.
- 1 test = 1 davranış; yan etkisiz, sıralamadan bağımsız olsun.
- “Gizli” assert’ler yerine açık assertThat (AssertJ, Hamcrest) kullanın.
- Süre < 200 ms; build’i yavaşlatmasın.
- Test verisi net; magic number yok; fixture’ı yeniden kullanın.

## 6 – Why we need Exception Handling ?
Bir test metodu başarısız olduğunda JVM stack’i orada kesilir. JUnit, test bütünlüğü bozulmasın diye kalan kodu çalıştırmaz; dolayısıyla yalnızca ilk AssertionError görünür. Çoklu doğrulama gerekiyorsa:
- JUnit 5’in assertAll bloğu, tüm assert’leri toplayıp tek seferde raporlar.
- Veya senaryoyu ayrı test metodlarına bölün.

## 7 – What are the benefits and drawbacks of Microservices ?
- Bağımsız dağıtım (independent deploy), farklı ekip-teknoloji seçimi.
- Bölgesel ölçekleme: sadece dar boğaz hizmeti çoğaltılır.
- Arızaya dayanıklılık: bir servis düşse tüm sistem çökmez.

- Operasyonel karmaşıklık (network, gözlemlenebilirlik, versiyonlama).
- Dağıtık transaction, tutarlılık sorunları (CAP, Saga, vb.).
- Geliştirici deneyimi: lokal ortamda tüm sistemi ayağa kaldırmak zor.
- DevOps/CI/CD olgunluğu şart; aksi hâlde “dağıtık monolit” oluşur.


## 8 – What is the role of actuator in spring boot ?
spring-boot-starter-actuator, uygulamaya hazır gözlem uç noktaları (endpoints) ekler: /health, /metrics, /env, /loggers, /prometheus vb. Böylece Kubernetes liveness/readiness probe, Prometheus scraping, dinamik log seviyesi değişimi gibi işlemler kod yazmadan yapılır.

## 9 – What are the differences between OAuth2 and  JWT ?
- Servis keşfi ve yük dengeleme (Eureka, Consul, Istio).
- Konfigürasyon yönetimi (Spring Cloud Config, Vault).
- Dağıtık izleme & tracing (OpenTelemetry, Jaeger, Loki).
- Tutarlılık, Saga/Outbox, idempotent tasarım.
- Versiyon uyumu backward-compat API, canary release.
- Güvenlik, merkezi kimlik (OAuth2, Keycloak) + zero-trust network.
- her servise özel şema (Database per service) => raporlama karmaşıklığı.


## 10 - How independent microservices communicate with each other?
### Senkron İletişim 
Gönderen servis, yanıtı alana kadar bloklanır. İstek–yanıt aynı bağlantı (HTTP/2, gRPC-unary, REST) üzerinde “anında” gerçekleşir.
#### 1. HTTP/REST + JSON
HTTP/1.1 veya HTTP/2 (Spring WebFlux, Quarkus RESTEasy).
- Stateless endpoint, açık timeout (<< 3 sn) ve retry-safe idempotent GET/PUT.
- Versioning: /v1/users → /v2/users.
- Circuit breaker (Resilience4j, Istio) ile “fail-fast”.


```java
// Spring Boot (WebClient) senkron örnek
String total = WebClient.create("http://order")
        .get().uri("/v1/total/{id}", orderId)
        .retrieve()
        .bodyToMono(String.class)
        .timeout(Duration.ofSeconds(2))
        .block();
```
##### 2. gRPC (HTTP/2)
IDL: Protocol Buffers; contract-first.
- Unary (request/response)
- Server-streaming (push)
- Client-streaming (upload)
- Bidirectional-streaming (chat, IoT telemetry)

```proto
syntax = "proto3";

package pricing;                

// ---- Service ----
service Pricing {
  rpc GetQuote (QuoteRequest) returns (QuoteReply);
}

// ---- Messages ----
message QuoteRequest {
  string symbol = 1;           
}

message QuoteReply {
  string symbol    = 1;
  double bid_price = 2;
  double ask_price = 3;
  int64  timestamp = 4;        
}
```
### Asenkron (Olay-Tabanlı) İletişim
Gönderen, mesajı bir aracıya (kuyruğa/lojik log’a) bırakır ve işlemini sürdürür; alıcı daha sonra tüketip cevap verebilir. Trafik “fire-and-forget” veya “event-driven” şeklindedir (Kafka, RabbitMQ, NATS, gRPC-streaming).
#### 1. Apache Kafka
- Topic = Parçalanabilir log dizisi; consumer offset ile “replay”.
- Tipik pattern:
- Choreography Saga – “OrderCreated → PaymentProcessed → StockReserved”.
- CQRS – Write tarafı komut, read tarafı farklı storage.
- Idempotency: orderId + eventType hash tutarak çift işlem engellenir.
#### 2. RabbitMQ 
- Work-queue: arka planda görev işleme (thumbnail, e-posta).
- Pub/Sub Routing Key: invoice.* → tüm fatura event’leri.
- DLQ (Dead Letter Queue) ve TTL ile sorunlu mesaj izolasyonu.

## 11 – What do you mean by Domain driven design ?

İş problemini “domain” ve “bounded context”’ler bazında modelleyip, terimleri geliştirici-iş birliğiyle belirleyen (ubiquitous language) tasarım yaklaşımıdır. Mikroservis sınırlarını  çizmek ve karmaşayı azaltmak için kullanılan taktiksel kalıplar (Entity, Value Object, Aggregate, Repository, Domain Event, vb.) içerir.
- Domain: Yazılımın çözdüğü gerçek dünya problem alanıdır. Örneğin, “sipariş yönetimi” veya “öğrenci kayıt sistemi”.
- Bounded Context: Ortak dilin geçerli olduğu sınırlı alandır. Her bounded context kendi modeline, veritabanına ve iş kurallarına sahiptir.
- Context Map: Farklı bounded context’lerin birbirine nasıl bağlandığını gösteren diyagram veya anlaşmalar bütünüdür.
- Entity: Kimliği (ID) olan, zamanla durumu değişebilen iş nesnesidir. Örneğin, bir Order nesnesi.
- Value Object: Kimliği olmayan, yalnızca değer taşıyan ve değiştirilmeyen küçük nesnelerdir. Örneğin, Money veya Address.
-	Aggregate: Bir arada tutarlılığı korunması gereken Entity ve Value Object kümesidir. Dış dünya Aggregate Root üzerinden erişir.
- ggregate Root: Aggregate içindeki ana nesnedir. Dış kod Aggregate’ın iç elemanlarına doğrudan ulaşamaz, sadece root üzerinden erişir.
- Repository: Entity veya Aggregate’ları veritabanına kaydeden veya oradan getiren soyutlamadır.
- Domain Event: Domain’de gerçekleşen önemli bir olaydır. Mesela OrderPaid gibi.
-	Domain Service:Tek bir Entity’ye ait olmayan iş kurallarını içeren saf iş mantığı sınıfıdır.
- Application Service: Use-case’leri yöneten katmandır. Domain nesneleriyle çalışır ama iş kurallarını doğrudan içermez.
- Factory: Karmaşık nesne veya Aggregate üretim işlemini yöneten yapıdır.

## 12 – What is container in Microservices ?
Container, bir mikroservisin tüm kodunu, kütüphanelerini, bağımlılıklarını ve çalıştırma ortamını (runtime environment) tek bir paket haline getiren hafif ve taşınabilir bir birimdir.
Bu sayede mikroservis, hangi makinede çalıştığına bakmaksızın aynı şekilde çalışır: geliştiricinin bilgisayarında, test ortamında, bulutta…

### Linux seviyesinde container, aslında 2 temel özelliğin birleşimidir:
#### Namespace’ler (İsim Alanları)
- Bir container kendi ağını, dosya sistemini, işlem ağacını diğerlerinden izole eder.
- Örneğin, her container kendi eth0 ağına ve kendi / kök dizinine sahipmiş gibi görünür.
#### Control Groups (cgroups)
- Her container’a CPU, bellek, disk gibi kaynaklardan sınır koyar.
- Böylece bir container aşırı kaynak tüketse bile diğerlerini etkilemez.

Yani Linux aslında:
- Aynı kernel’i paylaşan,
- İzole edilmiş,
- Hafif (bir VM gibi ağır olmayan)
çalışma ortamları yaratır.

Docker ve benzeri sistemler bu izolasyonu kullanıcı dostu hale getirip otomatikleştirir.

## 13 - What are the main components of Microservices architecture ?

- API Gateway: Tüm istemci isteklerinin girdiği tek giriş noktasıdır. Kimlik doğrulama (authentication), yetkilendirme (authorization), rate limiting, yük dengeleme (load balancing) gibi işleri yapar.
- Service Discovery: Mikroservislerin ağdaki yerini (IP/port) bulmayı sağlar. Çünkü mikroservisler dinamik olarak ölçeklenir ve yerleri değişebilir.
- Configuration Management: Mikroservisler çalışma zamanında konfigürasyon (veritabanı URL’si, API anahtarları, özellik bayrakları vs.) bilgilerini buradan alır.
- Service Communication: Mikroservisler arasında iletişim kurmak gerekir:
  - Senkron: REST API, gRPC
  - Asenkron: Kafka, RabbitMQ, NATS
  - Ayrıca Service Mesh kullanılarak bu iletişim güvenli ve izlenebilir hale getirilebilir (mTLS, retry, timeout).
- Data Management: Her mikroservis kendi veritabanına sahip olur (Database per service prensibi).
- Observability (Monitoring, Logging, Tracing): Sistem sağlığını ve sorunları görebilmek için:
  - Logging: Centralized log toplama (ELK, Loki)
  - Metrics: Sistem performansı (Prometheus, Grafana)
  - Tracing: İsteklerin servisler arası akışını izleme (Jaeger, Zipkin)
- Security: Kimlik doğrulama (OAuth2, OpenID Connect), Yetkilendirme (role-based access control), Servisler arası güvenli iletişim (mTLS, API Gateway kontrolleri)
- Containerization & Orchestration:	Mikroservisler container içine alınır ve containerlar yönetilir. Otomatik ölçekleme, iyileşme (self-healing), rollout/rollback gibi özellikler burada sağlanır.
- CI/CD Pipeline: Mikroservislerin hızlı, güvenli ve otomatik bir şekilde inşa edilmesi, test edilmesi ve dağıtılması için CI/CD süreçleri kurulur.


## 14 - How does a Microservice architecture work?

- Uygulama küçük, bağımsız servislere bölünür. Her mikroservis tek bir iş alanına odaklanır.
- Her servis kendi veritabanına sahiptir. Mikroservisler veritabanı paylaşmaz.
- Servisler birbiriyle iletişim kurar.
  - Senkron iletişim: REST API veya gRPC ile doğrudan cevap alırlar.
  - Asenkron iletişim: Kafka, RabbitMQ gibi mesaj kuyrukları üzerinden haberleşirler.
- API Gateway kullanılır. Tüm dış istekler önce bir API Gateway üzerinden alınır.
- Service Discovery kullanılır. Servisler sürekli başlatılıp durdurulabileceği için, nerede çalıştıklarını otomatik bulmak gerekir.
- Mikroservisler konfigürasyon bilgilerini (örneğin veritabanı URL’si) merkezi bir sunucudan alır.
