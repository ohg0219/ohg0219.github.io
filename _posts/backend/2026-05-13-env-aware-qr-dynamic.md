---
title: "환경별 QR 을 부팅 시 동적 생성"
description: "PNG 3장을 ZXing + @PostConstruct 로"
date: 2026-05-13 14:42:00 +0900
categories: [backend]
tags: [spring-boot, qr-code, zxing, post-construct, environment-config]
image:
  path: /assets/img/posts/env-aware-qr-dynamic/cover.jpg
  alt: Spring Boot 부팅 시 환경별 QR 코드를 동적으로 생성하고 짧은 URL 로 리다이렉트하는 흐름을 표현한 기술 커버 이미지
---

메일에 박혀 나가는 QR 코드 한 장을 바꾸기가 이렇게 번거로울 일인가 — 그게 시작이었습니다.

기존에는 QR 이미지를 그냥 정적 PNG 한 장으로 들고 있었습니다. `src/main/resources/static/images/mail/mail_qr.png`. 메일 템플릿이 `<img src="/images/mail/mail_qr.png">` 로 이 파일을 참조하고, 사용자는 메일 안의 그 이미지를 휴대폰으로 찍어 안내 페이지로 들어옵니다. 단순하고, 일년에 한두 번 건드릴 일도 없을 것 같은 파일이었습니다.

문제는 그 "한두 번" 이 닥쳤을 때였습니다.

## PNG 3장을 들고 다녔다

QR 이 가리키는 URL 은 환경마다 다릅니다. 각 환경이 다른 도메인을 쓰고, 안내 페이지의 실제 위치도 환경마다 갈립니다. 정적 PNG 1장으로는 한 환경밖에 못 담으니, 결국 **환경 수만큼 PNG 를 따로 만들어 두고 빌드 프로파일에 맞춰 골라 쓰는** 식으로 굴러가야 하는 구조였습니다.

여기서 끈적한 게 두 가지 더 붙습니다.

1. **외부 도구 의존성**: QR 안의 URL 이 바뀌면 매번 외부 QR 생성기에 URL 을 넣어 PNG 를 다시 받아 커밋해야 합니다. URL 한 글자가 달라져도 마찬가지.
2. **테스트 불가**: 빌드되어 나간 PNG 가 실제로 어느 URL 을 인코딩하고 있는지를 코드 차원에서 검증할 방법이 없었습니다. 휴대폰으로 찍어보기 전까지는 모릅니다.

QR 한 장이 환경 분리 · 변경 · 검증 세 가지를 동시에 막고 있었습니다.

## 부팅할 때 한 번만 만들면 충분했다

해법은 단순했습니다. **정적 PNG 를 지우고, 부팅 시점에 ZXing 으로 한 번 인코딩한 다음 메모리에 들고 있자.**

ZXing 의존성을 `pom.xml` 에 추가하고:

```xml
<dependency>
    <groupId>com.google.zxing</groupId>
    <artifactId>core</artifactId>
    <version>3.5.3</version>
</dependency>
<dependency>
    <groupId>com.google.zxing</groupId>
    <artifactId>javase</artifactId>
    <version>3.5.3</version>
</dependency>
```

PNG 바이트만 뽑는 얇은 유틸을 둡니다:

```java
public class QrCodeGenerator {

    public static byte[] generatePng(String content, int size) throws WriterException, IOException {
        QRCodeWriter writer = new QRCodeWriter();
        BitMatrix bitMatrix = writer.encode(content, BarcodeFormat.QR_CODE, size, size,
                Map.of(EncodeHintType.MARGIN, 1));

        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        MatrixToImageWriter.writeToStream(bitMatrix, "PNG", outputStream);
        return outputStream.toByteArray();
    }
}
```

컨트롤러는 `@PostConstruct` 에서 단 한 번 QR 을 만들어 `byte[]` 로 캐시하고, `/images/mail/mail_qr.png` 요청이 오면 그 바이트를 그대로 응답합니다:

```java
@Slf4j
@Controller
@RequiredArgsConstructor
public class QrCodeController {

    private final AppProperty           appProperty;
    private final RedirectUrlProperty redirectUrlProperty;
    private       byte[]                  cachedQrImage;

    @PostConstruct
    public void init() {
        try {
            String targetUrl = appProperty.getDomain() + "/qr/mail";
            cachedQrImage = QrCodeGenerator.generatePng(targetUrl, 300);
            log.info("Mail QR code generated successfully for URL: {}", targetUrl);
        } catch (Exception e) {
            log.error("Failed to generate mail QR code", e);
        }
    }

    @GetMapping(value = "/images/mail/mail_qr.png", produces = MediaType.IMAGE_PNG_VALUE)
    public ResponseEntity<byte[]> getMailQrCode() {
        if (cachedQrImage == null) {
            return ResponseEntity.internalServerError().build();
        }
        return ResponseEntity.ok()
                             .contentType(MediaType.IMAGE_PNG)
                             .body(cachedQrImage);
    }
}
```

여기서 환경 분리 문제는 그냥 사라집니다. `appProperty.getDomain()` 은 `application-{env}.yml` 에서 환경별로 주입되니까, **코드는 그대로 둔 채 yml 만 다르면 환경마다 다른 QR 이 자동으로 만들어집니다.** 빌드해서 들고 다닐 PNG 가 없어졌고, QR 안의 URL 도 로그 한 줄로 확인됩니다 (`Mail QR code generated successfully for URL: ...`).

`@PostConstruct` 안에서 try-catch 로 감싼 건 의도적입니다 — QR 한 장 인코딩이 실패한다고 앱 부팅 자체를 막을 수는 없으니, 실패하면 `cachedQrImage` 가 null 로 남고 해당 엔드포인트만 500 을 돌려주도록 했습니다.

## 긴 URL 을 그대로 박을 수는 없었다

여기까지였으면 그냥 평이한 리팩토링인데, 한 가지 결정이 더 들어갔습니다. **QR 에 박는 URL 은 최종 목적지가 아니라 우리 도메인의 짧은 자체 경로(`/qr/mail`) 입니다.** 그리고 그 경로에 도달하면 서버가 진짜 가야 할 곳으로 302 리다이렉트를 칩니다.

```java
@GetMapping("/qr/mail")
public String redirectMailQr() {
    String redirectUrl = "redirect:" + redirectUrlProperty.getMail().getTargetUrl();
    log.info("Redirect URL: {}", redirectUrl);
    return redirectUrl;
}
```

왜 굳이 한 단계를 더 끼웠을까요. **최종 URL 이 너무 길었습니다.**

QR 코드는 인코딩할 문자열이 길어질수록 표현해야 할 비트 수가 늘어나고, 그만큼 모듈(셀) 수가 늘어납니다. 같은 물리 크기 안에서 모듈 수가 늘어나면 셀 하나하나가 작아져 패턴이 빽빽해지고, 인쇄·캡처·휴대폰 스캔 환경에서 인식 안정성이 떨어집니다. 메일 본문이라는 작은 표시 공간에 들어가야 하는데, 긴 URL 을 그대로 박으면 점 무더기처럼 빽빽한 QR 이 나옵니다.

해결은 단순했습니다. **QR 에는 짧은 자체 URL 만 새기고, 길이는 우리가 통제한다.** `{domain}/qr/mail` 은 길어봐야 수십 자 수준이라 QR 패턴 자체가 깔끔합니다. 실제 긴 URL 은 서버 안에서만 살고, 클라이언트(스캐너)는 우리 도메인의 짧은 경로만 본 뒤 리다이렉트를 따라갑니다.

부수 효과로 한 가지가 더 붙는데, 이건 의도라기보단 보너스에 가깝습니다 — 실제 목적지 URL 이 바뀌어도(파일이 다른 위치로 옮겨가거나, 외부 호스팅 키가 바뀌어도) 서버 설정만 갈아끼우면 끝입니다. 이미 인쇄되거나 캡처돼 돌아다니는 QR 은 우리 도메인의 짧은 경로만 가리키고 있으니 그대로 살아 있습니다.

## 남는 것

- **"정적 자원 한 장" 이 환경 분리·변경·테스트 세 가지를 동시에 막고 있다면, 그건 자원이 정적이라서가 아니라 자원이 잘못된 추상화 레벨에 있다는 신호입니다.** QR 의 내용물은 빌드 산출물이 아니라 런타임 설정의 일부였고, 그 자리로 옮기니 문제 셋이 한꺼번에 사라졌습니다.
- **QR 의 가독성은 표시 크기뿐 아니라 인코딩 문자열 길이에 비례합니다.** 긴 URL 을 그대로 박지 말고, 우리가 통제할 수 있는 짧은 자체 경로 + 서버 리다이렉트로 한 단계 끼우면 QR 패턴이 깔끔해지고, 부수적으로 인쇄된 QR 의 수명도 길어집니다.
- **`@PostConstruct` + 바이트 캐시는 "부팅 후 안 바뀌는 작은 산출물" 을 다루는 데 충분히 가벼운 패턴입니다.** 매 요청마다 인코딩할 이유가 없고, 메모리 수 KB 면 됩니다. 단, 실패가 부팅을 막으면 안 되는 종류의 부수 자원이라면 try-catch 로 감싸 null 캐시 + 엔드포인트 단위 500 으로 격리하는 게 안전합니다.
