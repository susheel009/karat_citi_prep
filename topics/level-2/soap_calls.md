### Topic: SOAP Calls — Level 2

> **Related:** [Level 2 — REST APIs](./rest_apis.md) · [Level 2 — Security (SSL)](./security.md)

**Why it matters (Karat angle)**
Citi has decades-old SOAP services that aren't going away. A senior Java dev needs to call them from modern Spring Boot apps, handle WSDL-generated clients, and manage SSL certificates. Saying "I've never used SOAP" at a bank interview is a gap.

**Core concept**

| | SOAP | REST |
|--|------|------|
| Protocol | XML over HTTP/SMTP/JMS | JSON over HTTP |
| Contract | WSDL (strict, auto-generated stubs) | OpenAPI/Swagger (optional) |
| Transport | Flexible (HTTP, JMS, SMTP) | HTTP only |
| State | Can be stateful (WS-Security sessions) | Stateless |
| Error handling | SOAP Fault (standardised) | HTTP status codes |
| Use at Citi | Legacy services, enterprise integrations | New APIs, microservices |

**Calling a SOAP service from Spring Boot:**

```java
// File: topics/level-2/SoapClientDemo.java

// Step 1: Generate Java classes from WSDL
// Maven plugin:
// <plugin>
//   <groupId>org.jvnet.jaxb2.maven2</groupId>
//   <artifactId>maven-jaxb2-plugin</artifactId>
//   <configuration>
//     <schemaLanguage>WSDL</schemaLanguage>
//     <schemas>
//       <schema><url>https://legacy.citi.com/payments?wsdl</url></schema>
//     </schemas>
//   </configuration>
// </plugin>
// Generates: PaymentService, PaymentRequest, PaymentResponse, ObjectFactory

// Step 2: Create a SOAP client using WebServiceGatewaySupport
@Component
public class PaymentSoapClient extends WebServiceGatewaySupport {

    public PaymentResponse processPayment(String accountId, BigDecimal amount) {
        PaymentRequest request = new PaymentRequest();
        request.setAccountId(accountId);
        request.setAmount(amount);

        PaymentResponse response = (PaymentResponse) getWebServiceTemplate()
            .marshalSendAndReceive(
                "https://legacy.citi.com/payments",
                request,
                new SoapActionCallback("processPayment")
            );

        return response;
    }
}

// Step 3: Configure the client
@Configuration
public class SoapConfig {

    @Bean
    public Jaxb2Marshaller marshaller() {
        Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
        marshaller.setContextPath("com.citi.generated.payments");  // package of generated classes
        return marshaller;
    }

    @Bean
    public PaymentSoapClient paymentClient(Jaxb2Marshaller marshaller) {
        PaymentSoapClient client = new PaymentSoapClient();
        client.setDefaultUri("https://legacy.citi.com/payments");
        client.setMarshaller(marshaller);
        client.setUnmarshaller(marshaller);
        return client;
    }
}
```

**SSL certificate handling for SOAP calls:**

```java
// File: topics/level-2/SoapSslDemo.java

@Configuration
public class SoapSslConfig {

    @Bean
    public HttpsUrlConnectionMessageSender messageSender() throws Exception {
        HttpsUrlConnectionMessageSender sender = new HttpsUrlConnectionMessageSender();

        // Load truststore (for verifying server's cert)
        KeyStore trustStore = KeyStore.getInstance("PKCS12");
        trustStore.load(new FileInputStream("truststore.p12"), "changeit".toCharArray());

        TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        tmf.init(trustStore);

        // Load keystore (for mutual TLS — client cert)
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        keyStore.load(new FileInputStream("client-keystore.p12"), "secret".toCharArray());

        KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        kmf.init(keyStore, "secret".toCharArray());

        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(kmf.getKeyManagers(), tmf.getTrustManagers(), null);

        sender.setSslContext(sslContext);
        return sender;
    }

    @Bean
    public PaymentSoapClient paymentClient(Jaxb2Marshaller marshaller,
                                            HttpsUrlConnectionMessageSender sender) {
        PaymentSoapClient client = new PaymentSoapClient();
        client.setDefaultUri("https://legacy.citi.com/payments");
        client.setMarshaller(marshaller);
        client.setUnmarshaller(marshaller);
        client.setMessageSender(sender);                  // use SSL-configured sender
        return client;
    }
}
```

**Error handling — SOAP Faults:**

```java
// File: topics/level-2/SoapFaultHandling.java

try {
    PaymentResponse response = paymentClient.processPayment("A001", BigDecimal.TEN);
} catch (SoapFaultClientException ex) {
    // Client-side fault (bad request)
    String faultCode = ex.getSoapFault().getFaultCode().getLocalPart();
    String faultMessage = ex.getSoapFault().getFaultStringOrReason();
    throw new PaymentException("SOAP_CLIENT_FAULT", faultMessage);
} catch (WebServiceIOException ex) {
    // Network issue
    throw new PaymentException("SOAP_NETWORK_ERROR", "SOAP service unreachable", ex);
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** SOAP is an XML-based protocol with strict contracts (WSDL). Spring's `WebServiceTemplate` calls SOAP services using JAXB-generated classes for marshalling/unmarshalling.
2. **Why/when:** Legacy Citi services expose SOAP — they're not being rewritten. New services call them via generated stubs and handle SOAP Faults, SSL/mutual TLS, and response mapping.
3. **Example:** Generate Java classes from WSDL (Maven JAXB plugin), configure `Jaxb2Marshaller`, create a client extending `WebServiceGatewaySupport`, call `marshalSendAndReceive()`.
4. **Gotcha/tradeoff:** SOAP services often require mutual TLS (client cert + server cert). You need both a keystore (your cert) and a truststore (their cert). Certificate rotation breaks clients if not managed.

**Common pitfalls**
- Not configuring SSL properly — `javax.net.ssl.SSLHandshakeException` on every call.
- Forgetting to regenerate stubs when the WSDL changes — runtime marshalling errors.
- Blocking the calling thread — SOAP calls are synchronous; use `@Async` or virtual threads for high-throughput.
- Hardcoding SOAP endpoints — use environment-specific properties.

**Self-check question**
A SOAP service requires mutual TLS. You have the server's CA cert and your client cert. Where do each go (keystore vs truststore), and what happens if you swap them?
