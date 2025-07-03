# 대호이엔지 홈페이지 운영 구성 정보

> Spring + JSP 기반 웹 애플리케이션으로, Tomcat 8.5와 Apache를 연동하여 HTTPS 접속이 가능한 실서비스용 홈페이지 프로젝트입니다.\
> Git 저장소, 서버 경로, DB 정보, SSL 인증서 구성 및 자동 갱신 스케줄 등을 정리하였습니다.

---
## 👨‍💻 개발자 정보

> 본 프로젝트는 **최광혁**이 기획 및 개발하였습니다.  
> **실제 회사에서 운영 중인 공식 홈페이지 프로젝트입니다.**  
> 디자인을 제외한 전체 아키텍처 설계, 백엔드 API 구현, 서버 구성, 배포 및 운영까지 **전 과정을 수행**하였습니다.


- Email: fkqlaus@daehoeng.com || fkqlaus@naver.com


---

## 🌐 서버 정보

- **원격 접속 IP**: `[비공개]`
- **접속 계정**: ID: `[비공개]`, PW: `[비공개]`
- **OS**: `Linux 기반 HamoniKR`
- **Git 저장소**: `[비공개]`

---

## 🗄️ 개발 환경 및 경로

- **프로젝트 패치 경로**: `/home/daeho/homepage/daehoeng`
- **첨부파일 업로드 경로**: `/home/daeho/homepage/upload`
- **시스템 서비스 등록**: `/etc/systemd/system/DH-Homepage.service`

---

## ⚙️ WAS 및 웹서버 정보

- **Apache 포트**: 80 (HTTP), 443 (HTTPS)
- **Tomcat Connector 포트**: 8080
- **Tomcat Shutdown 포트**: 8005
- **Java 버전**: 1.8
- **Tomcat 버전**: 8.5.99

---

## 🛢️ 개발서버 DB 정보 (MariaDB)

- **DB 버전**: 11.8.2
- **내부 IP**: `[비공개]`
- **외부 IP**: `[비공개]`
- **DB 이름**: `dh-homepage`
- **계정**: ID: `[비공개]`, PW: `[비공개]`

---

## 🔐 SSL 인증서 구성 정보

- **인증서 발급 도구**: Let's Encrypt (Certbot)
- **인증 도메인**: `daehoeng.com`, `www.daehoeng.com`
- **인증서 경로**:
  - `/etc/letsencrypt/live/daehoeng.com/fullchain.pem`
  - `/etc/letsencrypt/live/daehoeng.com/privkey.pem`
- **Apache SSL 설정 파일**: `/etc/apache2/sites-available/daehoeng-ssl.conf`
- **HTTPS 포트**: 443
- **Apache 모듈**: `ssl`, `proxy`, `proxy_http` 활성화
- **최초 발급 명령어**:
  ```bash
  sudo certbot certonly --standalone -d daehoeng.com -d www.daehoeng.com
  ```

---

## 🔄 인증서 자동 갱신 설정

- **갱신 주기**: 매주 일요일 새벽 3시
- **갱신 방식**: standalone (Apache 일시 중지)
- **Crontab 등록 위치**: `sudo crontab -e`
- **등록 명령어**:
  ```bash
  0 3 * * 0 systemctl stop apache2 && certbot renew --quiet && systemctl start apache2
  ```
- **로그 파일 위치**:
  - Certbot: `/var/log/letsencrypt/letsencrypt.log`
  - Apache:
    - Access: `/var/log/apache2/daehoeng-ssl-access.log`
    - Error: `/var/log/apache2/daehoeng-ssl-error.log`
- **테스트 명령어**:
  ```bash
  sudo systemctl stop apache2
  sudo certbot renew --force-renewal
  sudo systemctl start apache2
  ```

---

## 📝 유지보수 메모

- 인증서 유효기간: 90일 → 자동 갱신 구성 완료
- Apache 끄지 않고 갱신하려면 `--webroot` 방식으로 전환 가능
- Crontab 주석:
  ```cron
  # [Purpose] Homepage SSL certificate auto-renew
  # [Author] choi gwanghyeok
  # [Date] 2025-07-02
  ```

---

## ✅ 구성 효과

- 안전한 HTTPS 서비스 운영
- 인증서 만료 방지 → 인증서로 인한 서비스 중단 X


---

## 📦 기술 스택 요약

```
Java 1.8  
Apache 2.x  
Tomcat 8.5.99  
MariaDB 11.8.2  
Let's Encrypt (Certbot)  
Ubuntu (HamoniKR 기반)  
systemctl / crontab / bash
```

---

## 🌟 주요 기능 소개


### 사용자 기능

- 회사 소개 및 솔루션, 사업분야, 인사말 등 콘텐츠 제공
- 게시판(공지사항) 목록 및 상세 조회
- 이미지 및 첨부파일 포함 게시글 열람
- 게시글 작성, 수정, 삭제 (첨부파일 포함)
- 계층형 댓글 시스템 (등록, 수정, 삭제 포함)
- 썸머노트 에디터 기반 본문 작성 및 이미지 삽입
- 본문 이미지와 첨부파일 분리 저장 및 관리
- 게시글/댓글/파일 처리 시 비밀번호 확인 기능
- 온라인 문의 폼 제공 (이메일 전송 기능 포함)

### 보안 및 운영

- HTTPS 적용 (Let's Encrypt)
- 인증서 자동 갱신 구성
- Apache ↔ Tomcat 연동 (리버스 프록시 구조)
- 시스템 서비스 등록을 통한 Tomcat 자동 실행

---

## 🧩 실서비스 기술적 기여 요약

### ✅ N+1 문제 해결 및 API 성능 최적화

- JPA Lazy Loading → fetch join으로 최적화
- ModelMapper 제거, DTO 수동 매핑 도입
- 게시글 10건 → 쿼리 2건으로 감소 (101 → 2회)
- 성능 체감 3\~5배 향상 + 테스트 용이성 증가

### ✅ 해외 IP 차단 기능 구현

- Spring Interceptor + 공공 KISA API 연동
- IP에서 국가 코드 조회 후 KR이 아닐 경우 차단
- IPv6, 127.0.0.1 등은 예외 처리

### ✅ Spring XML ↔ Java Config 공존 구조 개선

- WebMvcConfigurer 사용, ComponentScan 명시화
- ApplicationContext.xml은 DB 중심으로 책임 분리

### ✅ AOP 기반 로깅 구현

- @Around 기반 모든 Controller 요청 기록
- 텍스트 기반 로그 저장 → 운영 중 실시간 추적 가능

### ✅ 사용자 보안 강화

- Google reCAPTCHA v2 적용
- 댓글/게시글 수정·삭제 시 비밀번호 확인 기능
- @Valid + 서버 유효성 재검증 적용
- bot 방지 및 CSRF 공격 예방

### ✅ 이메일 전송 기능

- JavaMailSender + Gmail SMTP 연동
- 문의 시 운영자 이메일로 자동 수신됨
- 발신자 인증 → Gmail 계정 사용
- 문의 폼을 통한 사용자 편의성 제공

### ✅ 네이버 지도 API 연동

- 방문자용 위치 안내 지도 삽입 (스크립트 기반)
- 모바일 대응 반응형 지도 출력

### ✅ WAR 배포 자동화 및 서비스 등록

- systemd 등록 → 시스템 재부팅 시 자동 실행
- status/restart/logs 등 명령으로 운영 제어

---
## 🔐 보안 및 인증

- Google reCAPTCHA v2를 적용하여 비정상적인 폼 요청(댓글, 게시글 작성 등)에 대한 봇 방지 기능 구현
- 댓글/게시글 수정 및 삭제 시 비밀번호 확인 절차 추가
- 비밀번호 DB 저장 시 BCrypt 해싱 알고리즘 사용
- 서버 간 통신 및 사용자 접근은 모두 HTTPS(443 포트) 기반
- 외부 API를 사용한 해외 IP 차단 기능 구현
  - KISA의 공공 API를 통해 IP의 국가 코드를 조회하고, 한국(KR)이 아닌 경우 접근 차단


---

## 📋 로깅 및 AOP

- AOP(Aspect-Oriented Programming)를 활용하여 요청 로그, 예외 로그 등을 파일로 자동 기록
- `@Around` 어노테이션 기반으로 Controller 단에 로그 인터셉트 구현
- 로그 파일은 텍스트 기반으로 서버 내 디렉토리에 저장되며, 운영 중 모니터링 가능

---

## 📦 상세 기술 스택

| 구분     | 사용 기술                                     |
| ------ | ----------------------------------------- |
| 언어     | Java 1.8                                  |
| 프레임워크  | Spring Framework (MVC), JSP, JSTL         |
| 서버     | Tomcat 8.5.99, Apache 2.x                 |
| 보안     | Let's Encrypt (Certbot), HTTPS, reCAPTCHA |
| DB     | MariaDB 11.8.2                            |
| 템플릿/뷰  | JSP, Tiles                                |
| UI 에디터 | Summernote                                |
| 파일 업로드 | Multipart + 썸머노트 이미지 분리 처리                |
| 인증     | 비밀번호 확인 및 reCAPTCHA 검증                    |
| 로깅     | AOP 기반 로그 파일 출력                           |
| 배포 자동화 | systemd 서비스 등록 (DH-Homepage.service)      |
| 운영 자동화 | certbot + crontab 기반 인증서 자동 갱신            |
| OS 환경  | Ubuntu HamoniKR 기반 (서버용)                  |

---

## 📧 이메일 문의 기능 상세

- 홈페이지 내 "온라인 문의" 폼을 통해 방문자가 직접 문의를 남길 수 있음
- 입력된 문의 내용은 **Gmail SMTP**를 통해 운영자 이메일로 전송됨
- 발신자는 Gmail 계정 인증을 통해 메일 전송
- 수신자는 사내 운영자 계정 (예: [daehoeng@gmail.com](mailto\:daehoeng@gmail.com) 등)
- JavaMailSender를 활용한 SMTP 연동 구현

