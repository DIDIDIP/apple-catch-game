# 🎮 HTML 웹 게임 만들고 GitHub Pages로 배포하기

> **강의 노트** · 2026-10-07
> HTML 파일 하나로 브라우저 게임을 만들고, GitHub 저장소에 올린 다음, GitHub Pages로 누구나 접속할 수 있게 배포하는 과정을 정리했습니다.

---

## 📌 오늘의 결과물

| 게임 | 바로 플레이 | 저장소 |
|---|---|---|
| 🍎 사과 받기 | https://dididip.github.io/apple-catch-game/ | https://github.com/DIDIDIP/apple-catch-game |
| 🍉 과일 합치기 | https://dididip.github.io/fruit-merge-game/ | https://github.com/DIDIDIP/fruit-merge-game |

---

## 🎯 학습 목표

1. HTML `<canvas>`와 JavaScript로 간단한 게임을 만들 수 있다.
2. **게임 루프**(입력 → 업데이트 → 그리기)의 구조를 이해한다.
3. 외부 라이브러리(물리 엔진 Matter.js)를 CDN으로 불러와 사용할 수 있다.
4. `git`과 `gh`(GitHub CLI)로 저장소를 만들고 코드를 올릴 수 있다.
5. GitHub Pages로 정적 웹사이트를 배포할 수 있다.

---

## 1교시 · 웹 게임의 기본 구조

### 1-1. 파일 하나로 끝나는 웹 게임

두 게임 모두 `index.html` **파일 하나**로 되어 있습니다. 한 파일 안에 세 가지가 함께 들어 있어요.

```
index.html
├── <style>   … 화면 꾸미기 (CSS)
├── <body>    … 캔버스, 점수판, 시작/게임오버 화면 (HTML)
└── <script>  … 게임 로직 (JavaScript)
```

> 💡 **왜 `index.html`일까?**
> GitHub Pages는 주소로 접속하면 `index.html`을 자동으로 보여 줍니다. 그래서 파일 이름을 꼭 `index.html`로 해야 주소 뒤에 파일 이름을 붙이지 않아도 됩니다.

### 1-2. 게임 루프 (Game Loop)

모든 실시간 게임의 심장입니다. 1초에 약 60번 아래 과정을 반복해요.

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ 입력 받기 │ →  │ 상태 갱신 │ →  │ 화면 그리기│ ─┐
└──────────┘    └──────────┘    └──────────┘  │
      ↑                                        │
      └──────── requestAnimationFrame ─────────┘
```

사과 받기 게임의 실제 코드:

```js
function loop(t) {
  const dt = Math.min((t - last) / 1000, 0.05); // 지난 프레임 이후 흐른 시간(초)
  last = t;
  if (running) update(dt);  // 위치, 충돌, 점수 계산
  draw();                   // 캔버스에 그리기
  requestAnimationFrame(loop); // 다음 프레임 예약
}
```

> 📝 **메모 · `dt`(델타 타임)를 쓰는 이유**
> 컴퓨터마다 1초에 그리는 횟수(프레임)가 다릅니다. 이동 거리를 `속도 × dt`로 계산하면 빠른 컴퓨터든 느린 컴퓨터든 같은 속도로 움직여요.
> `Math.min(..., 0.05)`는 탭을 잠깐 다른 곳에 두었다가 돌아올 때 `dt`가 너무 커져서 사과가 순간이동하는 것을 막아 줍니다.

### 1-3. 입력 처리 — 키보드, 마우스, 터치

```js
window.addEventListener('keydown', e => { if (e.key === 'ArrowLeft') keys.left = true; });
canvas.addEventListener('mousemove', e => setPointer(e.clientX));
canvas.addEventListener('touchmove', e => { setPointer(e.touches[0].clientX); e.preventDefault(); },
                        { passive: false });
```

> 📝 **메모** `touchmove`에서 `preventDefault()`를 호출해야 모바일에서 손가락으로 끌 때 화면이 같이 스크롤되지 않습니다. 이를 위해 `{ passive: false }` 옵션이 필요합니다.

### 1-4. 최고 점수 저장 — `localStorage`

```js
function getBest() {
  try { return +localStorage.getItem('appleBest') || 0; } catch { return 0; }
}
```

> 📝 **메모** 브라우저의 시크릿 모드 등에서는 `localStorage`가 막혀 에러가 날 수 있어서 `try/catch`로 감쌌습니다. 저장된 점수는 **그 브라우저에만** 남습니다.

---

## 2교시 · 🍎 사과 받기 게임

### 게임 규칙

- 떨어지는 사과를 바구니로 받습니다.
- 🍎 빨간 사과 **+1점**, 🍏 초록 사과 **+3점**
- 💣 폭탄을 받으면 **즉시 게임 오버**
- 사과를 **3번 놓치면** 게임 오버
- 조작: `←` `→` (또는 `A` `D`) / 마우스 / 터치, 시작·재시작은 `Space` 또는 `Enter`

### 핵심 개념 ① 난이도가 점점 올라가게 만들기

경과 시간(`elapsed`)을 이용해 시간이 지날수록 어려워지게 했습니다.

```js
const bombChance = Math.min(0.08 + elapsed * 0.002, 0.25); // 폭탄 확률: 8% → 최대 25%
vy: 140 + elapsed * 6 + Math.random() * 60,                // 낙하 속도 증가
spawnTimer = Math.max(0.35, 1.0 - elapsed * 0.012);        // 생성 간격: 1초 → 최소 0.35초
```

> 💡 `Math.min` / `Math.max`로 **상한·하한**을 두면 난이도가 끝없이 올라가 게임이 불가능해지는 것을 막을 수 있습니다.

### 핵심 개념 ② 충돌 판정 (받았는지 확인)

사과의 아래쪽이 바구니 윗면에 닿았고, 가로 위치가 바구니 폭 안에 있으면 "받았다"고 판단합니다.

```js
const caught = it.y + it.r * .6 >= top && it.y - it.r < top + 14 &&
               Math.abs(it.x - basket.x) < basket.w / 2 + it.r * .3;
```

### 핵심 개념 ③ 이미지 없이 그리기

사과, 폭탄, 바구니, 나무, 잔디를 **이미지 파일 없이** 캔버스 도형(`arc`, `ellipse`, `lineTo` …)만으로 그렸습니다. 덕분에 파일 하나만 올려도 게임이 동작합니다.

---

## 3교시 · 🍉 과일 합치기 게임 (수박게임 스타일)

### 게임 규칙

- 클릭·터치한 위치에 과일을 떨어뜨립니다. (키보드: `←` `→`로 위치 조정, `Space`로 떨어뜨리기)
- **같은 과일끼리 닿으면** 한 단계 큰 과일로 합쳐집니다.
- 진화 순서 (11단계):

  🍒 체리 → 🍓 딸기 → 🍇 포도 → 🍊 귤 → 🟠 감 → 🍎 사과 → 🍐 배 → 🍑 복숭아 → 🍍 파인애플 → 🍈 멜론 → 🍉 수박

- 떨어지는 과일은 **체리~감 5종류** 중에서 무작위로 나오고, 화면 위에서 **다음 과일**을 미리 볼 수 있습니다.
- 수박끼리 합치면 **보너스 100점**
- 과일이 **빨간 점선 위로 2초 넘게** 넘쳐 있으면 게임 오버

> 📝 **메모** 감은 딱 맞는 이모지가 없어서 🟠(주황 동그라미)로 대신 표시했습니다.

### 핵심 개념 ① 물리 엔진 — Matter.js

과일이 굴러가고, 쌓이고, 서로 밀어내는 움직임을 직접 계산하기는 매우 어렵습니다. 그래서 **물리 엔진**을 사용했어요.

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/matter-js/0.19.0/matter.min.js"></script>
```

```js
const { Engine, Bodies, Body, Composite, Events } = Matter;
engine = Engine.create();
engine.gravity.y = 1.2;                               // 중력
const b = Bodies.circle(x, y, f.r, { restitution: 0.15, friction: 0.3 }); // 원 모양 과일
Composite.add(engine.world, b);                       // 세계에 추가
```

| 용어 | 뜻 |
|---|---|
| `Engine` | 물리 계산을 담당하는 엔진 |
| `Bodies.circle` / `rectangle` | 원·사각형 물체 만들기 (과일·벽) |
| `isStatic: true` | 움직이지 않는 물체 (벽, 바닥) |
| `restitution` | 탄성 (얼마나 튕기는지) |
| `friction` | 마찰 (얼마나 미끄러지는지) |

> 📝 **메모 · CDN이란?** 라이브러리 파일을 내 저장소에 넣지 않고, 공용 서버(cdnjs)에서 불러오는 방식입니다. 간편하지만 **인터넷이 연결되어 있어야** 게임이 동작합니다.

### 핵심 개념 ② 같은 과일 합치기

물리 엔진이 알려 주는 **충돌 이벤트**를 듣고 있다가, 같은 단계의 과일이면 둘을 지우고 다음 단계 과일을 가운데에 만듭니다.

```js
Events.on(engine, 'collisionStart', handleCollisions);
Events.on(engine, 'collisionActive', handleCollisions);

function handleCollisions(e) {
  for (const pair of e.pairs) {
    const a = pair.bodyA, b = pair.bodyB;
    if (a.label !== 'fruit' || b.label !== 'fruit') continue;
    if (a.level !== b.level || a.merged || b.merged) continue; // 이미 합쳐진 건 건너뜀
    a.merged = b.merged = true;
    Composite.remove(engine.world, [a, b]);
    createFruit(a.level + 1, (a.position.x + b.position.x) / 2,
                             (a.position.y + b.position.y) / 2);
  }
}
```

> 📝 **메모 · `merged` 표시가 필요한 이유**
> 과일 하나가 같은 프레임에 두 개의 같은 과일과 동시에 닿을 수 있습니다. 표시 없이 처리하면 과일 하나가 **두 번 합쳐지는 버그**가 생겨요.
> `collisionActive`도 함께 듣는 이유는, 처음 닿을 때 놓친 쌍이 계속 붙어 있을 때도 합쳐지게 하기 위해서입니다.

### 핵심 개념 ③ 고정 시간 간격으로 물리 계산

```js
const STEP = 1000 / 60;
acc += dt;
while (acc >= STEP) { Engine.update(engine, STEP); acc -= STEP; }
```

> 💡 물리 엔진은 매번 **같은 시간 간격**으로 계산해야 안정적입니다. 화면 프레임이 들쭉날쭉해도 물리는 항상 1/60초 단위로 계산하도록 "시간 저금통(`acc`)" 방식을 썼습니다.

### 핵심 개념 ④ 게임 오버 판정

```js
if (now - b.bornAt < 1500) continue;  // 막 떨어뜨린 과일은 판정에서 제외
if (b.position.y - FRUITS[b.level].r < DANGER_Y) over = true;
dangerTimer = over ? dangerTimer + dt : 0;
if (dangerTimer > 2) gameOver();      // 2초 넘게 넘치면 게임 오버
```

> 📝 **메모** 과일은 위에서 떨어지기 때문에 처음엔 항상 선 위에 있습니다. 그래서 **생긴 지 1.5초가 안 된 과일은 제외**하고, 잠깐 튀어 오르는 경우도 봐주도록 **2초 유예**를 두었습니다.

### 핵심 개념 ⑤ 화면 크기에 맞추기

게임 안의 좌표는 항상 **400 × 600**으로 고정하고, 실제 화면 크기에 맞게 확대·축소(`scale`)해서 그립니다. 이렇게 하면 PC와 휴대폰 어디서든 같은 규칙으로 동작합니다.

```js
scale = Math.min(sw / W, sh / H, 1.4);
ctx.setTransform(scale * dpr, 0, 0, scale * dpr, 0, 0);
```

---

## 4교시 · GitHub에 올리고 Pages로 배포하기

### 4-1. 전체 흐름

```
[내 컴퓨터]                [GitHub]                    [인터넷]
index.html ──git push──▶  저장소(main 브랜치) ──Pages──▶ https://아이디.github.io/저장소이름/
```

### 4-2. 실제로 사용한 명령어

**① 커밋하기**

```bash
git checkout -b main          # (사과 게임) 기존 master 대신 main 브랜치 사용
# 또는 새 폴더라면
git init -b main              # (과일 게임) 처음부터 main 브랜치로 시작

git add index.html
git commit -m "Add apple catching web game"
```

**② GitHub 저장소 만들고 올리기** — `gh` (GitHub CLI) 사용

```bash
gh repo create apple-catch-game --public --source=. --remote=origin --push
```

| 옵션 | 뜻 |
|---|---|
| `--public` | 공개 저장소로 만들기 |
| `--source=.` | 현재 폴더를 저장소로 사용 |
| `--remote=origin` | 원격 이름을 `origin`으로 등록 |
| `--push` | 만들자마자 바로 올리기 |

**③ GitHub Pages 켜기**

```bash
gh api -X POST repos/DIDIDIP/apple-catch-game/pages \
  -f "source[branch]=main" -f "source[path]=/"
```

`main` 브랜치의 **루트 폴더(`/`)** 를 웹사이트로 배포하라는 뜻입니다.
(웹에서 하려면: 저장소 → **Settings** → **Pages** → Branch를 `main` / `/(root)`로 선택)

**④ 저장소 소개란에 게임 주소 등록** (선택)

```bash
gh repo edit DIDIDIP/apple-catch-game --homepage https://dididip.github.io/apple-catch-game/
```

**⑤ 배포 완료 확인**

```bash
gh api repos/DIDIDIP/apple-catch-game/pages/builds/latest --jq .status   # building → built
curl -s -o /dev/null -w "%{http_code}\n" https://dididip.github.io/apple-catch-game/  # 200이면 성공
```

### 4-3. 📝 작업 메모 (주의할 점)

- **브랜치 이름**: 사과 게임 폴더는 처음에 `master` 브랜치였는데, 기본 브랜치를 `main`으로 맞추기 위해 `main`으로 바꿔서 올렸습니다.
- **공개 저장소**: GitHub **무료 계정**에서 Pages를 쓰려면 저장소가 **public**이어야 합니다. 그래서 두 저장소 모두 공개로 만들었습니다.
- **폴더 분리**: 과일 합치기 게임은 사과 게임과 섞이지 않도록 별도 폴더(`fruit-merge-game`)와 별도 저장소로 만들었습니다.
- **반영 시간**: 이후 코드를 고치고 `git push` 하면 **1~2분 뒤** 사이트에 자동으로 반영됩니다.
- **줄바꿈 경고**: 커밋할 때 `LF will be replaced by CRLF` 경고가 나왔는데, Windows의 줄바꿈 방식 차이로 생기는 안내 메시지라 무시해도 괜찮습니다.
- **확인한 범위**: 두 사이트 모두 접속 시 정상 응답(200)이 오는 것까지 확인했습니다. 컴퓨터에 Node.js가 없어서 코드 문법 자동 검사는 하지 못했고, 실제 플레이 테스트는 브라우저에서 직접 해 봐야 합니다.

---

## ✅ 오늘의 정리

| 주제 | 핵심 한 줄 |
|---|---|
| 게임 루프 | `requestAnimationFrame`으로 **입력 → 갱신 → 그리기**를 반복 |
| 델타 타임 | `속도 × dt`로 계산해야 컴퓨터 성능과 상관없이 같은 속도 |
| 캔버스 그리기 | 이미지 없이 도형만으로도 캐릭터를 그릴 수 있음 |
| 물리 엔진 | 굴러가고 쌓이는 움직임은 Matter.js 같은 라이브러리에 맡기기 |
| 충돌 이벤트 | `merged` 표시로 중복 처리 버그 막기 |
| 배포 | `gh repo create` → Pages 켜기 → `아이디.github.io/저장소` 주소로 공개 |

---

## ✏️ 복습 문제

1. 게임 루프에서 `dt`를 쓰지 않고 매 프레임 `x += 5`처럼 움직이면 어떤 문제가 생길까요?
2. 과일 합치기에서 `merged` 표시를 지우면 어떤 버그가 생길 수 있을까요?
3. 막 떨어뜨린 과일을 게임 오버 판정에서 제외하지 않으면 어떻게 될까요?
4. GitHub Pages에서 파일 이름이 `game.html`이라면 접속 주소는 어떻게 달라질까요?

<details>
<summary>정답 보기</summary>

1. 컴퓨터 화면 주사율(60Hz, 144Hz 등)에 따라 움직이는 속도가 달라집니다.
2. 과일 하나가 두 과일과 동시에 닿으면 두 번 합쳐져서 과일이 복제될 수 있습니다.
3. 과일을 떨어뜨리는 위치가 위험선보다 위라서, 떨어뜨리자마자 게임 오버 판정이 쌓일 수 있습니다.
4. `https://아이디.github.io/저장소이름/game.html` 처럼 파일 이름까지 붙여야 합니다.

</details>

---

## 🚀 더 해 볼 만한 과제

- [ ] 사과 받기: 효과음 넣기 (`new Audio('catch.mp3').play()`)
- [ ] 사과 받기: 일정 시간 바구니가 커지는 🍯 아이템 추가
- [ ] 과일 합치기: 과일을 이모지 대신 직접 그린 그림으로 바꾸기
- [ ] 과일 합치기: Matter.js 파일을 저장소에 직접 넣어서 인터넷 없이도 동작하게 하기
- [ ] 두 게임을 한 페이지에서 고를 수 있는 "게임 모음" 메인 화면 만들기
