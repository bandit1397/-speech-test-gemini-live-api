# -speech-test-gemini-live-api

Gemini **Live API**(WebSocket)로 만든 경찰 실시간 통역 **시험판**입니다.
기존 버전 [speech-test](https://github.com/bandit1397/speech-test)의 `app.html`(요청 방식)과 속도를 나란히 비교하기 위한 별도 저장소입니다. 기존 버전은 그대로 둡니다.

## 휴대폰 접속 주소 (GitHub Pages)

| 버전 | 주소 |
|---|---|
| **Live 시험판 (이 저장소)** | **https://bandit1397.github.io/-speech-test-gemini-live-api/app-live.html** |
| 기존 버전 (비교용) | https://bandit1397.github.io/speech-test/app.html |

- 루트 주소 https://bandit1397.github.io/-speech-test-gemini-live-api/ 로 열어도 app-live.html로 이동합니다.
- 푸시 후 반영까지 1~2분 걸립니다. 폰에 예전 화면이 보이면 새로고침하세요.
- 두 버전은 같은 도메인(bandit1397.github.io)을 쓰므로 Firebase 로그인 상태도 함께 유지됩니다.

## 사용 순서

1. 두 폰 모두 app-live.html을 엽니다.
2. ⚙️ 설정에서 Gemini API 키를 입력하고 Live 모델을 고릅니다. 기본값은 통역 전용 모델입니다.
3. Firebase에 로그인합니다. 기존 버전과 같은 계정입니다.
4. 두 폰의 방 코드를 똑같이 맞춥니다. 기본값은 `live-room`이고, 기존 버전의 `test-room`과 섞이지 않게 다르게 해 두었습니다.
5. 역할을 고릅니다. 이때 마이크와 Live 세션이 미리 연결됩니다. 외국인 폰에서는 외국인이 직접 언어를 고릅니다.
6. 🎤 버튼을 누르고 있는 동안 말하고, 떼면 끝납니다. 짧게 톡 누르면 다시 누를 때까지 녹음합니다.

## 통역 절차와 원리

### 기존 버전(app.html)과의 차이

| | 기존 app.html | Live app-live.html |
|---|---|---|
| 연결 | 말할 때마다 HTTP 요청 1회 | 역할 선택 때 WebSocket 1개를 열어 계속 사용 |
| 음성 전송 | 버튼을 **뗀 뒤** 녹음 전체를 한 번에 업로드 | **말하는 동안** 0.1초씩 바로 전송 |
| 번역 시작 | 업로드가 끝난 뒤 | 통역 전용 모델은 **말하는 도중에** 시작 (동시통역) |
| 지시문(용어집 등) | 사용 가능 | 통역 전용 모델은 불가, 일반 Live 모델은 가능 |

### 흐름 (통역 전용 모델 `gemini-3.5-live-translate-preview`)

```
[말하는 폰]                                        [듣는 폰]
① 역할 선택 → 마이크 열기 + Live 세션 연결
   (목표 언어 지정: 경찰 폰 → 외국인 언어, 외국인 폰 → ko)
② 버튼 누름 → 직전 0.3초 + 말소리를 0.1초씩
   16kHz PCM으로 WebSocket에 계속 전송
③ 말하는 도중 번역 자막(outputTranscription)이
   흘러나옴 → Firebase에 덮어씀 ───────────────→ 흐리게 표시 ("말하는 중 도착")
④ 버튼 뗌 → audioStreamEnd(음성 끝) 신호
⑤ 0.9초 동안 자막이 더 안 오면 완료로 확정 ─────→ 진하게 표시 + 진동 + 기록
   내 말 받아쓰기(inputTranscription)는 '내가 한 말'에 표시
```

- 응답은 번역 **음성**과 **자막**으로 오는데, 이 앱은 자막만 상대 폰에 보내고 음성은 쓰지 않습니다.
- 내 말이 이미 목표 언어이면 모델이 번역하지 않습니다(`echoTargetLanguage:false`).
- 지원 언어 코드: 중국어는 `zh-Hans`, 포르투갈어는 `pt-BR`을 쓰고, 나머지는 기존 코드와 같습니다. 20개 언어 모두 지원 목록에 있습니다.

### 일반 Live 모델 (`gemini-3.8-live` 등, 설정에서 선택)

- 자동 음성 감지를 끄고, 누를 때 `activityStart`, 뗄 때 `activityEnd`를 보냅니다.
- 지시문("모든 문장을 빠짐없이 번역")대로 번역 음성을 만들고, 그 자막을 상대에게 보냅니다. 서버가 `turnComplete`를 보내면 완료입니다.
- 말하는 동안 업로드가 끝나 있어서 기존 버전보다 빠를 것으로 기대하지만, 번역은 뗀 뒤에 시작합니다.

### 연결 관리

- Live 세션은 시간 제한(음성 세션 약 15분)이 있습니다. 끊기면 자동으로 다시 연결합니다(연속 5회까지). 서버가 곧 끊는다고 알리면(`goAway`) 발화가 끝난 뒤 새로 연결합니다.
- 외국인이 언어를 바꾸면 새 목표 언어로 세션을 다시 엽니다.
- 연결 준비 전에 녹음된 소리는 모아 두었다가 연결되면 한꺼번에 보냅니다.

### 측정값 (화면 아래 6칸)

| 항목 | 뜻 |
|---|---|
| 번역 첫 조각 (누른 후) | 버튼을 누른 순간 → 첫 번역 자막. "(말하는 중)"이면 떼기 전에 번역이 나온 것 |
| 번역 완료 (뗀 후) | 버튼을 뗀 순간 → 번역 확정 |
| Live 연결 준비 | WebSocket 연결 + 세션 설정 완료까지 |
| 전송 지연 (네트워크) | 상대 폰이 Firebase에 쓴 시각 → 내 폰이 받은 시각 |
| 수신: 첫 표시 / 완료 (말 끝→) | 상대가 버튼을 뗀 순간 → 내 화면에 처음 표시 / 완료까지. "말하는 중 도착"이면 상대가 말하는 도중에 이미 표시된 것 |

## 무료 등급 안내

- 요금표에는 Live 모델(`gemini-3.5-live-translate-preview`, `gemini-3.8-live`, `gemini-3.1-flash-live-preview`, `gemini-2.5-flash-native-audio-preview-12-2025`) 모두 무료 등급이 "Free of charge"로 나와 있습니다(2026-11 확인).
- Live API 한도는 요청 수가 아니라 **세션/동시 연결** 단위입니다. 정확한 값은 https://aistudio.google.com/rate-limit 에서 확인하세요.
- 무료 등급은 입력 음성이 Google 제품 개선에 쓰일 수 있습니다. 연습용 문장만 사용하세요.
- API 키는 브라우저에서 WebSocket 주소에 들어갑니다(기존 버전과 같은 수준의 노출). 실제 운영하려면 서버에서 임시 토큰(ephemeral token)을 발급하는 방식으로 바꿔야 합니다.

## 아직 확인 못 한 것 (실기기 시험 필요)

- 통역 전용 모델에 `audioStreamEnd`가 잘 작동하는지(뗀 뒤 남은 번역이 바로 나오는지)
- 자막 설정 위치: 문서 예시대로 `generationConfig` 안에 넣었고, 서버가 거부하면 자동으로 `setup` 바로 아래로 옮겨 다시 연결합니다. 기술 로그에서 확인할 수 있습니다.
- 긴 발화(미란다 고지 등)에서 번역이 빠지지 않는지

## 참고 문서

- Live API: https://ai.google.dev/gemini-api/docs/live-api
- Live 통역 모델: https://ai.google.dev/gemini-api/docs/live-api/live-translate
- 지원 언어 코드: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/gemini/3-5-live-translate
