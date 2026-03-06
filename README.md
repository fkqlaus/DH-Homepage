# 대호이엔지 공식 홈페이지

> 실제 운영 중인 회사 공식 홈페이지입니다.  
> 기획부터 개발, 서버 구성, 배포, 운영까지 전 과정을 단독으로 수행했습니다.

**개발 기간**: 2025.06 ~ 2025.06
**개발 규모**: 1인 (단독 개발)  
**운영 여부**: 실서비스 운영 중

---

## 기술 스택

| 구분 | 기술 |
|---|---|
| Language | Java 1.8 |
| Framework | Spring Framework (MVC), JSP, JSTL |
| Server | Apache 2.x + Tomcat 8.5 (리버스 프록시 구조) |
| DB | MariaDB 11.8.2 |
| View | JSP, Tiles, Summernote 에디터 |
| Security | Let's Encrypt (HTTPS), Google reCAPTCHA v2, BCrypt |
| Infra | HamoniKR Linux, systemd, crontab, Certbot |

---

## 담당 역할 및 기여 범위

- 전체 아키텍처 설계 (Apache ↔ Tomcat 리버스 프록시 구성)
- 백엔드 API 구현 전체 (게시판, 댓글, 파일 업로드, 이메일 전송)
- HTTPS 인증서 발급 및 자동 갱신 구성 (Certbot + crontab)
- systemd 서비스 등록으로 서버 재시작 시 자동 구동
- 보안 설정 (reCAPTCHA, 해외 IP 차단, BCrypt 비밀번호 해싱)
- 디자인을 제외한 기획 ~ 운영 전 과정

---

## 주요 기능

- 회사 소개, 사업 분야, 솔루션 콘텐츠 제공
- 계층형 댓글 (등록/수정/삭제 + 비밀번호 확인)
- Summernote 에디터 기반 게시글 작성 (본문 이미지 ↔ 첨부파일 분리 저장)
- 온라인 문의 폼 → Gmail SMTP로 운영자 이메일 자동 수신
- 네이버 지도 API 연동 (방문자 위치 안내)
- Google reCAPTCHA v2 적용 (봇 방지)

---

## 기술적 도전과 해결 과정

### 1. N+1 문제 해결 — 쿼리 101회 → 2회로 감소

**문제**  
게시글 목록 10건 조회 시 쿼리가 101회 발생하는 N+1 문제가 있었습니다.  
JPA Lazy Loading 구조에서 각 게시글마다 댓글 수를 별도 쿼리로 조회하는 방식 때문이었습니다.

**해결**  
Lazy Loading을 fetch join으로 교체하고, ModelMapper를 제거한 뒤 DTO 수동 매핑을 도입했습니다.  
쿼리 횟수가 101회 → 2회로 감소했고, 응답 속도가 체감상 3~5배 향상됐습니다.

```sql
-- Before: 댓글 수 조회가 게시글마다 별도 실행 (N+1)
SELECT * FROM post;                            -- 1회
SELECT COUNT(*) FROM comment WHERE post_id=1;  -- N회 반복

-- After: 한 번의 쿼리로 처리
SELECT p.*, COUNT(c.id) as comment_count
FROM post p
LEFT JOIN comment c ON p.id = c.post_id
GROUP BY p.id;
```

---

### 2. 해외 IP 차단 기능 구현

**배경**  
공공기관 납품 시스템 특성상 외부 접근에 보안이 중요한 상황이었고,  
해외에서 들어오는 비정상적인 접근 로그가 지속적으로 발생하고 있었습니다.

**구현 방식**  
Spring Interceptor에서 요청 IP를 추출하고, KISA 공공 API로 국가 코드를 조회해 KR이 아닌 경우 접근을 차단했습니다.  
IPv6, 로컬호스트(127.0.0.1), 내부망 대역은 예외 처리했습니다.

```java
@Component
public class IpBlockInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response, Object handler) throws Exception {

        String clientIp = getClientIp(request);

        // 로컬/내부망 예외 처리
        if (isInternalIp(clientIp)) {
            return true;
        }

        String countryCode = ipInfoService.getCountryCode(clientIp);

        if (!"KR".equals(countryCode)) {
            response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access denied");
            return false;
        }

        return true;
    }
}
```

---

### 3. AOP 기반 공통 로깅 구현

**문제**  
각 컨트롤러마다 로그 코드를 개별로 작성하다 보니 중복이 심하고,  
로그 형식이 제각각이어서 운영 중 문제 추적이 어려웠습니다.

**해결**  
`@Around` 어드바이스로 모든 컨트롤러 요청과 응답 시간을 공통으로 기록했습니다.  
1,500줄 이상의 중복 로그 코드를 제거하고 단일 관리 지점으로 통합했습니다.

```java
@Aspect
@Component
@Slf4j
public class LoggingAspect {

    @Around("execution(* com.daeho..controller..*(..))")
    public Object logRequest(ProceedingJoinPoint joinPoint) throws Throwable {
        String method = joinPoint.getSignature().toShortString();
        long start = System.currentTimeMillis();

        try {
            Object result = joinPoint.proceed();
            log.info("[{}] 완료 - {}ms", method, System.currentTimeMillis() - start);
            return result;
        } catch (Exception e) {
            log.error("[{}] 예외 발생 - {}", method, e.getMessage());
            throw e;
        }
    }
}
```

---

### 4. Spring XML ↔ Java Config 공존 구조 정리

**상황**  
기존 코드는 applicationContext.xml과 Java Config가 혼재하면서  
Bean 충돌과 설정 범위 문제가 자주 발생했습니다.

**해결**  
책임을 명확히 분리했습니다.
- `applicationContext.xml` → DB, MyBatis, 트랜잭션 등 인프라 설정만 담당
- `WebMvcConfigurer` (Java) → MVC 관련 설정, ViewResolver, ComponentScan 담당

분리 이후 Bean 충돌이 사라졌고, 설정 파일을 찾는 데 드는 시간도 줄었습니다.

---

## 인프라 및 배포

Apache + Tomcat 리버스 프록시 구조를 직접 설계하고 구성했습니다.

```
사용자 요청 (80/443)
    ↓
Apache 2.x  (SSL 종료, 리버스 프록시)
    ↓
Tomcat 8.5  (Spring MVC 애플리케이션)
    ↓
MariaDB
```

- Let's Encrypt로 HTTPS 인증서 발급 및 자동 갱신 구성
- systemd 서비스 등록으로 서버 재시작 시 자동 구동
- crontab 기반 인증서 갱신 자동화 (주 1회 새벽 3시)

---

## 보안 구현

| 항목 | 구현 방식 |
|---|---|
| HTTPS | Let's Encrypt + Apache SSL 설정 |
| 봇 방지 | Google reCAPTCHA v2 |
| 해외 접근 차단 | Spring Interceptor + KISA API |
| 비밀번호 | BCrypt 해싱 |
| 입력 검증 | @Valid + 서버 측 재검증 이중 적용 |

---

## 배운 점

**운영 서비스를 직접 구성하면서 느낀 것들**

코드가 동작하는 것과 실제 서비스로 운영되는 것은 다르다는 걸 이 프로젝트에서 처음 체감했습니다.  
개발 환경에서는 문제없던 코드가 운영 서버에서 방화벽, 포트 설정, 망 구성 문제로 동작하지 않는 경우가 생겼고,  
이를 해결하면서 애플리케이션 코드 바깥의 인프라 구조를 이해하는 것이 얼마나 중요한지 알게 됐습니다.

또한 다른 사람의 코드를 유지보수하면서 주석 없이 수백 줄로 이어진 함수를 해석해야 했던 경험이,  
"내 코드를 다음에 보는 사람이 같은 어려움을 겪지 않게 하자"는 동기로 이어졌습니다.  
AOP 로깅과 설정 파일 분리 작업은 그 생각에서 시작한 작은 실천이었습니다.
