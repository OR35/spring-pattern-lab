# [Refactoring] 중복된 인증 정보 추출 로직의 캡슐화 및 서비스 코드 간소화

사용자(User)와 관리자(Admin) 권한이 분리된 시스템에서, 중복되는 인증 정보 조회 로직을 유틸리티화하여 서비스 레이어의 응집도를 높인 기록입니다.

---

## 1. 문제 상황 (As-Is)
Service 레이어의 삭제 로직 내에서 클라이언트 타입에 따라 직접 인증 객체를 캐스팅하고 정보를 추출하는 방식이었습니다.

```java
public class SecurityUtil {  
  
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

```java
// AS-IS: 서비스 로직 내에 인증 정보 추출 로직이 혼재됨
@Transactional  
public void deleteOptimization(String optimizationId, String deleteReason, String clientType) {  
    String delrId;  
    String delrNm;  
  
    if ("admin".equals(clientType)) {  
        LoginAdmin loginAdmin = SecurityUtil.getLoginAdmin();  
        delrId = loginAdmin.getAdminId();  
        delrNm = loginAdmin.getAdminNm();  
    } else {  
        LoginUser loginUser = SecurityUtil.getLoginUser();  
        delrId = loginUser.getUserId();  
        delrNm = loginUser.getUserNm();  
    }  
    // ... 실제 비즈니스 로직 시작
}
```

🧐 문제점
- 관심사 분리 실패: 서비스 로직이 "누가 삭제하는지"를 알아내기 위한 인프라적인 로직(SecurityContext 접근 등)까지 직접 관리함.

- 코드 중복: 삭제뿐만 아니라 등록, 수정 등 사용자 정보가 필요한 모든 서비스 메서드에서 위와 같은 if-else 블록이 반복됨.

- 유지보수성 저하: 사용자 정보 필드명이 변경되거나 권한 체계가 추가될 경우, 모든 서비스 코드를 수정해야 함.

---

## 2. 해결 과정 및 개선 (To-Be)
SecurityUtil 내부에 공통 DTO(UserInfo)를 정의하고, 클라이언트 타입에 따라 정보를 통합 반환하는 인터페이스를 구축했습니다.

개선 로직
1. 내부 클래스 UserInfo 도입: 사용자/관리자 구분 없이 필요한 정보(ID, 이름)만 담는 공통 규격 생성.

2. getCurrentUser 메서드 구현: 인증 로직을 유틸리티로 완전히 격리.

```java
public class SecurityUtil {  
  
    @Getter  
    @AllArgsConstructor    public static class UserInfo {  
        private final String userId;  
        private final String userNm;  
    }  
      
    //클라이언트 타입에 따라 관리자 또는 사용자 정보를 통합 반환  
    public static UserInfo getCurrentUser(String clientType) {  
        if ("admin".equals(clientType)) {  
            LoginAdmin admin = getLoginAdmin();  
            return (admin != null) ? new UserInfo(admin.getAdminId(), admin.getAdminNm()) : null;  
        } else {  
            LoginUser user = getLoginUser();  
            return (user != null) ? new UserInfo(user.getUserId(), user.getUserNm()) : null;  
        }  
    }  
      
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

```java
// TO-BE: 서비스 코드는 한 줄로 사용자 정보 획득 가능
@Transactional  
public void deleteOptimization(String optimizationId, String deleteReason, String clientType) {  
    SecurityUtil.UserInfo user = SecurityUtil.getCurrentUser(clientType); // 캡슐화된 메서드 호출
    
    OptimizationEntity optimization = optimizationQuery.getOptimization(optimizationId);  
    optimization.markAsDeleted(deleteReason, user.getUserId(), user.getUserNm());  
    // ... 
}
```

---

## 3. 리팩토링 핵심 포인트
✅ SRP (단일 책임 원칙) 준수
서비스 레이어는 오직 비즈니스 흐름(삭제 및 외부 연동)에만 집중합니다. 인증 정보를 파싱하고 검증하는 책임은 SecurityUtil로 완전히 분리되었습니다.

✅ 타입 추상화를 통한 결합도 낮춤
내부 클래스 UserInfo를 통해 서로 다른 두 로그인 객체의 인터페이스를 통일했습니다. 이제 서비스 레이어는 구체적인 구현 클래스가 아닌 규격화된 DTO에 의존합니다.

---

## 4. 성과 및 회고

가독성 향상: 10줄에 가까운 인증 체크 로직이 단 1줄로 줄어들어 핵심 비즈니스 로직이 한눈에 들어옵니다.

재사용성 극대화: 사용자 정보가 필요한 다른 도메인(센서 관리, 모델 관리 등)에서도 동일한 방식으로 처리가 가능합니다.

확장성: 추후 새로운 권한(예: 시스템 매니저)이 추가되어도 SecurityUtil의 한 지점만 수정하면 됩니다.

---

> **연관 패턴:** [Spring Security 인증 정보 통합 추출 패턴](../../patterns/auth/auth-security-userinfo.md)