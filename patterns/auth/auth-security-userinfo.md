# [Pattern] Spring Security 인증 정보 통합 추출 패턴
애플리케이션 내에서 권한 체계(User/Admin)가 이원화되어 있을 때, 서비스 레이어의 의존성을 줄이고 일관된 방식으로 인증 정보를 획득하기 위한 설계 패턴입니다.

---

## 1. 문제 상황 (Before)
서비스 레이어에서 SecurityContext에 직접 접근하여 구체적인 구현 클래스(LoginUser, LoginAdmin)로 캐스팅하는 로직이 산재함.

사용자 정보가 필요한 모든 비즈니스 로직마다 중복된 if-else 조건문과 수동 필드 매핑이 발생하여 코드 가독성을 저해함.

---

## 2. 해결 방안 (After): UserInfo DTO 및 통합 유틸리티 도입
인증 객체의 종류에 상관없이 필요한 정보(ID, 이름 등)만 추출하여 하나의 공통 DTO로 규격화하고, 이를 반환하는 통합 인터페이스를 구축했습니다.

- 관심사 분리(SoC)

서비스 로직은 "누가" 요청했는지에 대한 인프라적 세부 사항을 몰라도 비즈니스 로직을 수행할 수 있습니다.

- 타입 추상화

서로 다른 로그인 객체의 공통 필드를 UserInfo라는 단일 모델로 통합하여 서비스 레이어와의 결합도를 낮췄습니다.

### 핵심 로직
통합 정보 객체 선언: 유틸리티 내부 정적 클래스(UserInfo)로 필요한 최소 정보를 정의.

1. 캡슐화된 추출 메서드: SecurityContext 접근 및 타입 체크 로직을 유틸리티 메서드 내부로 은닉.
2. 불변 객체 반환: 생성된 DTO는 수정이 불가능하도록 설계하여 데이터 신뢰성 확보.

---

## 3. 구현 코드 (Java / Spring Security)

```java
public class SecurityUtil {

    // 1. 통합 사용자 정보 DTO (Single Model for Service Layer)
    @Getter
    @AllArgsConstructor
    public static class UserInfo {
        private final String userId;
        private final String userNm;
    }

    // 2. 통합 추출 인터페이스 (Type Abstraction)
    public static UserInfo getCurrentUser(String clientType) {
        if ("admin".equals(clientType)) {
            LoginAdmin admin = getLoginAdmin(); // 내부 전용 메서드
            return (admin != null) ? new UserInfo(admin.getAdminId(), admin.getAdminNm()) : null;
        } else {
            LoginUser user = getLoginUser();   // 내부 전용 메서드
            return (user != null) ? new UserInfo(user.getUserId(), user.getUserNm()) : null;
        }
    }

    // 3. 내부 구현 (SecurityContext 접근 은닉)
    public static LoginUser getLoginUser() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        if (auth == null || !(auth.getPrincipal() instanceof LoginUser)) {
            return null;
        }
        return (LoginUser) auth.getPrincipal();
    }

    public static LoginAdmin getLoginAdmin() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        if (auth == null || !(auth.getPrincipal() instanceof LoginAdmin)) {
            return null;
        }
        return (LoginAdmin) auth.getPrincipal();
    }
}
```

---

> **리팩토링 사례 보기:** [중복된 인증 정보 추출 로직의 캡슐화](../../dev-notes/auth/auth-userinfo-refactoring-log.md)