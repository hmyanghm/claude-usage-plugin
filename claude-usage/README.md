# claude-usage — Claude Code 플러그인

터미널을 떠나지 않고 Claude 사용 한도를 확인하고, 메뉴바 앱을 설치·진단합니다.

```
/plugin marketplace add hmyanghm/claude-usage-plugin
/plugin install claude-usage
```

## 쓰는 법

| | |
|---|---|
| `/claude-usage` | 지금 한도 — 5시간·주간·모델 전용(Fable 등), 리셋까지 남은 시간, 한 줄 판정 |
| `/claude-usage team` | 팀원들의 한도 — 누가 여유 있는지 (앱에 팀 로그인 필요) |
| `/claude-usage install` | 메뉴바 앱 설치 (macOS 한 줄 · Windows 트레이 · 크롬 확장) |
| `/claude-usage status` | 설치·버전·팀 전송 진단 |
| `/claude-usage update` | 최신으로 갱신 |

`"한도 얼마나 남았어"`, `"지금 큰 작업 돌려도 돼?"` 처럼 물어도 됩니다.

## 왜 있나

메뉴바 앱은 화면 위쪽에 있고, 터미널에서 작업할 때는 눈이 거기 가지 않습니다.
긴 작업을 돌리기 전에 **"지금 돌려도 되나"** 를 그 자리에서 묻는 것이 이 플러그인의
용도입니다. 앱을 깔지 않아도 동작합니다 — Claude Code 가 이미 가진 인증으로
한도만 읽습니다.

## 무엇을 읽나

`api.anthropic.com/api/oauth/usage` 의 한도 수치(사용률·리셋 시각·플랜)뿐입니다.
대화 내용·프롬프트·파일은 읽지 않고, 어디로도 보내지 않습니다 — 이 플러그인에는
서버가 없습니다. 토큰 값은 화면에 찍지 않습니다.

## 알아 두실 점

이 도구는 Claude Code 가 저장한 OAuth 토큰으로 사용량 주소를 부릅니다. Anthropic
소비자 약관은 자동화된 접근을 API 키를 통한 경우로 한정하고 있어, 이 방식은 약관상
명확하지 않은 영역에 있습니다. 이 점을 알고 사용하시기 바랍니다.

Anthropic 과 제휴 관계가 없으며 보증받지 않았습니다.

---

메뉴바 앱·크롬 확장: <https://usage.diddk.kr> · 라이선스: <https://usage.diddk.kr/LICENSE.txt>
