# -speech-test-gemini-live-api

Gemini **Live API**(WebSocket)로 만든 경찰 실시간 통역 **시험판**입니다.
기존 버전 [speech-test](https://github.com/bandit1397/speech-test)의 `app.html`(요청 방식)과 속도를 나란히 비교하기 위한 별도 저장소입니다. 기존 버전은 그대로 둡니다.

## 시험 결과

- **2026-11-11:** 실제 폰 두 대로 정상 작동을 확인했습니다. 기본 모델(`gemini-3.5-live-translate-preview`)을 썼고, **무료 등급(결제 없음)에서 그대로 쓸 수 있습니다.**

## 휴대폰 접속 주소 (GitHub Pages)

| 버전 | 주소 |
|---|---|
| **Live 시험판 (이 저장소)** | **https://bandit1397.github.io/-speech-test-gemini-live-api/app-live.html** |
| 기존 버전 (비교용) | https://bandit1397.github.io/speech-test/app.html |

- 루트 주소 https://bandit1397.github.io/-speech-test-gemini-live-api/ 로 열어도 app-live.html로 이동합니다(`index.html`이 바로 넘겨 줌).
- 주소의 `io/` 바로 뒤에 하이픈(`-`)이 있습니다. 저장소 이름이 `-`로 시작하기 때문입니다.
- 푸시 후 반영까지 1~2분 걸립니다. 폰에 예전 화면이 보이면 새로고침하세요.
- 두 버전은 같은 도메인(bandit1397.github.io)을 쓰므로 Firebase 로그인 상태도 함께 유지됩니다.

## 사용 순서

1. 두 폰 모두 app-live.html을 엽니다.
2. ⚙️ 설정에서 Gemini API 키를 입력하고 Live 모델을 고릅니다. 기본값은 통역 전용 모델입니다. 키는 저장되지 않으므로 페이지를 열 때마다 입력합니다.
3. Firebase에 로그인합니다. 기존 버전과 같은 계정이고, 한 번 하면 브라우저에 유지됩니다.
4. 두 폰의 방 코드를 똑같이 맞춥니다. 기본값은 `live-room`이고, 기존 버전의 `test-room`과 섞이지 않게 다르게 해 두었습니다.
5. 역할을 고릅니다. 이때 마이크와 Live 세션이 미리 연결됩니다. 외국인 폰에서는 외국인이 직접 언어를 고릅니다.
6. 🎤 버튼을 누르고 있는 동안 말하고, 떼면 끝납니다. 짧게 톡 누르면(0.4초 미만) 다시 누를 때까지 녹음합니다.

---

## 통역 절차와 원리

### 기존 버전(app.html)과의 차이

| | 기존 app.html | Live app-live.html |
|---|---|---|
| 연결 | 말할 때마다 HTTP 요청 1회 (`streamGenerateContent`) | 역할 선택 때 WebSocket 1개를 열어 계속 사용 (`BidiGenerateContent`) |
| 음성 전송 | 버튼을 **뗀 뒤** 녹음 전체를 WAV로 한 번에 업로드 | **말하는 동안** 0.1초씩 16kHz PCM으로 바로 전송 |
| 번역 시작 | 업로드가 끝난 뒤 | 통역 전용 모델은 **말하는 도중에** 시작 (동시통역) |
| 받아쓰기 | 모델 출력 2줄째 | 서버의 `inputTranscription` |
| 번역문 | 모델 출력 1줄째 | 서버의 `outputTranscription` (번역 음성의 자막) |
| 지시문(용어집 등) | 사용 가능 | 통역 전용 모델은 불가, 일반 Live 모델은 가능 |
| 무료 한도 | 요청 수 (분당·하루) | 세션/동시 연결 단위 |
| 오디오 처리 | ScriptProcessor | AudioWorklet (메인 스레드와 분리) |

### 전체 구조

```
 [경찰 폰 A]                    Firebase Realtime DB                  [외국인 폰 B]
  마이크 ─▶ AudioWorklet           (asia-southeast1)                 AudioWorklet ◀─ 마이크
            │ 0.1초 조각                                                    │
            ▼                                                                ▼
  Gemini Live 세션 (WebSocket)                                 Gemini Live 세션 (WebSocket)
  목표 언어 = 외국인 언어                                        목표 언어 = ko
            │ 번역 자막                                                     │ 번역 자막
            ▼                                                                ▼
     rooms/{방}/AtoB  ──────────────▶ B 화면에 표시      A 화면에 표시 ◀────────────── rooms/{방}/BtoA
                          rooms/{방}/meta.bLang  ◀── B가 고른 언어 (A가 읽어서 세션 언어를 맞춤)
```

- **폰마다 Live 세션을 하나씩** 엽니다. 각 폰은 자기 말만 자기 세션으로 보내고, 번역 결과(텍스트)만 Firebase로 상대 폰에 넘깁니다.
- 상대 폰은 Gemini를 부르지 않고 Firebase를 구독하기만 합니다.

### 흐름 (통역 전용 모델 `gemini-3.5-live-translate-preview`, 기본값)

```
[말하는 폰]                                        [듣는 폰]
① 역할 선택 → 마이크 열기 + Live 세션 연결
   (목표 언어 지정: 경찰 폰 → 외국인 언어, 외국인 폰 → ko)
② 버튼 누름 → 직전 0.3초 + 말소리를 0.1초씩
   16kHz PCM으로 WebSocket에 계속 전송
③ 말하는 도중 번역 자막(outputTranscription)이
   흘러나옴 → Firebase에 덮어씀 ───────────────→ 흐리게 표시 ("말하는 중 도착")
④ 버튼 뗌 → 0.12초 뒤(마지막 조각 전송 후) audioStreamEnd 신호
⑤ 0.9초 동안 자막이 더 안 오면 완료로 확정 ─────→ 진하게 표시 + 진동 + 기록
   내 말 받아쓰기(inputTranscription)는 '내가 한 말'에 표시
```

- 응답은 번역 **음성**(24kHz PCM)과 **자막**으로 오는데, 이 앱은 자막만 상대 폰에 보내고 음성은 버립니다.
- 내 말이 이미 목표 언어이면 모델이 번역하지 않습니다(`echoTargetLanguage:false`).
- 이 모델은 말의 원래 언어를 스스로 알아내므로 원래 언어는 지정하지 않습니다.
- 지원 언어 코드: 중국어는 `zh-Hans`, 포르투갈어는 `pt-BR`을 쓰고, 나머지는 기존 코드와 같습니다(`LANGS[...].live`). 20개 언어 모두 지원 목록에 있습니다.

### 흐름 (일반 Live 모델: `gemini-3.8-live` 등, 설정에서 선택)

1. 자동 음성 감지를 끕니다(`realtimeInputConfig.automaticActivityDetection.disabled: true`).
2. 누르면 `activityStart`, 떼면 `activityEnd`를 보내 발화 경계를 앱이 정합니다.
3. 지시문(`sysPrompt()`: "모든 문장을 빠짐없이 번역하고, 대답하거나 요약하지 말 것")대로 번역 음성을 만들고, 그 자막을 상대에게 보냅니다.
4. 서버가 `turnComplete`(또는 `generationComplete`)를 보내면 완료로 확정합니다. 자막이 2.5초 동안 끊기거나 8초 동안 아무것도 오지 않으면 그대로 확정합니다.

- 번역은 버튼을 뗀 뒤 시작하지만, 업로드는 이미 끝나 있습니다.
- 한 세션 안에서 대화 맥락이 쌓입니다.

### 한 번의 발화(utt) 처리

`startTalk()`에서 발화 객체 `cur`를 만들고, 그 발화의 번역 자막이 올 때마다 Firebase의 **같은 id에 덮어씁니다**.

| 필드 | 뜻 |
|---|---|
| `id` | `A-1731300000000-ab12` 형식. 상대 폰은 id가 바뀌면 새 발화로 보고 진동·기록 추가 |
| `t0` / `tRel` | 누른 시각 / 뗀 시각 (performance.now, 내 폰 측정용) |
| `rel` | 뗀 시각 (서버 기준 ms). 상대 폰이 "말 끝→표시" 지연을 계산. 떼기 전에는 0 |
| `out` / `inTr` | 지금까지 받은 번역 자막 / 내 말 받아쓰기 (조각을 이어 붙임) |
| `done` | 확정 여부. 확정 뒤 늦게 온 자막도 `done:true`로 다시 덮어써 반영 |
| `stopping` | 떼기 처리 중복 방지 |

- 새로 누를 때 이전 발화가 아직 확정 전이면, 그때까지 받은 번역으로 즉시 확정합니다.
- 이전 발화의 늦은 꼬리 자막은 새 발화 쪽에 붙을 수 있습니다(알려진 한계).

### 연결 관리

- **미리 연결:** 역할을 고를 때, 그리고 화면이 다시 보일 때 Live 세션을 연결합니다. 결과는 측정값 "Live 연결 준비"에 표시됩니다.
- **연결 기준:** 모델·키·내 언어·상대 언어(`sig()`)가 바뀌면 세션을 새로 엽니다. 외국인이 언어를 바꾸면 `liveReconnectSoon()`이 말이 끝난 뒤 다시 연결합니다.
- **연결 전 소리:** `wsQueue`에 모아 두었다가 `setupComplete`를 받으면 한꺼번에 보냅니다. 최대 600조각, 약 60초입니다.
- **끊김:** 세션 시간 제한(음성 세션 약 15분)이나 네트워크 문제로 끊기면 1초, 2초, …… 간격으로 자동 재연결합니다(연속 5회까지). 다만 키 오류·한도 초과·형식 오류(코드 1007/1008, "quota" 등)면 재연결하지 않고 오류를 표시합니다.
- **goAway:** 서버가 곧 끊는다고 알리면, 말하는 중이 아닐 때 바로, 말하는 중이면 발화가 끝난 뒤 새로 연결합니다.
- **자막 설정 위치 보정:** 문서마다 `inputAudioTranscription`/`outputAudioTranscription` 위치가 달랐습니다. 그래서 먼저 `generationConfig` 안에 넣고, 서버가 1007로 거부하면 `setup` 바로 아래로 옮겨 다시 연결합니다(`trPlace`). 이렇게 바뀌면 기술 로그에 "자막 설정 위치 변경 후 재연결"이 남습니다.

### 오디오 처리

1. `getUserMedia`로 마이크를 엽니다. 에코 제거, 잡음 제거, 자동 음량을 켭니다.
2. **AudioWorklet**(`pcm-chunker`)이 기기 샘플레이트(보통 48kHz) 기준으로 정확히 0.1초씩 잘라 메인 스레드로 넘깁니다.
3. `toPcm16()`이 16kHz로 줄이고(구간 평균) 16bit little-endian PCM으로 바꾼 뒤 base64로 인코딩합니다.
4. 누르지 않을 때도 최근 3조각(0.3초)을 보관해 두었다가(`preroll`), 누르면 먼저 보냅니다. 그래서 첫 음절이 잘리지 않습니다.
5. 워클릿 출력은 음량 0인 GainNode를 거쳐 스피커에 연결합니다. 연결해야 처리가 멈추지 않고, 소리는 나지 않습니다.

---

## 개발 참고 자료

### 파일 구성

| 파일 | 내용 |
|---|---|
| `app-live.html` | 앱 전체 (HTML·CSS·JS 한 파일, 빌드 과정 없음) |
| `index.html` | app-live.html로 넘겨 주는 페이지 |
| `README.md` | 이 문서 |

- `UI`(20개 언어 화면 문구)와 `LANG_BTN`(국기 + 언어 이름)은 기존 `speech-test/app.html`에서 그대로 복사해 왔습니다. 문구를 고치면 두 저장소 모두 고쳐야 합니다.
- Firebase DB 규칙 파일(`database.rules.json`)은 **speech-test 저장소에 있습니다.** 두 앱이 같은 규칙을 씁니다.

### 코드 지도 (app-live.html 안의 주요 함수)

| 영역 | 함수/변수 | 하는 일 |
|---|---|---|
| 설정 | `config`, `LANGS`, `UI`, `LANG_BTN` | Firebase 설정, 언어 정보(이름·Live 코드), 화면 문구 |
| Firebase | `listen()` | 상대 방향 경로(`AtoB`/`BtoA`)와 `meta` 구독, 수신 측정값 계산 |
| | `publishLang()` | 외국인 폰이 고른 언어를 `meta`에 씀 |
| 역할/화면 | `choose(r)`, `applyUI()`, `showLang()` | 역할 선택(마이크·세션 미리 연결), 화면 언어 적용 |
| Live 연결 | `setupMsg()` | 첫 메시지(setup) 만들기. 통역 전용과 일반 모델에 따라 내용이 다름 |
| | `liveConnect()`, `liveClose()` | WebSocket 연결·종료, 재연결, 오류 처리 |
| | `liveReconnectSoon()` | 언어가 바뀌면 말이 끝난 뒤 다시 연결 |
| | `wsSend(obj)` | 보내기. 연결 전이면 `wsQueue`에 모음 |
| 서버 응답 | `onServer(m)` | 자막·받아쓰기·turnComplete·goAway 처리 |
| | `push(u,t,done)` | Firebase에 번역 쓰기 |
| | `armIdle(u)`, `finish(u)` | 뗀 뒤 완료 판정 타이머, 완료 확정(기록·측정값) |
| 오디오 | `WORKLET`, `micInit()`, `toPcm16()`, `onChunk()` | 마이크 → 0.1초 조각 → 16kHz PCM base64 → 전송 또는 preroll 보관 |
| 버튼 | `startTalk()`, `stopTalk()`, pointer 이벤트 | 누르고 말하기 / 톡 눌러 켜고 끄기 |

### Live API 메시지 형식 (실제로 보내는 것)

접속 주소 (키는 쿼리로 전달):
```
wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent?key=API_KEY
```

setup, 통역 전용 모델 (경찰 폰, 외국인 언어가 영어일 때):
```json
{"setup":{
  "model":"models/gemini-3.5-live-translate-preview",
  "generationConfig":{
    "responseModalities":["AUDIO"],
    "inputAudioTranscription":{},
    "outputAudioTranscription":{},
    "translationConfig":{"targetLanguageCode":"en","echoTargetLanguage":false}
  }}}
```

setup, 일반 Live 모델:
```json
{"setup":{
  "model":"models/gemini-3.8-live",
  "generationConfig":{"responseModalities":["AUDIO"],"inputAudioTranscription":{},"outputAudioTranscription":{}},
  "systemInstruction":{"parts":[{"text":"You are a live interpreter ..."}]},
  "realtimeInputConfig":{"automaticActivityDetection":{"disabled":true}}
}}
```

보내는 메시지:

| 시점 | 메시지 |
|---|---|
| 0.1초마다 | `{"realtimeInput":{"audio":{"data":"<base64>","mimeType":"audio/pcm;rate=16000"}}}` |
| 누름 (일반 모델만) | `{"realtimeInput":{"activityStart":{}}}` |
| 뗌 (일반 모델) | `{"realtimeInput":{"activityEnd":{}}}` |
| 뗌 (통역 전용) | `{"realtimeInput":{"audioStreamEnd":true}}` |

받는 메시지 (서버는 바이너리 프레임으로 보낼 때가 있어 `binaryType="arraybuffer"` + TextDecoder로 읽음):

| 필드 | 사용 |
|---|---|
| `setupComplete` | 연결 완료 → 모아 둔 소리 전송 |
| `serverContent.outputTranscription.text` | 번역 자막 조각 → 이어 붙여 Firebase로 |
| `serverContent.inputTranscription.text` | 내 말 받아쓰기 조각 → '내가 한 말' |
| `serverContent.modelTurn.parts[].inlineData` | 번역 음성(24kHz PCM) → 현재는 버림 |
| `serverContent.turnComplete` / `generationComplete` | 일반 모델의 발화 완료 |
| `goAway` | 곧 연결 종료 예고 → 재연결 준비 |

### Firebase 데이터 형식 (기존 버전과 같음)

```
rooms/{방코드}/AtoB   경찰 → 외국인 (마지막 발화 하나만 덮어씀)
rooms/{방코드}/BtoA   외국인 → 경찰
  { id, text, done, src, tgt, relMs, sentMs }
rooms/{방코드}/meta   { bLang }   외국인이 고른 언어
```

| 필드 | 형식 | 뜻 |
|---|---|---|
| `id` | 문자열 (<64) | 발화 id |
| `text` | 문자열 (<2000) | 번역문 (앱에서 1990자로 자름) |
| `done` | 불린 | false=작성 중(흐리게), true=확정 |
| `src` / `tgt` | 문자열 (<8) | 원래 언어 / 번역 언어 코드 (`ko`, `en`, `zh` 등 앱 내부 코드) |
| `relMs` | 숫자 | 말한 사람이 버튼을 뗀 시각 (서버 기준 ms), 떼기 전이면 0 |
| `sentMs` | 숫자 | 이 값을 쓴 시각 (서버 기준 ms) |

- **DB 규칙:** 정해진 계정으로 로그인해야 읽고 쓸 수 있고, 위 필드 외에는 쓸 수 없습니다(`$other: false`).
- **새 필드를 추가하려면:** speech-test의 `database.rules.json`을 고친 뒤 Firebase 콘솔(Realtime Database → 규칙)에 반영해야 합니다. 반영하지 않으면 쓰기가 거부됩니다.
- 시간은 `.info/serverTimeOffset`으로 서버 시각에 맞춰, 두 폰 시계가 달라도 지연을 잴 수 있게 했습니다.

### 측정값 (화면 아래 6칸)

| 항목 | 뜻 |
|---|---|
| 번역 첫 조각 (누른 후) | 버튼을 누른 순간 → 첫 번역 자막. "(말하는 중)"이면 떼기 전에 번역이 나온 것 |
| 번역 완료 (뗀 후) | 버튼을 뗀 순간 → 번역 확정. 통역 전용은 마지막 자막 뒤 0.9초 대기가 포함됨 |
| Live 연결 준비 | WebSocket 연결 + 세션 설정 완료까지 |
| 전송 지연 (네트워크) | 상대 폰이 Firebase에 쓴 시각 → 내 폰이 받은 시각 |
| 수신: 첫 표시 / 완료 (말 끝→) | 상대가 버튼을 뗀 순간 → 내 화면에 처음 표시 / 완료까지. "말하는 중 도착"이면 상대가 말하는 도중에 이미 표시된 것 |

### 기술 로그로 문제 찾기

| 로그 | 뜻 / 조치 |
|---|---|
| `Live 세션 준비 (0.xx초) · Korean → English` | 정상 연결 |
| `Live 연결 종료 (코드 1007 · ...)` | 보낸 형식이 거부됨 → 이유 문구를 보고 `setupMsg()` 수정 |
| `Live 연결 종료 (코드 1008 · ...)` | 키·권한 문제 또는 모델 이름 오류 |
| `... quota / exceeded ...` | 무료 한도 초과 → 잠시 후, 또는 다른 Google 계정 키 사용 |
| `자막 설정 위치 변경 후 재연결` | 자막 설정이 `setup` 바로 아래로 바뀜 (자동 보정) |
| `서버가 곧 연결을 끊음` | 세션 시간 제한 → 자동으로 새로 연결 |
| `번역 결과 없음` | 소리가 작거나, 이미 목표 언어로 말했음(통역 전용은 이 경우 조용함) |

### 설계 결정과 이유

| 결정 | 이유 |
|---|---|
| 기존 앱을 고치지 않고 별도 저장소로 만듦 | 정상 작동하는 기존 버전을 지키면서 나란히 비교하기 위해 |
| 같은 Firebase 프로젝트, 다른 방 코드 | 로그인·규칙을 그대로 쓰고 데이터는 섞이지 않게 |
| 자막만 보내고 번역 음성은 버림 | 기존 앱과 같은 화면 방식. 음성을 상대 폰으로 보내려면 별도 전송이 필요함 |
| 응답 방식을 `AUDIO`로 고정 | 통역 전용과 native-audio 모델은 AUDIO만 지원하고, 텍스트는 자막으로 받음 |
| 뗀 뒤 0.9초 조용하면 완료 | 통역 전용 모델은 "발화 끝" 신호가 따로 없어서 |
| 지원 안 되는 브라우저 대비 없음 | 시험판이라 최신 크롬 기준 (AudioWorklet 필요) |

---

## 무료 등급 안내

- **2026-11-11 실기기 시험에서 결제 없이 정상 작동했습니다.**
- 요금표에는 Live 모델(`gemini-3.5-live-translate-preview`, `gemini-3.8-live`, `gemini-3.1-flash-live-preview`, `gemini-2.5-flash-native-audio-preview-12-2025`) 모두 무료 등급이 "Free of charge"로 나와 있습니다(2026-11 확인).
- Live API 한도는 요청 수가 아니라 **세션/동시 연결** 단위입니다. 정확한 값은 https://aistudio.google.com/rate-limit 에서 확인하세요.
- 유료로 바꿀 경우 요금표 기준 단가(2026-11 확인, 바뀔 수 있음):
  - 통역 전용 모델: 음성 입력 약 $0.0053/분, 음성 출력 약 $0.0315/분
  - 일반 Live 모델: 음성 입력 약 $0.005/분, 음성 출력 약 $0.018/분
- 무료 등급은 입력 음성이 Google 제품 개선에 쓰일 수 있습니다. 연습용 문장만 사용하세요.
- API 키는 브라우저에서 WebSocket 주소에 들어갑니다(기존 버전과 같은 수준의 노출). 실제 운영하려면 서버에서 임시 토큰(ephemeral token)을 발급하는 방식으로 바꿔야 합니다. 이때 접속 주소는 `BidiGenerateContentConstrained?access_token=...`입니다.

## 다음에 개발할 것 (후보)

| 후보 | 내용 | 참고 |
|---|---|---|
| 고지문 버튼 | 미란다 고지 등 정해진 문장을 20개 언어로 미리 번역해 두고 버튼으로 표시 | 정확도 100%, 즉시 표시, Gemini 사용 0. 통역 전용 모델은 용어집을 넣을 수 없어서 특히 유용 |
| 번역 음성 재생 | 받은 번역 음성(24kHz)을 상대 폰에서 들려주기 | 음성을 Firebase로 보내기엔 커서, 듣는 폰에서 TTS로 읽거나 WebRTC로 보내는 방안 검토 |
| 꼬리 자막 정확히 나누기 | 이전 발화의 늦은 자막이 새 발화에 붙는 문제 | 발화마다 경계 표시가 필요 |
| 임시 토큰 | API 키를 브라우저에 두지 않기 | 서버(Firebase Functions 등) 필요, Blaze 요금제 |
| 기존 버전과 통합 | 속도·정확도 비교 후 app.html에 Live 방식을 선택 항목으로 넣기 | 비교 결과를 이 README에 기록 |
| 긴 발화 정확도 | 일반 Live 모델 + 용어집 지시문과 통역 전용 모델 비교 | 기존 README의 "나중에 고칠 것" 참고 |

## 참고 문서

- Live API: https://ai.google.dev/gemini-api/docs/live-api
- Live API WebSocket 시작하기: https://aistudio.google.com/docs/live-api/get-started-websocket
- Live API 기능(VAD, 자막, 세션 제한): https://aistudio.google.com/docs/live-api/capabilities
- Live 통역 모델: https://ai.google.dev/gemini-api/docs/live-api/live-translate
- 지원 언어 코드: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/gemini/3-5-live-translate
- 요금: https://aistudio.google.com/docs/pricing
- 무료 한도 확인: https://aistudio.google.com/rate-limit
