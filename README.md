# 달러 투자 추적 (Firebase + GitHub Pages)

매수 환율과 금액을 기록하고, 현재 환율 기준 평가손익을 그래프와 숫자로 보는 웹앱입니다.
서버 없이 정적 파일만으로 동작합니다.

- 호스팅: GitHub Pages
- 로그인/데이터: Firebase Authentication (Google) + Cloud Firestore
- 환율: Frankfurter API (키 불필요, 유럽중앙은행 고시 기준, 하루 1회 갱신)

## 파일 구성

| 파일 | 역할 |
|---|---|
| `index.html` | 화면 |
| `style.css` | 스타일 |
| `app.js` | 로그인, 데이터 저장, 환율 조회, 그래프 |
| `firebase-config.js` | Firebase 연결 설정 (직접 채워야 함) |
| `firestore.rules` | 데이터 보안 규칙 |

## 1. Firebase 설정

1. [Firebase 콘솔](https://console.firebase.google.com)에서 프로젝트를 만듭니다.
2. **Authentication > 시작하기 > 로그인 방법**에서 **Google**을 사용 설정합니다.
3. **Firestore Database > 데이터베이스 만들기**를 누르고, 위치는 `asia-northeast3 (서울)`을 고릅니다. 프로덕션 모드로 시작해도 됩니다.
4. Firestore의 **규칙** 탭에 `firestore.rules` 내용을 붙여넣고 **게시**합니다.
5. **프로젝트 설정 > 내 앱**에서 웹 앱(`</>`)을 추가하고, 나오는 `firebaseConfig` 값을 `firebase-config.js`에 붙여넣습니다.

## 2. GitHub Pages 배포

1. GitHub에 새 저장소를 만들고 이 폴더의 파일 5개를 올립니다. (`README.md`는 선택)
2. 저장소의 **Settings > Pages**에서 Source를 `Deploy from a branch`, Branch를 `main` / `/ (root)`로 지정합니다.
3. 잠시 뒤 `https://내아이디.github.io/저장소이름/` 주소가 생깁니다.

## 3. 로그인 허용 도메인 추가 (중요)

Firebase 콘솔 **Authentication > 설정 > 승인된 도메인**에 `내아이디.github.io`를 추가하세요.
이걸 빼먹으면 로그인 팝업에서 `auth/unauthorized-domain` 오류가 납니다.

## 참고

- `firebaseConfig`의 apiKey는 공개돼도 괜찮은 값입니다. 데이터는 `firestore.rules`가 지켜서, 로그인한 본인 외에는 읽거나 쓸 수 없습니다.
- 나만 쓰려면 **Authentication > 설정 > 사용자 작업**에서 새 계정 가입을 막거나, 규칙의 `request.auth != null`을 `request.auth.token.email == '내이메일@gmail.com'`처럼 바꿔 특정 계정만 허용할 수 있습니다.
- 환율 API는 Frankfurter v1을 사용합니다. 공식 문서에 v1은 deprecated지만 계속 동작한다고 적혀 있어요. 나중에 막히면 `app.js`의 `API` 주소와 응답 처리만 v2에 맞게 고치면 됩니다.
- 은행 고시환율이나 실제 체결 환율과는 차이가 있을 수 있습니다. 매수 환율은 실제 체결된 값을 직접 입력하세요.
