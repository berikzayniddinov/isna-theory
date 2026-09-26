# 75. ЭЦП Kalkan (казахстанская электронно-цифровая подпись)

Как работает ЭЦП в Казахстане. Kalkan library. NCALayer. Использование в ИСНА.

---

## 1. Что такое ЭЦП

**ЭЦП** = **Электронно-Цифровая Подпись** = **electronic digital signature**.

Аналог рукописной подписи, но для электронных документов:
- **Подтверждает автора**.
- **Гарантирует целостность** — документ не изменён после подписи.
- **Non-repudiation** — автор не может отказаться от факта подписи.

Основа — **асимметричная криптография** (RSA / GOST / ECDSA).

---

## 2. Базовые концепты

### 2.1 Ключевая пара

- **Private key** — секретный, только у владельца.
- **Public key** — открытый, все могут получить.

Свойства:
- Данные зашифрованные **public** → расшифровываются только **private**.
- Данные подписанные **private** → проверяются **public**.

### 2.2 Как подпись работает

```
Document
   │
   ▼
Hash (SHA-256 / GOST)
   │
   ▼
Encrypted with Private Key = Signature
   │
   ▼
Attached to Document
```

**Verify**:
```
Signature + Public Key → decrypted → Hash1
Document → Hash → Hash2
Hash1 == Hash2 → valid
```

Если документ изменён — hashes не совпадут.
Если signature от другого private key — не расшифруется public'ом владельца.

### 2.3 Certificate (X.509)

Public key + метаданные (имя владельца, срок действия, issuer), подписан **CA (Certificate Authority)**.

Формат X.509 стандарт:
```
Subject: CN=BERIK CHOTAM, IIN=123456789012, ...
Issuer: НУЦ РК (National Certification Authority of RK)
Valid: 2026-01-01 to 2027-01-01
Public Key: 30820122300D06092A864886F70D...
Signature: ...
```

При проверке signature — проверяется и cert (не отозван, не истёк, cert path до trusted CA).

### 2.4 Chain of trust

```
Root CA (self-signed)
    ↓
Intermediate CA (signed by Root)
    ↓
User certificate (signed by Intermediate)
```

Verifier доверяет Root CA → доверяет всей цепочке.

---

## 3. НУЦ РК

**НУЦ РК** = **Национальный Удостоверяющий Центр Республики Казахстан**.

- Государственный CA.
- Выдаёт сертификаты гражданам и юр. лицам.
- Root of trust для всех гос-систем.

Типы сертификатов:
- **AUTH** — для аутентификации (login на сайты).
- **SIGN (RSA)** — для подписи документов.
- **GOST** — на алгоритмах ГОСТ (совместимо с РФ/CIS).

Каждый гражданин РК может получить бесплатно.

---

## 4. Kalkan

**Kalkan** — казахстанская crypto library.

- Реализация ГОСТ + RSA алгоритмов.
- **JCE Provider** — интегрируется с Java стандартным API (`java.security`).
- Разработан НИТ (Национальные Информационные Технологии).
- Distributed jar-файл + native libraries.

Иногда называют **KalkanCrypt**.

### 4.1 Зачем нужен

Java стандарт имеет RSA, но **не ГОСТ** (Российские стандарты).

Казахстанские сертификаты — ГОСТ. Отсюда нужен Kalkan для их поддержки.

### 4.2 Установка

Не в Maven Central. Отдельная лицензия для организаций.

Обычно:
```
kalkan-crypt.jar
libkalkan.so / .dll (native)
```

Registration в Java:
```java
Security.addProvider(new KalkanProvider());
```

---

## 5. Форматы подписей

### 5.1 PKCS#7 / CMS

**PKCS#7** (RFC 2315) → эволюция **CMS** (Cryptographic Message Syntax, RFC 5652).

Универсальный формат:
- **SignedData** — данные + подписи.
- **EnvelopedData** — зашифрованные данные.
- **DigestedData** — данные + hash.

Пример PKCS#7 подписи:
```
SignedData {
    version: 1
    digestAlgorithms: [ SHA-256 ]
    contentInfo: {
        contentType: data
        content: <XML/PDF/other>
    }
    certificates: [ user cert, CA cert ]
    signerInfos: [
        {
            signerIdentifier: cert serial,
            digestAlgorithm: SHA-256,
            signature: <encrypted hash>
        }
    ]
}
```

Часто в base64.

### 5.2 XML Digital Signature (XMLDSig)

XML-специфичный формат — signature встроена в XML:
```xml
<Envelope>
  <Body>...</Body>
  <Signature xmlns="http://www.w3.org/2000/09/xmldsig#">
    <SignedInfo>
      <CanonicalizationMethod Algorithm="..."/>
      <SignatureMethod Algorithm="..."/>
      <Reference URI="">
        <DigestMethod Algorithm="..."/>
        <DigestValue>...</DigestValue>
      </Reference>
    </SignedInfo>
    <SignatureValue>...</SignatureValue>
    <KeyInfo>
      <X509Data><X509Certificate>...</X509Certificate></X509Data>
    </KeyInfo>
  </Signature>
</Envelope>
```

Используется в **WS-Security** для SOAP messages.

### 5.3 PDF signature

PDF имеет встроенный формат signature (PDF spec ISO 32000).

### 5.4 CAdES / XAdES

**Advanced Electronic Signatures** — расширения PKCS#7 / XMLDSig:
- **CAdES-B** — базовая (только signature).
- **CAdES-T** — + timestamp.
- **CAdES-LT** — + long-term validation (chain + CRL/OCSP embedded).
- **CAdES-A** — archival (для долгосрочного хранения).

Для legal / compliance важны T+ уровни (можно проверить и через N лет).

---

## 6. NCALayer

**NCALayer** — desktop application для работы с ЭЦП в браузере.

### 6.1 Проблема

Браузер не может напрямую:
- Читать USB-токен.
- Работать с private key.
- Вызывать OS-специфичные crypto libraries.

### 6.2 Решение

NCALayer работает **как локальный сервер** (WebSocket на порту 13579):
- Установлен у пользователя.
- Общается с USB-токенами (JavaCard, eToken).
- Общается с ключами в файле (.p12).
- Работает с настоящим Kalkan провайдером.

### 6.3 Flow

```
Browser (JavaScript)
    │  WebSocket
    ▼
NCALayer (localhost:13579)
    │
    ├─ ← USB token (P11 driver, JavaCard)
    ├─ ← File key (PKCS#12 .p12)
    └─ ← Sign with Kalkan
```

Пользователь:
1. Открывает сайт КНП.
2. Нажимает «Подписать ЭЦП».
3. Frontend через WebSocket шлёт документ в NCALayer.
4. NCALayer показывает диалог "введите PIN".
5. Подписывает.
6. Возвращает signature JavaScript'у.
7. JavaScript отправляет в backend.

### 6.4 Установка NCALayer

Пользователь качает с сайта НУЦ РК. Windows / Mac / Linux версии.

Требования: Java Runtime, драйверы токенов.

---

## 7. Где ЭЦП используется в ИСНА

### 7.1 Подача ФНО

Налогоплательщик заполняет ФНО в кабинете → подписывает ЭЦП → отправка.

Backend проверяет:
- Signature валидна.
- Cert не истёк.
- Владелец = автор (совпадает IIN).

### 7.2 Обращения

Отправка обращений в НАП, налоговую → ЭЦП.

### 7.3 Платежи

Согласие на списание → ЭЦП владельца счёта.

### 7.4 Официальные ответы

Инспектор подписывает ответы налогоплательщикам → ЭЦП инспектора.

### 7.5 Из memory

- **`knp-e2e-fno-java21-prod-smoke-main-contour`** — упоминается `-MyEds` (личный ЭЦП), **`BBPartner`** — тестовые ключи. Для FL: `-MyEds` личный, НЕ BBPartner (там FL externalProfile=true → 404).
- **`knp-cameral-explanation-regnumber-gate`** — упоминается FAMILIAR (что-то связанное с familiar-ключом).

---

## 8. Java-интеграция

### 8.1 Kalkan Provider

```java
Security.addProvider(new KalkanProvider());

// Load key from PKCS#12 file
KeyStore ks = KeyStore.getInstance("PKCS12", "KALKAN");
try (FileInputStream fis = new FileInputStream("user.p12")) {
    ks.load(fis, "password".toCharArray());
}

PrivateKey privateKey = (PrivateKey) ks.getKey(alias, "password".toCharArray());
X509Certificate cert = (X509Certificate) ks.getCertificate(alias);
```

### 8.2 Sign PKCS#7

```java
Signature signature = Signature.getInstance("SHA256withRSA", "KALKAN");
signature.initSign(privateKey);
signature.update(data);
byte[] signed = signature.sign();
```

### 8.3 Verify

```java
Signature verify = Signature.getInstance("SHA256withRSA", "KALKAN");
verify.initVerify(cert.getPublicKey());
verify.update(data);
boolean valid = verify.verify(signed);
```

### 8.4 Check cert validity

```java
cert.checkValidity();   // throws if expired

// проверка chain
CertPathValidator validator = CertPathValidator.getInstance("PKIX");
PKIXParameters params = new PKIXParameters(trustStore);
params.setRevocationEnabled(true);
CertPathValidatorResult result = validator.validate(certPath, params);
```

### 8.5 XMLDSig

```java
XMLSignatureFactory factory = XMLSignatureFactory.getInstance("DOM");
Reference ref = factory.newReference("", ...);
SignedInfo si = factory.newSignedInfo(...);

DOMSignContext ctx = new DOMSignContext(privateKey, xmlDoc.getDocumentElement());
XMLSignature signature = factory.newXMLSignature(si, keyInfo);
signature.sign(ctx);
```

---

## 9. Тестовые ключи

Для разработки — не использовать реальные ЭЦП.

Тестовые сертификаты:
- **BBPartner** — упоминаемые в memory ИСНА (для юр. лиц).
- **MyEds** — личный ЭЦП тестового пользователя.

Обычно генерируются на dev-CA. Не проходят проверку в проде (не подписаны real НУЦ РК).

---

## 10. Типовые проблемы

### 10.1 Cert expired

Cert имеет expiration date (обычно 1 год).

Symptom: подпись валидна крипто, но `checkValidity()` throws.

Fix: обновить cert в НУЦ РК.

### 10.2 Cert revoked

Cert может быть отозван (compromised, lost).

Verification:
- **CRL** (Certificate Revocation List) — список отозванных.
- **OCSP** — real-time проверка.

Kalkan / bouncy castle умеют.

### 10.3 Cert chain broken

Не найден intermediate CA. Verification fails.

Fix: убедиться что trust store содержит all NUC RK intermediates.

### 10.4 Signature vs canonical form

XML имеет проблему **whitespace**: `<a>x</a>` и `<a>  x  </a>` — разные для signature.

**Canonicalization** — приведение к каноническому виду перед hash.

Алгоритмы:
- **C14N** — Canonical XML 1.0.
- **Exclusive C14N** — для namespace handling.

Ошибка в canonicalization → signature не валидна.

### 10.5 Timezone на подписи

Timestamp на подписи в UTC или local? Разные системы могут интерпретировать по-разному.

### 10.6 Encoding

XML text — UTF-8. Bytes → hash → signature.

Если файл в другой encoding (Windows-1251) → hash другой → verification fails.

### 10.7 Big files

RSA sign медленный на больших данных. Правильно: sign **hash**, не сами данные.

Ключевой размер: RSA 2048 или 4096 bit — норма.

### 10.8 Kalkan native libraries

Kalkan использует native lib (`.so` / `.dll`) для performance. Нужны для JVM.

Deployment: включать в Docker image, устанавливать в правильный path.

---

## 11. Integration с Spring / Keycloak

### 11.1 Login через ЭЦП

Стандартный Keycloak не поддерживает ЭЦП из коробки. Нужен **custom Authenticator SPI**.

Flow:
1. Пользователь на login page.
2. Frontend показывает "Sign with ЭЦП" button.
3. Через NCALayer — sign challenge (случайная строка).
4. Frontend шлёт signed challenge на Keycloak.
5. Custom Authenticator валидирует signature + cert.
6. Извлекает IIN → mapping на пользователя.
7. Keycloak выдаёт JWT.

### 11.2 Backend валидация

Отдельный сервис проверяет ЭЦП полученные документы (ФНО, обращения).

```java
@Service
class KalkanSignatureVerifier {

    public VerifyResult verify(byte[] document, byte[] signature) {
        try {
            CMSSignedData cms = new CMSSignedData(
                new CMSProcessableByteArray(document), signature);

            for (SignerInformation signer : cms.getSignerInfos().getSigners()) {
                X509CertificateHolder certHolder = getCert(signer);
                cert.checkValidity();

                boolean valid = signer.verify(new JcaSimpleSignerInfoVerifierBuilder()
                    .setProvider("KALKAN")
                    .build(certHolder));

                if (!valid) return VerifyResult.failed("bad signature");

                // Проверить cert chain
                validateChain(certHolder);
            }

            return VerifyResult.success();
        } catch (Exception e) {
            return VerifyResult.failed(e.getMessage());
        }
    }
}
```

---

## 12. Best practices

1. **Sign hash**, не данные (для больших файлов).
2. **CAdES-T или выше** для legal compliance (timestamp).
3. **CRL / OCSP** для revocation check.
4. **Правильные canonicalization** для XML.
5. **UTC для timestamps**.
6. **Явные encodings** (UTF-8 для XML).
7. **Не хранить private keys** в приложении (только на USB-токенах у пользователей).
8. **Trust store** актуальный (все НУЦ РК intermediates).
9. **Testing** через тестовые ЭЦП (BBPartner, dev-CA).
10. **NCALayer** установка у пользователей — часть UX.

---

## 13. Собесные вопросы

1. **Что такое ЭЦП?** — Электронно-цифровая подпись; аналог рукописной для электронных документов.
2. **Основа?** — Асимметричная криптография (private/public key).
3. **Как подписать?** — Hash документа → encrypt hash с private key = signature.
4. **Как проверить?** — Decrypt signature с public key → сравнить с hash документа.
5. **Что такое сертификат X.509?** — Public key + метаданные + подпись CA.
6. **НУЦ РК?** — Национальный Удостоверяющий Центр РК; государственный CA.
7. **Что такое Kalkan?** — Казахстанская crypto library; ГОСТ + RSA; JCE Provider для Java.
8. **PKCS#7 / CMS?** — Стандартный формат подписи (SignedData).
9. **XMLDSig?** — XML-specific формат signature (в WS-Security).
10. **CAdES vs XAdES?** — CAdES: advanced signature на PKCS#7; XAdES на XMLDSig.
11. **Что такое NCALayer?** — Desktop app как мост между browser и USB-токенами.
12. **Как проверить cert revoked?** — CRL или OCSP.
13. **Canonicalization XML?** — Приведение к каноническому виду для стабильного hash.
14. **Chain of trust?** — Root CA → Intermediate → User cert; всё проверяется до trusted root.
15. **Где ЭЦП в ИСНА?** — ФНО, обращения, платежи, ответы инспекторов.

---

## Итог

- **ЭЦП** = digital signature на асимметричной криптографии.
- **НУЦ РК** — государственный CA.
- **Kalkan** — Java crypto library с ГОСТ + RSA.
- **NCALayer** — desktop app для работы с USB-токенами через browser.
- Форматы: **PKCS#7 / CMS**, **XMLDSig**, **PDF signature**.
- **CAdES-T+** для long-term validation.
- В **ИСНА**: ЭЦП обязательна для подачи ФНО, обращений, платежей.
- Тестовые ключи: **BBPartner** (юр.лицо), **MyEds** (личный).

---

## Финальный итог блока 72-75

- 72 — Redis (in-memory KV, богатые structures)
- 73 — Hazelcast (Java-native distributed grid, в ИСНА)
- 74 — Разница Redis vs Hazelcast (когда что)
- 75 — ЭЦП Kalkan (казахстанская крипто, NCALayer, ИСНА)

**Итого 75 файлов** в `isna-theory\`.

Дальше могу: **CAP теорема / distributed systems теория**, **алгоритмы для собесов**, **system design**, **GitLab CI углубленно**. Что берём?
