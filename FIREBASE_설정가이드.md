# Firebase 설정 가이드 (관리자 1회 설정)

이 앱은 영수증·예산 데이터를 **Firebase(클라우드)** 에 저장합니다.
한 번만 설정하면 전 임직원이 각자 로그인해서 사용하고, 캐시를 지우거나 폰을 바꿔도 데이터가 유지됩니다.

## 1. Firebase 프로젝트 만들기
1. https://console.firebase.google.com 접속 → **프로젝트 추가**
2. 프로젝트 이름 입력(예: `company-card`) → 생성

## 2. 로그인(인증) 켜기
1. 좌측 **빌드 → Authentication → 시작하기**
2. **이메일/비밀번호** 선택 → **사용 설정** → 저장

## 3. 데이터베이스 만들기
1. 좌측 **빌드 → Firestore Database → 데이터베이스 만들기**
2. 위치: **asia-northeast3 (서울)** 권장
3. **프로덕션 모드**로 시작 → 다음 단계에서 규칙을 아래로 교체

## 4. 보안 규칙 설정 (본인 데이터는 본인만)
Firestore → **규칙(Rules)** 탭에 아래를 붙여넣고 **게시**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null
        && request.auth.uid == userId
        && request.auth.token.email_verified == true
        && request.auth.token.email.matches('.*@lotte[.]net');
    }
  }
}
```

> - `@lotte[.]net` : **회사 이메일만** 데이터 접근(서버 강제)
> - `email_verified == true` : **이메일 인증을 마친 계정만** 접근 → 동료 이메일 선점 가입 방지
> (앱 화면의 도메인 제한·인증 게이트는 편의용이고, 실제 차단은 이 규칙이 담당)

## 4-1. Storage 만들기 (영수증 사진 저장)
영수증 사진은 Firestore가 아니라 **Storage**에 저장합니다(용량·비용 효율).
1. 좌측 **빌드 → Storage → 시작하기** → 위치 서울 권장
2. **규칙(Rules)** 탭에 아래를 붙여넣고 **게시**:

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /users/{userId}/{allPaths=**} {
      allow read, write: if request.auth != null
        && request.auth.uid == userId
        && request.auth.token.email_verified == true
        && request.auth.token.email.matches('.*@lotte[.]net')
        && request.resource.size < 2 * 1024 * 1024;   // 사진 2MB 제한
    }
  }
}
```

## 5. 웹 앱 등록 & 설정값 복사
1. 좌측 상단 ⚙ **프로젝트 설정 → 일반** 탭
2. "내 앱"에서 **웹 아이콘(`</>`)** 클릭 → 앱 닉네임 입력 → 등록
3. 표시되는 `firebaseConfig` 객체를 복사

## 6. 앱에 설정값 넣기
`index.html` 상단의 아래 부분을 복사한 값으로 교체 후 커밋·푸시:

```js
const firebaseConfig = {
  apiKey: "PASTE_API_KEY",
  authDomain: "PASTE_PROJECT.firebaseapp.com",
  projectId: "PASTE_PROJECT",
  storageBucket: "PASTE_PROJECT.appspot.com",
  messagingSenderId: "PASTE_SENDER_ID",
  appId: "PASTE_APP_ID"
};
```

## 7. 접속 도메인 허용
Authentication → **Settings → 승인된 도메인(Authorized domains)** 에
`sitacule.github.io` 가 없으면 **추가**. (로그인 동작에 필요)

---

### 참고
- 이메일/비밀번호는 직원이 직접 가입(회원가입 버튼)하거나, 관리자가 콘솔에서 추가할 수 있습니다.
  - 가입하면 **인증 메일**이 발송되고, **링크를 클릭해 인증을 마쳐야** 앱을 사용할 수 있습니다.
- `apiKey`는 공개돼도 괜찮습니다(Firebase 설계). 실제 보안은 위 **규칙**이 담당합니다.
- 무료(Spark) 요금제: 전사 규모라면 사용량 모니터링 후 **Blaze(종량제)** 전환을 권장합니다.

### 관리자/퇴사자 데이터 회수 (운영)
현재 앱은 **개인 도구**라 각자 본인 데이터만 봅니다. 회사가 특정 직원의 증빙을 회수해야 할 때:
1. (간편) 직원이 앱의 **💾 전체 백업(JSON)** + 사진을 내보내 제출
2. (정식) 관리자 계정에 **커스텀 클레임(admin)** 을 부여하고, 규칙에 관리자 읽기 허용 조항을 추가
   → 별도 관리자 화면이 필요하므로, 전사 운영 단계에서 확장 권장
- 퇴사 시: 콘솔에서 해당 계정 비활성/삭제 전에 위 방법으로 데이터를 먼저 확보하세요.
