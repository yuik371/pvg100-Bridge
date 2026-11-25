# pvg100-Bridge (PC 카카오톡 <-> 안드로이드 기기 양방향 알림 및 답장 브릿지 시스템 구현
2018년 11월쯤에 처음 사서... Palm PVG100을 지금까지 쓰다 카톡이 안되서 만들어버린 개빡대가리 벌레충 레파지토리

## 1. 프로젝트 개요 및 목적
나는 초소형 스마트폰 **Palm PVG100 (Android 8.1 Oreo, API 27, 3.3인치 화면, 800mAh 배터리)**을 메인으로 사용하던 사용자이다. 
하지만, 카카오톡 측에서 안드로이드 9 이상부터 지원한다고 발표한 이후 종료되어 사용하기 어려워서 메시지 앱을 개발한 후 보조 알림 및 간편 답장 단말기로 활용하고자 한다.
상시 켜져 있는 **Windows 서버 PC**에 설치된 PC 카카오톡의 새 메시지 알림을 실시간으로 감지하여 PVG100으로 푸시 알림을 보내고, PVG100의 상단 알림창에서 **인라인 빠른 답장(RemoteInput)** 및 **원터치 상용구 답장**을 입력하면 서버 PC의 해당 카카오톡 채팅방에 자동으로 메시지가 전송되는 양방향 브릿지 시스템을 처음부터 끝까지 완벽하게 구현해 달라.

---

## 2. 하드웨어 및 환경 제약 사항 (반드시 코드에 반영할 것)
1. **클라이언트 기기 (Palm PVG100)**:
   - OS: **Android 8.1 (API Level 27)** (`minSdk = 26`, `targetSdk = 27` 또는 호환 가능한 최신 설정)
   - 디스플레이: **3.3인치 (720 x 1280)** 초소형 화면이므로 복잡한 액티비티 UI 대신 **상단 알림바(Notification) 중심 UX**로 설계할 것.
   - 배터리: **800mAh** 초소형 배터리이므로 백그라운드 상시 소켓/폴링 연결을 절대 금지하며, 반드시 **Firebase Cloud Messaging (FCM)**을 통해서만 기기를 깨워 알림을 표시할 것.
2. **서버 PC (Windows 10/11)**:
   - PC 카카오톡이 로그인되어 실행 중이며, 알림 미리보기가 켜져 있는 상태.
   - 외부 망(핫스팟)에서 서버 PC로 안전하게 답장 요청을 보낼 수 있도록 **Cloudflare Tunnel (`cloudflared`)** 또는 로컬 HTTP API 연동 구조를 포함할 것.

---

## 3. 시스템 아키텍처 요구사항

```text
[Windows PC 카카오톡]
       │ (1. Windows Notification Listener API로 토스트 알림 감지)
       ▼
[Python 서버 에이전트] ──(2. firebase-admin SDK로 FCM 발송)──► [Firebase (FCM)]
       ▲                                                              │
       │ (4. HTTP POST /reply 수신 및 pywinauto 자동 입력)            │ (3. 백그라운드 푸시 수신)
       │                                                              ▼
[PC 카톡 채팅방 전송] ◄──(HTTPS / Cloudflare Tunnel)────── [PVG100 전용 Android 앱]
```

---

## 4. 세부 기능 명세

### Part A: Windows 서버 PC 에이전트 (Python)
1. **비동기 병렬 실행 구조**:
   - **알림 감지 워커**: Windows WinRT API(`winsdk`의 `UserNotificationListener`)를 사용하여 카카오톡(`KakaoTalk`)에서 발생하는 알림의 발신자(또는 채팅방 이름)와 메시지 본문을 실시간으로 추출한다. 중복 알림 필터링 로직을 포함한다.
   - **FCM 발송 모듈**: `firebase-admin` SDK를 통해 추출된 `room_name`, `sender`, `message`, `timestamp` 데이터를 PVG100 기기의 FCM 토큰으로 전송한다.
   - **답장 수신 API 서버 (`FastAPI` + `uvicorn`)**: `POST /reply` 엔드포인트를 열고, 보안 검증용 `X-API-KEY` 헤더를 확인한다.
2. **카카오톡 자동 답장 매크로 (`pywinauto` / `win32gui` / `pyperclip`)**:
   - `room_name`과 일치하는 열려 있는 카카오톡 채팅방 창을 찾는다. (창이 닫혀 있을 경우 카카오톡 메인 창 검색창을 통해 방을 여는 폴백 로직도 포함하면 가산점)
   - 해당 창을 포커스한 뒤 클립보드(`pyperclip`)에 답장 텍스트를 복사하고 `Ctrl+V` 및 `Enter` 키를 전송하여 한글 깨짐 없이 안정적으로 발송한다.
   - 발송 성공/실패 여부를 JSON 응답으로 반환한다.

### Part B: Palm PVG100 전용 안드로이드 앱 (Kotlin)
1. **프로젝트 설정 및 의존성**:
   - Android 8.1(API 27)에서 오류 없이 빌드 및 실행되는 `build.gradle.kts` (또는 `build.gradle`) 및 `AndroidManifest.xml` 전체 코드를 제공한다.
   - 배터리 최적화 제외 요청 및 알림 채널(`IMPORTANCE_HIGH`) 생성 로직을 포함한다.
2. **메인 화면 (`MainActivity.kt`)**:
   - 3.3인치 화면에 한눈에 들어오는 심플한 설정 UI:
     - 현재 기기의 **FCM Registration Token** 표시 및 원클릭 복사 버튼
     - 서버 PC API 주소(`Server URL`, 예: Cloudflare Tunnel 주소) 및 `API Key` 입력·저장(`SharedPreferences`) 필드
     - 서버 연결 테스트(`GET /health`) 버튼
3. **푸시 수신 및 초소형 화면 최적화 알림 (`MyFirebaseMessagingService.kt`)**:
   - FCM 데이터 메시지를 수신하면 동일한 `room_name`별로 알림을 그룹화하거나 업데이트하여 알림창이 지저분해지지 않게 관리한다.
   - **인라인 답장 (`RemoteInput`)**: 알림창에서 바로 키보드를 열어 텍스트를 입력하고 전송할 수 있는 `[답장]` 액션을 추가한다.
   - **원터치 상용구 답장 액션**: 작은 화면에서 타이핑을 최소화할 수 있도록 `[확인]`, `[잠시만요]` 원터치 즉시 전송 버튼을 알림 액션에 함께 배치한다.
4. **백그라운드 답장 전송 (`ReplyReceiver.kt`)**:
   - `BroadcastReceiver`에서 인라인 입력 텍스트 또는 상용구 텍스트를 받아 `OkHttp` (또는 `Retrofit` + Coroutines)를 이용해 서버 PC의 `POST /reply`로 비동기 전송한다.
   - 전송 성공 시 알림을 자동으로 갱신(또는 제거)하고, 실패 시 사용자에게 실패 알림/토스트를 띄운다.

---

## 5. 출력 요청 형식
생략(placeholders like `// TODO: implement here`) 없이 **바로 복사해서 실행 및 빌드할 수 있는 완전한 프로덕션 수준의 전체 코드**를 다음 순서대로 작성해 달라.

1. **전체 디렉토리 및 파일 트리 구조**
2. **Firebase 프로젝트 생성 및 `serviceAccountKey.json` / `google-services.json` 발급·배치 가이드**
3. **Part A: Python 서버 전체 코드 (`requirements.txt`, `config.py`, `main.py`)**
4. **Part B: Android 앱 전체 코드 (`AndroidManifest.xml`, `build.gradle.kts`, `MainActivity.kt`, `MyFirebaseMessagingService.kt`, `ReplyReceiver.kt`, XML 레이아웃)**
5. **실행 및 외부 접속(Cloudflare Tunnel) 테스트 체크리스트**
