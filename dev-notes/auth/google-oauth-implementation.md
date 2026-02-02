# [Refactoring] Google OAuth 2.0 구현 방식 비교 및 최적화

구글 로그인을 구현하며 사용한 두 가지 방식(Custom Popup + postMessage / vue3-google-login 라이브러리)의 차이점과 보안적 고려사항을 정리합니다.

## 1. 구현 방식 비교

| 구분 | 방식 1: Custom Popup (`postMessage`) | 방식 2: `vue3-google-login` 라이브러리 |
| :--- | :--- | :--- |
| **흐름** | 백엔드 Redirect → Callback 페이지에서 JS로 데이터 전송 | 프론트에서 직접 Google 인증 → `idToken` 백엔드 전송 |
| **장점** | 백엔드에서 인증 로직을 전담하여 보안 제어 용이 | 프론트 구현이 매우 간편하고 UX가 매끄러움 |
| **특징** | `window.open`과 `window.opener.postMessage` 활용 | JWT 디코딩을 통한 프론트 정보 선확인 가능 |

---

## 2. 주요 이슈 및 해결 과정

### 이슈 1: 팝업창 데이터 통신 (Cross-Origin)
- **문제**: 백엔드 콜백 페이지에서 로그인을 완료한 후, 부모 창(Vue App)으로 데이터를 전달해야 함.
- **해결**: `window.opener.postMessage`를 사용하되, `event.origin` 체크를 통해 허용된 백엔드 도메인에서 온 메시지만 수락하도록 보안 로직 강화.

### 이슈 2: JWT (idToken) 디코딩
- **문제**: 구글에서 넘겨준 `credential`은 인코딩된 JWT 형태라 백엔드에 보내기 전 프론트에서 사용자 정보를 확인할 수 없음.
- **해결**: `atob`와 `decodeURIComponent`를 활용한 JWT 파싱 함수(`parseJwt`)를 구현하여 백엔드 호출 전 `email`, `name` 등을 선추출함.

---

## 3. 핵심 코드 (백엔드 HTML 반환 방식)
백엔드 콜백에서 JSON 데이터를 바로 반환하는 것이 아니라, 브라우저가 실행할 수 있는 HTML/JS를 반환하여 팝업을 닫고 부모 창에 데이터를 전달함.

```java
String html = "<!DOCTYPE html><html><body>" +
              "<script>" +
              "window.opener.postMessage(" + json + ", '" + uri + "');" +
              "window.close();" +
              "</script>" +
              "</body></html>";

---

> **연관 패턴:** [Google OAuth 2.0 인증 흐름 패턴](../../patterns/auth/google-oauth-flow.md)
# [Refactoring] Google OAuth 2.0: 라이브러리에서 커스텀 팝업 방식으로의 전환

디자인 커스터마이징(다크모드 대응) 및 Form 제어의 유연성을 확보하기 위해 기존 라이브러리(`vue3-google-login`)를 걷어내고 직접 OAuth 흐름을 구현한 기록입니다.

---

## 1. 문제 상황 (As-Is)
기존에는 `vue3-google-login` 라이브러리를 사용하여 빠르고 간편하게 로그인을 구현했습니다.

### 🧐 발생한 한계점
* **디자인 커스터마이징 제약:** 서비스의 다크모드/라이트모드 디자인 가이드에 맞춘 커스텀 버튼 구현이 까다로움.
* **비즈니스 로직 제어:** 로그인 성공 후 추가적인 Form 데이터 처리나 리다이렉션 로직을 라이브러리 내부 사이클에 맞추어야 하는 제약 발생.

---

## 2. 해결 과정 및 개선 (To-Be)
라이브러리를 제거하고, **`window.open` (팝업)**과 **`postMessage` API**를 사용하여 직접 구글 OAuth 2.0 흐름을 제어하도록 리팩토링했습니다.


### 개선 포인트
1. **커스텀 UI 구현:** 다크모드/라이트모드에 따라 아이콘과 스타일이 실시간으로 변하는 커스텀 버튼(`google-btn`)을 완전하게 제어.
2. **팝업 통신 규격화:** 백엔드 콜백 페이지에서 로그인을 처리한 후, 결과를 부모 창으로 전달하는 `postMessage` 보안 통신 구축.
3. **중앙 집중식 상태 관리:** 로그인 결과를 수신하자마자 `auth.setAuth(user)`를 통해 전역 상태를 업데이트하고 동적 리다이렉션 수행.

```javascript
const googleLogin = () => {
  // 1. 화면 중앙에 팝업 위치 계산 및 오픈
  const popup = window.open(redirectUrl, 'google-oauth', `width=${width},height=${height},top=${top},left=${left}`);

  // 2. 백엔드(팝업)로부터 인증 결과 수신 대기
  window.addEventListener('message', (event) => {
    if (event.origin !== backendServerUrl) return; // 보안: 신뢰할 수 있는 서버만 수락
    
    const { user } = event.data;
    auth.setAuth(user); // 전역 상태 저장
    router.push(route.query.redirectTo || '/');
  }, { once: true }); // 이벤트 리스너 자동 제거
};
```
---

## 3. 성과 및 회고

1. 자율성 확보: 더 이상 외부 라이브러리의 업데이트나 제약 사항에 얽매이지 않고 서비스 요구사항에 맞는 로그인을 구현할 수 있게 됨.
2. 보안 강화: postMessage 수신 시 origin 검증 로직을 직접 구현함으로써 크로스 사이트 스크립팅(XSS) 위협에 대응함.

---

> **연관 패턴:** [커스텀 팝업 기반의 소셜 로그인 연동 패턴](../../dev-notes/auth/auth-userinfo-refactoring-log.md)