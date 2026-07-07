# 백그라운드 상시 동작 · 배터리/절전 설정 (핵심 운영 문서)

이 앱은 **문자 수신 브로드캐스트(`SMS_RECEIVED`)로 깨어나는 방식**이다. 상주 포그라운드
서비스가 없으므로, 단말이 절전/딥슬립에 빠져 브로드캐스트를 못 받으면 **입금 문자를 통째로
놓친다.** 따라서 "이 앱이 절대 잠들지 않게" 만드는 것이 배포의 핵심이며, 아래를 반드시 확인·조치한다.

핵심 두 가지:

1. **OS(안드로이드) 도즈 예외** — 배터리 최적화 화이트리스트 등록. (adb/앱에서 처리 가능)
2. **제조사(삼성) 자체 절전 제외** — 초절전/절전 상태 앱 목록에서 제외 + 자동 절전 예외 앱 등록.
   **adb로 못 바꾼다. 화면에서 직접.**
   → 실무상 앱이 죽는 원인은 대부분 여기(삼성 절전)다. OS 예외만 해두고 이걸 빼먹으면 소용없다.

---

## 1. OS 레벨 — adb로 확인

`adb`는 `~/Library/Android/sdk/platform-tools/adb` (macOS 기준). 단말이 여러 대면 `-s <serial>` 지정.

```bash
PKG=kr.co.pincoin.paynotify
D=<serial>            # adb devices -l 로 확인

# ① 도즈(배터리 최적화) 화이트리스트 — 결과에 패키지가 나와야 정상
adb -s $D shell dumpsys deviceidle whitelist | grep paynotify
#   기대: user,kr.co.pincoin.paynotify,<uid>

# ② 앱 대기(App Standby) 버킷 — 5(EXEMPTED)가 최상
adb -s $D shell am get-standby-bucket $PKG
#   기대: 5    (5=EXEMPTED, 10=active, 20=working_set, 30=frequent, 40=rare, 45=restricted)

# ③ 백그라운드 실행 제한 — allow 여야 정상
adb -s $D shell cmd appops get $PKG RUN_ANY_IN_BACKGROUND
#   기대: Default mode: allow  (또는 allow)

# ④ 강제중지 여부 — stopped=false 여야 정상
adb -s $D shell dumpsys package $PKG | grep -E "stopped=|notLaunched="
#   기대: stopped=false ... notLaunched=false

# ⑤ 배터리 최적화 예외 권한
adb -s $D shell dumpsys package $PKG | grep BATTERY_OPT
#   기대: REQUEST_IGNORE_BATTERY_OPTIMIZATIONS: granted=true
```

### 조치 (OS 레벨)

- 화이트리스트에 **없으면** 앱 첫 실행 시 뜨는 "배터리 사용량 최적화 중지" 팝업에서 **허용**을 누른다.
  (앱이 `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`로 직접 요청함)
- adb로 강제 등록도 가능:
  ```bash
  adb -s $D shell dumpsys deviceidle whitelist +$PKG
  # SMS 런타임 권한도 adb로 부여 가능 (사이드로드 시)
  adb -s $D shell pm grant $PKG android.permission.RECEIVE_SMS
  adb -s $D shell pm grant $PKG android.permission.READ_SMS
  ```

---

## 2. 제조사(삼성 One UI) 레벨 — 화면에서 직접 확인·조치

**adb로는 확인/변경 불가.** 삼성 Device Care가 별도로 관리한다. 아래는 One UI 4.x(Android 12,
Galaxy A31 등) 기준 메뉴명이며, 버전에 따라 명칭이 조금 다를 수 있다.

삼성 절전 목록은 `설정 › 배터리 및 디바이스 케어 › 배터리 › 백그라운드 사용 제한` 한 화면에 모여 있다.

| # | 확인 항목 | 경로 | 올바른 상태 | 중요도 |
|---|---|---|---|---|
| ① | **앱 배터리 = 제한 없음** | `설정 › 애플리케이션 › paynotify › 배터리` | **"제한 없음"** 선택 (기본 "최적화됨" 아님) | ⭐필수 |
| ② | **초절전 상태 앱 제외** | `백그라운드 사용 제한 › 초절전 상태 앱` | 목록에 paynotify **없어야** 함 | ⭐필수 |
| ③ | **절전상태 앱 제외** | `백그라운드 사용 제한 › 절전상태 앱` | 목록에 paynotify **없어야** 함 | ⭐필수 |
| ④ | **자동 절전 예외 앱에 등록** | `백그라운드 사용 제한 › 자동 절전 예외 앱 › 추가 › paynotify` | 목록에 paynotify **있어야** 함 | ⭐권장 |
| ⑤ | **사용하지 않는 앱 자동 절전 OFF** | `백그라운드 사용 제한 › 사용하지 않는 앱을 절전 상태로 전환` | **끄기** | 권장 |
| ⑥ | **적응형 배터리 OFF** | `설정 › 배터리 및 디바이스 케어 › 배터리 › 기타 배터리 설정 › 적응형 배터리` | 꺼두면 더 안전 | 선택 |
| ⑦ | **자동 최적화 재시작 확인** | `설정 › 배터리 및 디바이스 케어 › ⋮ › 자동화` | "필요시 자동 재시작"이 앱을 잠재우지 않는지 | 선택 |

> **②③이 가장 중요하다.** 여기 목록에 들어가면 OS 도즈 예외(1번)를 해놨어도 문자 수신
> 브로드캐스트가 차단되어 앱이 아무 반응을 못 한다.
> **④(자동 절전 예외 앱 등록)는 미래 방지책이다** — 지금 절전에 안 걸려 있어도, OS 업데이트나
> 자동 최적화가 나중에 절전 토글을 되살리면 예외 목록에 있는 앱만 살아남는다. "현재 제외(②③)"와
> "미래 방지(④)"는 별개이므로 둘 다 해둔다.

### 조치 요령

- 초절전/절전 상태 앱 목록에 paynotify가 보이면 **길게 눌러 제거**한다.
- `자동 절전 예외 앱`에는 **추가(+)** 로 paynotify를 넣는다.
- ①의 "제한 없음"을 먼저 선택하면 이후 자동으로 절전 목록에 잘 안 들어간다.
- ※ 이미 ①(제한 없음)을 해둔 경우, 시스템이 앱을 이미 "안 재우는 앱"으로 취급하여
  `자동 절전 예외 앱 › 추가` 목록에 paynotify가 **안 뜰 수 있다 — 정상이며 추가 안 해도 된다.**
  (나중에 ①이 "최적화됨"으로 되돌아가면 그때 목록에 나타나므로 그때 추가한다.)

---

## 3. 부팅 후 자동 복구

- 권한: `RECEIVE_BOOT_COMPLETED` 보유. 부팅 완료 시 `MainActivity`가 자동 실행되고
  텔레그램으로 "부팅 완료"를 통보한다 (`Receiver.kt`).
- 재부팅되어도 다시 상시 대기로 복귀하도록 설계되어 있으나, **재부팅 후에도 위 2번(삼성 절전)
  설정은 유지**되는지 가끔 확인한다 (OS 업데이트/자동 최적화가 되돌리는 경우가 있음).

---

## 4. 최종 점검 체크리스트

배포/재설치 때마다 이 순서로 확인한다.

- [ ] `deviceidle whitelist`에 패키지 등록됨 (adb ①)
- [ ] standby 버킷 = 5 (adb ②)
- [ ] 백그라운드 실행 allow (adb ③)
- [ ] SMS 권한 granted=true
- [ ] 앱 배터리 "제한 없음" (화면 ①)
- [ ] 초절전 상태 앱 목록에 **없음** (화면 ②)
- [ ] 절전상태 앱 목록에 **없음** (화면 ③)
- [ ] 자동 절전 예외 앱에 paynotify **등록됨** (화면 ④)
- [ ] 사용하지 않는 앱 자동 절전 OFF (화면 ⑤)
- [ ] **실제 은행 문자 1통으로 엔드투엔드 전달 확인** ← 마지막에 반드시

---

## 참고 — 왜 이렇게까지 하나

- SMS 권한 앱은 공개 Play Store 배포가 막혀 있어(정책) 전용 단말에 사이드로드/관리형으로 깐다.
  그래서 제조사 절전을 무력화할 표준 MDM 채널이 항상 있는 게 아니라, **단말별 수동 확인**이 필요하다.
- 상주 포그라운드 서비스(+ 지속 알림)로 바꾸면 절전 내성이 더 강해지지만, 현재 구조는
  브로드캐스트 수신 방식이므로 위 절전 예외 설정이 사실상의 생명줄이다.
