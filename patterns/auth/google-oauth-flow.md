# [Pattern] 커스텀 팝업 기반의 소셜 로그인 연동 패턴

외부 라이브러리에 의존하지 않고, 브라우저의 팝업과 메시징 시스템을 활용하여 안전하게 소셜 인증을 수행하는 설계 패턴입니다.

---

## 1. 설계 원칙 (Design Principles)
- **UI/UX 자유도:** 컴포넌트 라이브러리에 종속되지 않는 독립적인 UI 구현.
- **통신 보안:** `postMessage` 전송 시 타겟 도메인 명시, 수신 시 소스 도메인 검증 필수.
- **사용자 경험:** 부모 창의 새로고침 없이 팝업을 통한 매끄러운 인증 프로세스 제공.

---

## 2. 구현 구조

### `vue3-google-login` 기존 구조

``` vue
<GoogleLogin
  :callback="handleGoogleLoginSuccess"
  @error="handleError"
  class="btn-round-line"
/>
```

```javascript
// 토큰 값 디코딩
const parseJwt = token => {
  const base64Url = token.split('.')[1];
  const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
  const jsonPayload = decodeURIComponent(
    atob(base64)
      .split('')
      .map(c => '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2))
      .join(''),
  );
  return JSON.parse(jsonPayload);
};

// 구글 로그인
const handleGoogleLoginSuccess = async googleUser => {
  console.log('Google 로그인 성공:', googleUser);

  const decoded = parseJwt(googleUser.credential);
  console.log('디코딩 정보:', decoded);

  const providerId = decoded.sub;
  const email = decoded.email;
  const name = decoded.name;

  $axios
    .post(
      $apiUrls.auth.userGoogleLogin,
      {
        idToken: googleUser.credential,
        provider: 'google',
        providerId: providerId,
        email,
        name,
      },
      {
        headers: {
          'Content-Type': 'application/json',
        },
      },
    )
    .then(res => {
      console.log('Google 로그인 성공', res.data);
      // const token = res.data.token;
      const user = res.data.user;

      auth.setAuth(user);

      const redirectPath = route.query.redirectTo || '/';
      router.push(redirectPath);
    })
    .catch(err => {
      console.error('Google 로그인 실패', err);
    });
};
```

```java
@PostMapping("/login/google")
public ResponseEntity<?> googleLogin(@RequestBody Map<String, String> body) {
String idToken = body.get("idToken");
    if (idToken == null || idToken.isEmpty()) {
        return ResponseEntity.badRequest().body(Map.of("message", "idToken is required"));
    }
    Map<String, Object> loginResult = authService.googleLogin(body);
    return ResponseEntity.status(HttpStatus.OK).body(loginResult);
}
```

### 🔹 [Frontend] 팝업 생성 및 리스너 등록
부모 창은 인증 페이지를 팝업으로 띄우고, `message` 이벤트를 통해 인증 결과를 기다립니다.

``` vue
<div class="login-wrapper">
  <button
    class="google-btn"
    :class="{ 'google-btn-dark': isDarkMode }"
    @click="googleLogin"
  >
    <img
      :src="googleIcon"
      alt="Google"
      class="google-icon"
      :class="{ 'fade-out': isDarkMode, 'fade-in': !isDarkMode }"
    />
    <img
      :src="googleIconDark"
      alt="Google Dark"
      class="google-icon absolute top-0 left-0"
      :class="{ 'fade-in': isDarkMode, 'fade-out': !isDarkMode }"
    />
    <span>Google 계정으로 로그인</span>
  </button>
</div>
```

```javascript
const clientId = import.meta.env.VITE_GOOGLE_CLIENT_ID;
const redirectUrl = import.meta.env.VITE_APP_GOOGLE_REDIRECT_URI;
const backendServerUrl = import.meta.env.VITE_APP_GOOGLE_SERVER_URL;
const scope = 'openid email profile'; // Oauth 범위

const googleLogin = () => {
  const width = 500;
  const height = 600;
  const left = window.screenX + (window.outerWidth - width) / 2;
  const top = window.screenY + (window.outerHeight - height) / 2;

  window.open(
    redirectUrl,
    'google-oauth',
    `width=${width},height=${height},top=${top},left=${left}`,
  );

  window.addEventListener('message', event => {
    console.log(
      'event origin 체크 : ' +
        event.origin +
        ' , 백엔드 : ' +
        backendServerUrl +
        ', window : ' +
        window.location.origin,
    );
    if (event.origin !== backendServerUrl) {
      console.log(
        'event origin 불일치 : ' +
          event.origin +
          ' , 백엔드 : ' +
          backendServerUrl +
          ', window : ' +
          window.location.origin,
      );
      return;
    }
    const { token, refreshToken, user } = event.data;
    console.log('Google 로그인 성공');
    auth.setAuth(user);

    const redirectPath = route.query.redirectTo || '/';
    router.push(redirectPath);
  });
};

<style scoped>
.google-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  padding: 0.5rem;
  border-radius: 4px;
  border: 1px solid #ddd;
  background-color: white;
  font-weight: 500;
  cursor: pointer;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  transition:
    background 0.2s,
    box-shadow 0.2s;
}

.google-btn:hover {
  background-color: #f7f7f7;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.15);
}

.google-btn-dark {
  background-color: #213547;
  border: 1px solid #333;
  color: #fff;
}

.google-btn-dark:hover {
  background-color: #213547;
}

.google-icon {
  width: 20px;
  height: 20px;
  margin-right: 8px;
}
</style>
```

### 🔹[Backend] 콜백 페이지 구현
백엔드는 구글 콜백 페이지에서 `postMessage`로 전달된 데이터를 수신하고, 이를 처리하여 사용자 인증을 완료합니다.

```java
@GetMapping("/login/google")
public void googleLoginRedirect(HttpServletResponse response) throws IOException {
    String clientId = "google Oauth ID 값 입력";
    String redirectUri = authInfo.getGoogleRedirectUri();
    String scope = "openid email profile";

    String oauthUrl = "https://accounts.google.com/o/oauth2/v2/auth?" +
                        "client_id=" + clientId +
                        "&redirect_uri=" + URLEncoder.encode(redirectUri, "UTF-8") +
                        "&response_type=code" +
                        "&scope=" + URLEncoder.encode(scope, "UTF-8") +
                        "&prompt=select_account";
    response.sendRedirect(oauthUrl);
}

@GetMapping("/login/google/callback")
public void googleLogin(@RequestParam("code") String code, HttpServletResponse response) throws IOException, JsonProcessingException {
    Map<String, Object> loginResult = authService.googleLoginByOauth(code);

    String json = new ObjectMapper().writeValueAsString(loginResult);
    String uri = authInfo.getGoogleUri();

    // HTML + JS 반환
    String html = "<!DOCTYPE html><html><body>" +
                    "<script>" +
                    "window.opener.postMessage(" + json + ", '" + uri + "');" +
                    "window.close();" +
                    "</script>" +
                    "</body></html>";

    response.setContentType("text/html;charset=UTF-8");
    response.getWriter().write(html);
}
```
---

## 3. 기대 효과

1. 일관된 디자인: 서비스 테마(Dark/Light)에 맞춘 UI 디자인 가능.
2. 모듈화: 소셜 제공자(Google, Kakao, Naver 등)가 늘어나도 동일한 팝업-메시징 인터페이스로 확장 가능.

---

> **리팩토링 사례 보기:** [구글 로그인 구현 방식 전환](../../dev-notes/auth/google-oauth-implementation.md)