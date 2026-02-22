# Auth Domain Skill

auth-server 작업 시 헥사고날 아키텍처 패턴을 강제하는 스킬.

## 새 기능 추가 체크리스트

1. **도메인 레이어** (변경이 필요한 경우)
   - VO 추가/수정: `domain/vo/`
   - 엔티티 수정: `domain/entity/`

2. **애플리케이션 레이어**
   - 유스케이스 인터페이스: `application/port/in/`
   - 리포지토리 인터페이스: `application/port/out/`
   - 유스케이스 구현: `application/service/`

3. **인프라 레이어**
   - JPA 리포지토리: `infra/repository/`
   - JPA 엔티티 매핑 확인

4. **웹 레이어**
   - 컨트롤러: `web/controller/`
   - DTO: 요청/응답 분리
   - Spring Security 설정 확인

## 금지 패턴
- 컨트롤러에 비즈니스 로직 직접 작성 (❌)
- 서비스에서 웹 계층 클래스 참조 (❌)
- 도메인 엔티티를 API 응답으로 직접 노출 (❌)

## VO 생성 패턴
```java
public record Email(String value) {
    public Email {
        if (value == null || !value.contains("@"))
            throw new IllegalArgumentException("Invalid email");
    }
}
```
