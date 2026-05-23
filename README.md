# ✔️ MomentLock - 타임캡슐 소셜 플랫폼
<div align="center">
추억을 봉인하고, 미래에 함께 열어보는 소셜 타임캡슐 서비스
이미지 표시
이미지 표시
이미지 표시
</div>

## 📋 목차
- 프로젝트 소개
- 주요 기능
- 기술 스택
- 아키텍처
- 핵심 구현 사항
- 데이터베이스 설계
- 실행 방법

## 🎯 프로젝트 소개
MomentLock은 소중한 순간을 타임캡슐에 담아 미래의 특정 시점에 공개하는 소셜 플랫폼입니다.
개인 또는 그룹이 함께 추억을 저장하고, 지정된 날짜에 자동으로 공개되는 독특한 경험을 제공합니다.

## 💡 프로젝트 핵심 가치
협업 중심 설계: 여러 사용자가 하나의 박스에 캡슐을 함께 생성
시간 기반 잠금: 설정한 개봉 날짜까지 콘텐츠 보호
위치 기반 서비스: 지도에서 공개된 박스 탐색
소셜 기능: 좋아요, 공유, 초대 시스템

## 📊 프로젝트 성과
RESTful API 20+ 엔드포인트 설계 및 구현
복잡한 다대다 관계를 포함한 9개 엔티티 설계
AWS S3를 활용한 멀티미디어 파일 관리 시스템 구축
Spring Security 기반 인증/인가 체계 구현

## ✨ 주요 기능

### 1️⃣ 타임캡슐 관리
- 텍스트, 이미지, 동영상을 포함한 캡슐 생성
- 개봉 날짜 지정 및 자동 공개
- 랜덤 썸네일 자동 배정
- 좋아요 및 신고 시스템
  
### 2️⃣ 박스 시스템
- 공개/비공개 박스 생성
- 최대 참여 인원 설정
- 위치 정보 기반 박스 관리
- 준비 완료 후 자동 봉인 (Bury) 처리
  
### 3️⃣ 협업 기능
- 이메일 기반 초대 시스템
- 토큰 기반 초대 링크 (24시간 유효)
- 방장 권한 관리
- 멤버 강퇴 기능
  
### 4️⃣ 소셜 기능
- 인기 박스 랭킹 (좋아요 수 기반)
- 오픈 예정 박스 타임라인
- 지도 기반 박스 탐색
- 박스 전송 기능
  
### 5️⃣ 관리 시스템
- 회원/박스/캡슐 관리
- 신고 처리 및 제재
- 문의사항 관리
- 공지사항/QA 게시판
  
## 🛠 기술 스택

### Backend
- Framework: Spring Boot 3.x
- Security: Spring Security (BCrypt 암호화)
- ORM: JPA/Hibernate
- Database: Oracle Database
- Build Tool: Maven

### Cloud & Storage
- AWS S3: 멀티미디어 파일 저장 및 관리
- File Upload: MultipartFile 처리

### Communication
- Email: JavaMailSender + Thymeleaf Template
- Async Processing: @Async 비동기 메일 발송

### View
- Template Engine: Thymeleaf
- Frontend: HTML5, CSS3, JavaScript

## 🏗 아키텍처

```
┌─────────────────┐
│   Presentation  │  ← Controller Layer (RESTful API)
├─────────────────┤
│    Business     │  ← Service Layer (비즈니스 로직)
├─────────────────┤
│   Persistence   │  ← Repository Layer (JPA)
├─────────────────┤
│    Database     │  ← Oracle DB
└─────────────────┘
        ↓
   ┌─────────┐
   │  AWS S3 │  ← 파일 스토리지
   └─────────┘
```

### 주요 설계 패턴
- Layered Architecture: Controller → Service → Repository 구조
- DTO Pattern: Entity와 View 분리로 캡슐화
- Builder Pattern: 복잡한 객체 생성 단순화
- Composite Key: @EmbeddedId를 활용한 다대다 관계 처리

## 🔥 핵심 구현 사항
### 1. 복잡한 도메인 모델링

```Java
@Entity
@Table(name = "MEMBER_BOX")
public class MemberBox {
    @EmbeddedId
    private MemberBoxId id;  // 복합키
    
    @MapsId("username")
    @ManyToOne(fetch = FetchType.LAZY)
    private Member member;
    
    @MapsId("boxId")
    @ManyToOne(fetch = FetchType.LAZY)
    private Box box;
    
    @Column(name = "BOXMATERCODE")
    private String boxmatercode;  // 방장 권한 관리
}
```

### 포인트
- @EmbeddedId를 활용한 복합키 설계
- Lazy Loading으로 성능 최적화
- 다대다 관계에 추가 속성(방장 여부, 준비 상태 등) 관리

### 2. AWS S3 파일 관리 시스템

```Java
@Service
public class AfileServiceImpl implements AfileService {
    
    // 파일 업로드: UUID로 파일명 중복 방지
    public String uploadToS3(MultipartFile file) throws IOException {
        String fileName = UUID.randomUUID().toString() + extension;
        
        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentType(file.getContentType());
        metadata.setContentLength(file.getSize());
        
        amazonS3.putObject(new PutObjectRequest(
            bucketName, fileName, file.getInputStream(), metadata
        ));
        
        return amazonS3.getUrl(bucketName, fileName).toString();
    }
    
    // 트랜잭션 내에서 DB와 S3 동시 처리
    @Transactional
    public Afile saveFileToCapsule(MultipartFile file, Capsule capsule) {
        String s3Url = uploadToS3(file);
        
        Afile afile = Afile.builder()
            .afsname(file.getOriginalFilename())
            .afcname(s3Url)
            .capsule(capsule)
            .build();
            
        capsule.setCapafilecount(capsule.getCapafilecount() + 1);
        return afileRepository.save(afile);
    }
}
```

### 포인트
- UUID를 활용한 파일명 충돌 방지
- S3 업로드와 DB 저장을 하나의 트랜잭션으로 처리
- 파일 메타데이터 관리

### 3. 이메일 초대 시스템
```Java
@Service
public class InvitationServiceImpl implements InvitationService {
    
    @Transactional
    public void sendInvitation(String inviterNickname, String inviteeNickname, Long boxId) {
        // 토큰 생성 (24시간 유효)
        String token = UUID.randomUUID().toString().replace("-", "");
        
        InviteToken entity = new InviteToken();
        entity.setToken(token);
        entity.setExpiresAt(LocalDateTime.now().plusHours(24));
        tokenRepo.save(entity);
        
        // 초대 링크 생성
        String inviteUrl = UriComponentsBuilder
            .fromHttpUrl("http://localhost:8888")
            .path("/momentlock/invite")
            .queryParam("token", token)
            .build(true)
            .toUriString();
            
        // 비동기 메일 발송
        mailService.sendInviteMail(invitee.getUsername(), inviteUrl);
    }
}
```

### 포인트
- UUID 기반 일회용 토큰 생성
- 만료 시간 관리로 보안 강화
- @Async를 활용한 비동기 메일 발송으로 응답 속도 개선

### 4. 동적 쿼리 및 페이징
```Java
@Repository
public interface BoxRepository extends JpaRepository<Box, Long> {
    
    // Native Query로 복잡한 집계 쿼리 처리
    @Query("SELECT b.boxid, b.boxname, SUM(c.caplikecount) " +
           "FROM Capsule c JOIN c.box b " +
           "WHERE b.boxdelcode = 'BDN' AND b.boxcapcount >= 1 " +
           "GROUP BY b.boxid, b.boxname " +
           "ORDER BY SUM(c.caplikecount) DESC")
    Page<BoxLikeCountDto> getPopularBoxPage(Pageable pageable);
}
```

### 포인트
- JPQL을 활용한 복잡한 집계 쿼리
- Pageable로 페이징 처리 자동화
- DTO Projection으로 필요한 데이터만 조회

### 5. Spring Security 인증/인가
```Java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/momentlock/master/**").hasRole("ADMIN")
                .requestMatchers("/momentlock/**").hasAnyRole("ADMIN", "USER")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/html/member/login")
                .defaultSuccessUrl("/momentlock", true)
            )
            .sessionManagement(session -> session
                .maximumSessions(1)
                .maxSessionsPreventsLogin(false)
            );
            
        return http.build();
    }
}
```

### 포인트
- Role 기반 접근 제어 (ADMIN, USER)
- 세션 동시성 제어
- BCrypt 암호화로 비밀번호 보안

## 💾 데이터베이스 설계

### 주요 엔티티 관계
```
Member (회원)
  ├─ 1:N ─ Capsule (타임캡슐)
  ├─ 1:N ─ Payment (결제)
  ├─ 1:N ─ Inquiry (문의)
  └─ M:N ─ Box (박스) [through MemberBox]

Box (박스)
  ├─ 1:N ─ Capsule
  └─ M:N ─ Member [through MemberBox]

Capsule (캡슐)
  ├─ 1:N ─ Afile (첨부파일)
  ├─ 1:N ─ Emoji (이모지)
  └─ 1:N ─ Declaration (신고)
```

## 🚀 실행 방법

### 사전 요구사항
- JDK 17 이상
- Oracle Database
- AWS S3 계정 (파일 업로드용)
- SMTP 서버 (이메일 발송용)

### 환경 설정
```properties
# application.properties

# Database
spring.datasource.url=jdbc:oracle:thin:@localhost:1521:xe
spring.datasource.username=your_username
spring.datasource.password=your_password

# AWS S3
cloud.aws.credentials.access-key=your_access_key
cloud.aws.credentials.secret-key=your_secret_key
cloud.aws.s3.bucket=your_bucket_name
cloud.aws.region.static=ap-northeast-2

# Mail
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your_email
spring.mail.password=your_password
```

### 실행
```bash
# 프로젝트 클론
git clone [repository-url]

# 의존성 설치
mvn clean install

# 애플리케이션 실행
mvn spring-boot:run
```

### 접속: http://localhost:8888/momentlock

## 📈 개선 사항 및 학습 포인트

### 구현하면서 배운 것들

**1. 복잡한 연관관계 설계**
- @EmbeddedId를 활용한 복합키 처리
- Lazy/Eager Loading 전략 선택
- N+1 문제 해결 (Fetch Join)


**2. 클라우드 서비스 통합**
- AWS S3 SDK를 활용한 파일 관리
- 트랜잭션 내에서 외부 API 호출 처리
- 파일 삭제 시 DB-S3 동기화


**3. 보안 구현**
- Spring Security Filter Chain
- BCrypt 암호화
- 토큰 기반 인증 (초대 시스템)


**4. 비동기 처리**
- @Async를 활용한 이메일 발송
- 응답 속도 개선



## 향후 개선 계획
- Redis를 활용한 캐싱 전략 도입
- WebSocket 실시간 알림 시스템
- RESTful API 문서화 (Swagger/OpenAPI)
- 단위 테스트 커버리지 80% 이상
- CI/CD 파이프라인 구축
- Docker 컨테이너화
