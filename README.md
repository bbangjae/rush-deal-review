# Rush Deal 

## 프로젝트 개요
- **프로젝트명**: Rush Deal (타임딜 커머스)
- **기간**: 2025.11.24 ~ 2025.12 
- **팀 구성**: 백엔드 5명
- **담당 역할**: **팀장 / 인증·인가 / User 서비스(포인트) / 아키텍처 설계**
- **성격**: MSA 기반 대규모 트래픽 분산 처리를 위한 백엔드 엔지니어링 프로젝트

## 소개

Rush Deal은 **대규모 트래픽을 빠르게 처리(Rush)** 하는 것을 목표로 한 선착순 타임딜 커머스 플랫폼입니다.  
전반적인 내용과 제가 담당했던 핵심 역할을 **정리**하고, 개발 과정에서의 **기술적 회고**를 작성하였습니다.  

기존 Repository 주소: https://github.com/RushCrew/rush-deal

### 팀장 

#### 프로젝트 기획 및 아키텍처 설계
- **이벤트 스토밍 주도**: 도메인 이벤트 식별 및 서비스 경계 정의 ([상세 내용](https://a-steady-byun.tistory.com/8))
- **MSA 아키텍처 설계**: Spring Cloud(Eureka, Gateway, OpenFeign) 기반 서비스 디스커버리 및 통신 구조 설계
- **재고 관리 동시성 제어 전략 수립**: **Write-Behind 패턴** (Redis → Kafka → DB 비동기 반영)
- **이벤트 기반 보상 트랜잭션**: Saga Pattern 적용 (결제 실패 시 재고/포인트 자동 복구 플로우 설계)
- **API 명세 및 서비스 간 통신 정의**: RESTful API 설계, Kafka 이벤트 스키마 정의, OpenFeign 인터페이스 표준화
- **팀원 업무 조율 및 코드 리뷰**: DDD 기반 계층형 아키텍처 가이드라인 수립 및 코드, PR 리뷰

## 담당 영역

- **User Service**: 회원 관리, 포인트 시스템, 주소 관리 (DDD 기반 계층형 아키텍처 구현)
- **Auth Service**: JWT 기반 무상태 인증, Refresh Token 및 Blacklist 관리, Redis 기반 Rate Limiting 적용
- **API Gateway**: JWT 검증 필터, 라우팅 설정, 사용자 정보 헤더 주입

## 핵심 기술 및 주요 기능

### 1. DDD (Domain-Driven Design) 기반 계층형 아키텍처
- **4계층 분리**: Presentation, Application, Domain, Infrastructure로 명확한 관심사 분리
- **의존성 역전**: 도메인 레이어에서 인터페이스 정의, 인프라 레이어에서 구현 제공
- **Command/Result 패턴**: 불변 객체 기반 데이터 흐름으로 계층 간 데이터 전달
- **Multi-Level Validation**: Presentation(Bean Validation), Domain(비즈니스 규칙), Database(제약조건) 3단계 검증
- **유연한 아키텍처**: JWT 기반 인증과 OpenFeign·Kafka를 조합해 서비스 간 결합도를 낮춘 구조로 구축했습니다.

### 2. JWT 기반 무상태 인증 시스템
- **이중 토큰 전략**: Access Token(15분 TTL) + Refresh Token(7일 TTL, Redis 저장)로 보안성과 UX 균형
- **Gateway 중앙 인증**: JWT 검증 후 사용자 정보를 헤더(X-User-Id, X-User-Email, X-User-Role)로 주입하여 **내부 서비스 인증 부담 제거**
- **토큰 생성 분리**: AccessTokenProvider와 RefreshTokenProvider로 책임 분리, 각기 다른 Secret Key 사용
- **Token/User BlackList**: Redis 기반으로 로그아웃된 토큰 재사용 차단 및 전체 기기 로그아웃 지원
- **동시 로그인 제한**: 사용자당 최대 3개 세션 허용, 초과 시 가장 오래된 토큰 자동 삭제

### 3. MSA 서비스 간 통신 아키텍처
- **동기 통신(OpenFeign)**: Auth Service → User Service 간 비밀번호 검증 및 사용자 정보 조회 등 **즉각적인 응답이 필요**한 경우 사용
- **비동기 통신(Kafka)**: Order Service에서 결제 및 포인트 이벤트 발행, 보상 트랜잭션 처리 등 **느슨한 결합**이 **요구**되는 경우 사용
- **Feign 추상화**: Application Port(UserClient) + Infrastructure Adapter(UserClientImpl) 패턴으로 의존성 격리
- **Saga 패턴**: 주문 생성 실패 시 Kafka 이벤트 기반 보상 트랜잭션으로 데이터 정합성 보장

### 4. Redis 기반 보안 및 동시성 제어
- **Token BlackList**: 로그아웃된 Access Token을 남은 TTL 동안 Redis에 저장하여 재사용 차단
- **User BlackList**: 전체 기기 로그아웃 시 사용자를 블랙리스트 등록, 모든 Access Token 무효화
- **Refresh Token 관리**: Redis에 사용자별 토큰 리스트 저장, 동시 로그인 제한 정책 적용
- **분산 락(Redisson)**: 회원가입 중복 검증, 동시 로그인 제한, 포인트 증감 등 동시성 이슈 해결 (구현 예정)
- **동시성 제어**: Order Service에서 재고 차감 시 분산 락으로 Race Condition 방지 

### 5. 대규모 트래픽 처리 아키텍처
- **Write-Behind 패턴**: Redis 실시간 재고 차감 → Kafka 이벤트 발행 → DB 비동기 반영으로 쓰기 성능 최적화
- **이벤트 기반 아키텍처**: Kafka를 통한 주문-결제-포인트 서비스 간 비동기 통신으로 시스템 확장성 확보
- **Circuit Breaker**: Resilience4j 기반 서비스 장애 격리 및 Fallback 전략 (구현 예정)

---

## 기술 스택
| Category | Technology |
|----------|----------|
| Language | Java |
| Framework | Spring Boot, Spring Cloud |
| ORM | Spring Data JPA |
| Database | PostgreSQL |
| Cache | Redis |
| Message Queue | Kafka |
| Service Discovery | Eureka |
| API Gateway | Spring Cloud Gateway (WebFlux) |
| API Communication | OpenFeign |
| Authentication | JWT, BCrypt |
| Resilience | Resilience4j (Circuit Breaker) |
| Monitoring | Spring Actuator, Prometheus |

---

## 아키텍처
```
user-service/
├── presentation        # Controller Layer
│   ├── UserController.java          # REST API 엔드포인트
│   └── dto                           # HTTP 요청/응답 DTO
│       ├── request                   # 요청 DTO
│       └── response                  # 응답 DTO
│
├── application         # Service Layer (비즈니스 로직)
│   ├── UserService.java              # 유저 비즈니스 로직 처리
│   ├── command                       # Command 객체 (입력)
│   │   └── UserCreateCommand.java
│   └── result                        # Result 객체 (출력)
│       └── UserCreateResult.java
│
├── domain              # Domain Layer
│   ├── entity                        # 도메인 엔티티
│   │   └── User.java                 # 유저 엔티티
│   ├── enums                         # 도메인 열거형
│   │   └── UserRole.java             # 유저 역할 (USER, SELLER, MASTER)
│   ├── repository                    # 리포지토리 인터페이스
│   │   └── UserRepository.java
│   └── service                       # 도메인 서비스 인터페이스
│       └── UserValidator.java        # 유저 검증 서비스
│
└── infrastructure      # Infrastructure Layer
    ├── repository                    # 리포지토리 구현체
    │   ├── UserJpaRepository.java    # Spring Data JPA 인터페이스
    │   └── UserRepositoryImpl.java   # UserRepository 구현체
    └── validator                     # 검증 서비스 구현체
        └── UserValidatorImpl.java    # UserValidator 구현체
```


