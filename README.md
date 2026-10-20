### 프로젝트 설명
---
리그 오브 레전드 LCK 경기 정보를 크롤링하여 제공하는 서비스를 구현했습니다. 경기 정보를 수집하는 방식과 데이터 갱신 주기를 설계하고, 선호 팀의 경기 일정 및 시작 시점에 맞춰 FCM (Firebase Cloud Messaging)으로 알림을 전송하는 기능의 구현에 중점을 두었습니다.<br><br>

Blog : https://velog.io/@ayeah77/series/Lck-경기-정보-제공-프로젝트-크롤링<br>

### 기술 스택
---
Backend : Java, Spring Boot, Spring Data JPA, Spring Security

Frontend : React.js, JavaScript

Database : MySQL

Auth : JWT, Google OAuth2

Etc : Selenium WebDriver, Firebase ( push 알림 )<br><br>

### 아키텍처
---
<img width="1902" height="901" alt="image" src="https://github.com/user-attachments/assets/115f8077-c12c-454b-9939-ad173ed5b429" /><br><br>


### 구현
---
- Selenium 기반 LCK 데이터 수집 파이프라인
    
    LCK 일정과 순위는 공식 API가 없고, 페이지가 JavaScript로 렌더링되어 HTML만 받아오는 Jsoup으로는 데이터를 가져올 수 없었습니다. 따라서, 실제 브라우저로 페이지를 렌더링하는 Selenium으로 크롤링했습니다. 매 주기마다 전체를 덮어쓰지 않고 변경된 데이터만 저장, 갱신하여 불필요한 DB 쓰기를 줄였습니다.
    
- CompletableFuture 기반 병렬 크롤링
    
    일정과 순위 크롤링은 서로 독립적인데도 순차로 실행되고 있어, 두 작업을 병렬로 실행하도록 개선했습니다. 결과를 블로킹으로 기다려야 하는 `Future`와 달리, 여러 작업의 완료를 조합하고 작업별 예외를 처리할 수 있는 `CompletableFuture`를 사용했습니다. 그 결과 전체 크롤링 시간을 평균 `3,708ms에서 2,742ms로 약 26%` 단축했습니다.
    
- FCM 기반 선호 팀 경기 알림
    
    경기 시작 알림은 전달 시점이 중요하므로 이메일 대신 즉시 전달할 수 있는 푸시 알림을 선택했습니다. FCM을 활용해 플랫폼별 별도 구현 없이 다양한 기기에 알림을 전송하도록 구현했습니다.
    
- Spring Security 기반 JWT 인증과 OAuth 2.0 소셜 로그인을 구현했습니다
- JUnit 기반 단위 테스트
    
    크롤링과 FCM처럼 외부에 의존하는 부분은 실행할 때마다 결과가 달라질 수 있어, 이를 분리하고 핵심 비즈니스 로직만 단위 테스트로 검증했습니다.<br><br>


### 트러블슈팅
---  
- I/O-bound 크롤링을 위한 전용 스레드 풀 분리
    
    [ 상황 ]
    
    `CompletableFuture`를 활용하여 Selenium 크롤링 작업을 비동기로 처리하고 있었습니다.
    
    [ 문제 ]
    
    별도의 실행 풀을 지정하지 않아 작업이 `ForkJoinPool.commonPool()`에서 실행되었고, Selenium의 네트워크 I/O 동안 스레드가 점유되었습니다. 이로 인해 I/O-bound 작업이 공용 스레드 풀의 스레드를 장시간 점유할 수 있는 문제가 있었습니다.
    
    [ 해결 ]
    
    크롤링 작업을 위한 전용 `ThreadPoolTaskExecutor`를 구성하고 `CompletableFuture`가 해당 풀을 사용하도록 변경했습니다. 이를 통해 크롤링 작업과 다른 비동기 작업의 실행 자원을 분리했습니다.
    
- 트랜잭션 자기 호출 문제 해결
    
    [ 상황 ]
    
    서비스 로직의 일부 메서드에 `@Transactional`을 적용하여 트랜잭션 단위를 관리하고 있었습니다.
    
    [ 문제 ]
    
    `@Transactional`이 적용된 메서드를 동일 클래스의 다른 메서드에서 직접 호출하는 자기 호출 방식으로 사용하면서, Spring AOP 프록시를 거치지 않아 트랜잭션이 적용되지 않는 문제가 발생했습니다.
    
    [ 해결 ]
    
    트랜잭션이 필요한 로직을 별도의 클래스로 분리하고 해당 클래스를 통해 호출하도록 변경함으로써, Spring AOP 프록시를 거쳐 `@Transactional`이 정상적으로 적용되도록 개선했습니다.<br><br>

### 페이지
---
**[ 홈 페이지 ]**<br><br>
<img width="1710" height="922" alt="Image" src="https://github.com/user-attachments/assets/02cce9de-e574-4154-9193-f67b5936071f" /><br><br><br>

**[ 순위 페이지 ]**<br><br>
<img width="1710" height="804" alt="Image" src="https://github.com/user-attachments/assets/68beafa7-081d-4938-b0d2-6f7ab88eeb7a" /><br><br><br>

**[ 로그인 시 화면 ]**<br><br>
선호하는 팀을 선택 가능하며, 선호하는 팀의 경기를 필터링 해서 볼 수 있습니다.<br><br>
<img width="1710" height="804" alt="Image" src="https://github.com/user-attachments/assets/29a6e814-a42b-4a31-b390-1e28ffd6c0aa" /><br><br><br>

**[ 알림 기능 ]**<br><br>
알림 허용 기능을 구현하여 사용자 임의대로 알림 허용 여부 선택이 가능합니다.<br>
알림은 FCM을 사용하여 크롬 알림으로 구현했습니다.<br><br>
<img width="1710" height="804" alt="Image" src="https://github.com/user-attachments/assets/e9f7b38a-1a22-41c3-bf58-7bf58f5cc905" /><br><br><br>
