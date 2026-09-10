---
name: claude-usage
description: Claude Code 사용 한도를 터미널에서 확인하거나 메뉴바 앱을 설치·진단한다. 사용자가 /claude-usage 를 치거나 "한도 얼마나 남았어", "지금 큰 작업 돌려도 돼?", "사용량 모니터 설치해줘" 라고 물을 때 쓴다.
---

# Claude 사용 한도

`$ARGUMENTS` 가 비어 있으면 **지금 상태**를 보여준다.
`team`·`install`·`status`·`update`·`help` 면 아래 해당 절로 간다.

🔴 **처음 쓰는 사람에게는 결과 끝에 한 줄을 붙인다** — 이 도구에 뭐가 더 있는지
   모르면 다시 안 쓴다. 이번 대화에서 이미 안내했으면 반복하지 않는다:
   `팀원 한도는 /claude-usage team · 메뉴바 앱은 /claude-usage install`

---

## 지금 상태 (인자 없음)

Claude Code 가 쓰는 것과 같은 OAuth 토큰으로 한도를 읽는다. 앱이 설치돼 있지 않아도 된다.

🔴 **macOS 전용이다.** 인증을 keychain 에서 읽는다. Windows·Linux 에서는
   "이 명령은 macOS 에서만 됩니다. Windows 는 트레이 앱을 쓰세요" 라고 답하고
   `install` 절을 안내한다 — 스크립트를 억지로 돌리지 말 것.

```bash
python3 - <<'PY'
import json, subprocess, urllib.request, datetime as dt

raw = subprocess.run(["security", "find-generic-password", "-w",
                      "-s", "Claude Code-credentials"],
                     capture_output=True, text=True).stdout.strip()
if not raw:
    print("Claude Code 로그인 정보를 찾지 못했습니다. 터미널에서 `claude` 로 로그인돼 있어야 합니다.")
    raise SystemExit
tok = json.loads(raw)["claudeAiOauth"]["accessToken"]
req = urllib.request.Request(
    "https://api.anthropic.com/api/oauth/usage",
    headers={"Authorization": f"Bearer {tok}", "Content-Type": "application/json",
             "anthropic-beta": "oauth-2025-04-20", "User-Agent": "claude-code/2.1.59"})
d = json.loads(urllib.request.urlopen(req, timeout=15).read())

def left(iso):
    if not iso: return ""
    t = dt.datetime.fromisoformat(iso.replace("Z", "+00:00")).astimezone()
    m = int((t - dt.datetime.now().astimezone()).total_seconds() // 60)
    if m <= 0: return "곧 리셋"
    return f"{m//60}시간 {m%60}분 뒤 리셋" if m >= 60 else f"{m}분 뒤 리셋"

for e in (d.get("limits") or []):
    kind = e.get("kind")
    name = {"session": "5시간", "weekly_all": "주간"}.get(kind)
    if not name:
        if kind != "weekly_scoped": continue
        name = ((e.get("scope") or {}).get("model") or {}).get("display_name") or "모델 전용"
    mark = " ←지금 병목" if e.get("is_active") else ""
    sev = {"critical": " [한도 도달]", "warning": " [빠듯]"}.get(e.get("severity"), "")
    print(f"{name:>8}  {e.get('percent'):>3}%  {left(e.get('resets_at')):<16}{sev}{mark}")
PY
```

읽은 뒤 **한 줄로 판정해서 말한다.** 숫자만 나열하지 말 것:

- 어느 한도든 `[한도 도달]` → ⛔ 막혔다. 리셋까지 얼마 남았는지.
- 5시간이 45분 안에 리셋되고 이미 70% 이상 → ⏳ 기다리는 게 낫다.
- `[빠듯]` 이 있으면 → ⚠️ 그 한도가 빠듯하다. 급하지 않으면 리셋 뒤에.
- 그 외 → ✅ 지금 돌려도 된다. 병목 한도의 여유와 리셋 시각을 곁들인다.

`←지금 병목` 이 붙은 것이 실제로 발목을 잡는 한도다. 그것을 기준으로 말한다.

판정 뒤에 위의 한 줄 안내를 붙인다(이번 대화에서 처음일 때만).

🔴 **모델 전용 한도(예: Fable)를 빠뜨리지 말 것.** 주간 전체가 남아 있어도 그것이
100% 면 그 모델을 못 쓴다. 별개의 벽이다.

토큰·비용까지 묻거든 `~/.claude/projects/*/*.jsonl` 의 `message.usage` 를 집계한다
(`stop_reason` 이 있는 최종 메시지만, `message.id` 로 중복 제거). 실측 기준
**한도 1%p ≈ API 환산 $1.6** 이지만 오차가 ±40% 라 "대략" 이라고 밝힌다.

---

## team — 팀원들의 한도

같은 팀 사람들이 지금 얼마나 썼는지 본다. **메뉴바 앱에 팀 로그인이 돼 있어야** 한다 —
앱이 키체인에 둔 세션을 빌려 쓴다. 팀을 안 쓰면 이 절은 건너뛴다.

```bash
python3 - <<'PY'
import json, os, subprocess, urllib.request, urllib.error, datetime as dt, pathlib

cfg_p = pathlib.Path.home() / ".claude" / "menubar_config.json"
if not cfg_p.exists():
    print("메뉴바 앱이 설치돼 있지 않습니다."); raise SystemExit
cfg = json.loads(cfg_p.read_text())
url, key = (cfg.get("team_url") or "").rstrip("/"), cfg.get("team_anon_key")
if not (url and key and cfg.get("team_joined")):
    print("팀에 참여하지 않았습니다. 혼자 쓰는 중이면 정상입니다."); raise SystemExit

# 🔴 계정(-a)을 반드시 지정한다. 같은 서비스에 옛 항목이 남아 있어, 생략하면
#    엉뚱한(만료된) 세션을 집는다.
acct = os.environ.get("USER", "claude-code-user")
raw = subprocess.run(["security", "find-generic-password", "-a", acct,
                      "-w", "-s", "Claude Usage Monitor - team"],
                     capture_output=True, text=True).stdout.strip()
if not raw:
    print("팀 로그인이 없습니다. 메뉴바 ⚙️ → 팀 → [팀 로그인…]"); raise SystemExit
s = json.loads(raw)

def api(tok, path):
    req = urllib.request.Request(url + path,
                                 headers={"apikey": key, "Authorization": "Bearer " + tok})
    with urllib.request.urlopen(req, timeout=15) as r:
        return json.loads(r.read())

tok = s.get("access_token")
try:
    rows = api(tok, "/rest/v1/team_latest?select=*")
except urllib.error.HTTPError:
    try:
        rq = urllib.request.Request(url + "/auth/v1/token?grant_type=refresh_token",
            data=json.dumps({"refresh_token": s.get("refresh_token")}).encode(),
            headers={"apikey": key, "Content-Type": "application/json"}, method="POST")
        tok = json.loads(urllib.request.urlopen(rq, timeout=15).read())["access_token"]
        rows = api(tok, "/rest/v1/team_latest?select=*")
    except Exception:
        print("팀 세션이 만료됐습니다. 메뉴바 ⚙️ → 팀 → [다시 로그인]"); raise SystemExit

if not rows:
    print("팀에 아직 아무도 수집되지 않았습니다."); raise SystemExit

now = dt.datetime.now(dt.timezone.utc)
def ago(iso):
    if not iso: return "기록 없음"
    t = dt.datetime.fromisoformat(iso.replace("Z", "+00:00"))
    m = int((now - t).total_seconds() // 60)
    if m > 180: return f"{m//60}시간 전"
    return "방금" if m < 2 else f"{m}분 전"

by_team = {}
for r in rows:
    by_team.setdefault(r.get("team_name") or "팀", []).append(r)
for team, members in by_team.items():
    print(f"[{team}]  {len(members)}명")
    members.sort(key=lambda r: -(r.get("five_hour_pct") or 0))
    for r in members:
        h5, d7 = r.get("five_hour_pct"), r.get("seven_day_pct")
        who = r.get("display_name") or (r.get("user_email") or "").split("@")[0]
        if h5 is None:
            print(f"   {who:<14} NO SIGNAL")
            continue
        stale = " (낡음)" if ago(r.get("collected_at")).endswith("시간 전") else ""
        print(f"   {who:<14} 5시간 {h5:>3.0f}%  주간 {d7 if d7 is not None else 0:>3.0f}%"
              f"   {ago(r.get('collected_at'))}{stale}")
PY
```

읽은 뒤 **질문에 답한다.** 표를 그대로 옮기지 말 것:

- "누가 여유 있어?" → 5시간이 가장 낮고 `NO SIGNAL`·`낡음` 이 아닌 사람.
- "지금 다 같이 돌려도 돼?" → 한 명이라도 90% 이상이면 그 사람을 짚어 말한다.
- `NO SIGNAL` 은 팀에는 있는데 아직 한 번도 안 올라온 사람이다(설치 안 함/로그인 안 함).
- `낡음`(3시간 이상 전)은 그 사람 앱이 꺼져 있다는 뜻이다. 지금 값이 아니라고 밝힌다.

🔴 **다른 사람의 수치는 그 사람 기기가 마지막으로 올린 값**이다. 실시간이 아니다.
   몇 분 전인지 함께 말한다.
🔴 팀 밖 사람의 데이터는 서버가 아예 내주지 않는다(행 수준 보안). 안 보인다고
   "권한 문제" 라고 추측하지 말 것 — 그 팀에 없는 것이다.

---

## install — 메뉴바 앱 설치

macOS 는 한 줄이다. Python 이 없어도 된다(런처에 들어 있다).

```bash
curl -fsSL https://usage.diddk.kr/install.sh | sh
```

설치되는 것: `~/Applications/Claude Usage Monitor.app`(서명·공증됨)과
`~/.claude-menubar/`.

🔴 **이 한 줄이 전부다. 뒤에 붙일 명령은 없다.** 설치가 끝나면 앱이 뜨고 로그인 시
   자동 실행도 켜져 있다. 예전 안내에 있던 `launchctl load …` 를 덧붙이면 앱이
   **두 개** 뜬다(설치 스크립트가 하나, launchd 가 하나) — 메뉴바 아이콘이 둘,
   API 폴링도 두 배라 레이트리밋에 두 배로 빨리 걸린다. 3.2.1 부터 앱이 스스로
   막지만, 애초에 안내하지 말 것.

   끄려면: 시스템 설정 › 일반 › 로그인 항목에서 **Claude Usage Monitor** 를 내린다.

**Windows** 는 트레이 앱이다. 두 파일을 같은 폴더에 받아 `setup_windows.bat` 를 실행한다
(Python 3.8+ 필요):
`https://usage.diddk.kr/dist/claude_menubar_windows.py`,
`https://usage.diddk.kr/dist/setup_windows.bat`

**크롬 확장**(선택) — claude.ai 화면 안에서 게이지를 본다:
`https://chromewebstore.google.com/detail/dlkabhklgnehcchbjhlfmilgilkfcgai`

🔴 **계정은 필요 없다.** 앱·확장 모두 로그인 없이 게이지가 돈다. 팀과 공유하거나
웹 대시보드를 볼 때만 계정을 만든다. 설치 안내에 "로그인하세요" 를 덧붙이지 말 것.

---

## status — 설치·전송 진단

```bash
echo "앱:      $(pgrep -f 'Claude Usage Monitor.app/Contents/MacOS' >/dev/null && echo '실행 중' || echo '꺼짐')"
echo "설치본:  $(ls -d ~/Applications/'Claude Usage Monitor.app' 2>/dev/null || echo '없음')"
echo "앱 코드: $(grep -m1 '^APP_VERSION' ~/.claude-menubar/claude_menubar.py 2>/dev/null | cut -d'"' -f2 || echo '없음')"
echo "최신:    $(curl -sL --max-time 10 https://usage.diddk.kr/version.json | python3 -c 'import json,sys;print(json.load(sys.stdin)["version"])' 2>/dev/null || echo '확인 실패')"
echo "팀 세션: $(security find-generic-password -a "$USER" -s 'Claude Usage Monitor - team' -w >/dev/null 2>&1 && echo '있음' || echo '없음(팀 미사용이면 정상)')"
tail -5 ~/.claude-menubar/stderr.log 2>/dev/null | grep -iE 'error|traceback' || true
```

- 앱 코드 버전과 최신이 다르면 → 업데이트하라고 안내(아래 `update`).
- 팀 세션이 있는데 앱 로그에 "다시 로그인" 이 보이면 → 다른 기기·확장에서
  로그아웃하면 서버가 토큰을 전부 폐기한다. **메뉴바 ⚙️ → 팀 → [다시 로그인]** 으로 푼다.

---

## update — 업데이트

```bash
curl -fsSL https://usage.diddk.kr/install.sh | sh
```

같은 스크립트가 갱신도 한다. 런처는 버전이 바뀔 때만 다시 받고, 앱 코드만 교체된다.
메뉴바 팝오버 아래 **버전 숫자를 눌러도** 그 자리에서 확인·설치할 수 있다.

---

## help — 무엇을 할 수 있나

아래를 **그대로 보여준다.** 실행할 것은 없다.

```
/claude-usage           지금 내 한도 — 5시간·주간·모델 전용(Fable 등)과 한 줄 판정
/claude-usage team      같은 팀 사람들의 한도 — 누가 여유 있는지
/claude-usage install   메뉴바 앱·트레이 앱·크롬 확장 설치
/claude-usage status    설치·버전·전송 상태 진단
/claude-usage update    앱을 최신으로
```

말로 물어도 된다:
- "한도 얼마나 남았어"
- "지금 큰 작업 돌려도 돼?"
- "누가 여유 있어?"  (팀)
- "사용량 모니터 설치해줘"

**알아 둘 것**
- 앱을 깔지 않아도 `/claude-usage` 는 된다. Claude Code 가 이미 가진 인증으로
  한도만 읽는다 — 서버가 없고, 대화·프롬프트는 읽지 않는다.
- `team` 만 메뉴바 앱의 팀 로그인이 필요하다. 앱이 키체인에 둔 세션을 빌려 쓴다.
- 메뉴바 앱(macOS)·트레이 앱(Windows)·크롬 확장은 각각 따로다. 아무것도
  안 깔아도 이 플러그인은 동작한다.

---

## 하지 말 것

- **사용자 대신 임의로 설치하지 않는다.** `install` 을 명시적으로 요청했을 때만 실행한다.
- 토큰 값을 화면에 찍지 않는다. 위 스크립트는 토큰을 출력하지 않는다 — 그대로 쓴다.
- 이 도구는 브라우저·로컬 세션으로 사용량을 읽는다. Anthropic 소비자 약관은 자동화된
  접근을 API 키로 한정하고 있어 **약관상 명확하지 않은 영역**에 있다. 사용자가 처음
  설치할 때 이 사실을 한 번 알린다.
