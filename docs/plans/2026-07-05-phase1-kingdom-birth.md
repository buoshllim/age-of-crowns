# Age of Crowns — 1단계 "왕국의 탄생" 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** "짓고 → 막고 → 쳐들어가 함락"이 한 판 안에서 완결되는 최소 플레이어블 게임 (설계 문서 10장 1단계).

**Architecture:** 순수 로직(`src/core/` — Phaser 없음, Vitest로 테스트)과 렌더링(`src/scenes/`, `src/game/` — Phaser 의존, 수동 검증)을 분리한다. 밸런스 숫자는 전부 `src/data/`에 두고 슈퍼천재가 코드 수정 없이 밸런싱한다. 맵은 시드 기반 랜덤 생성, 전투는 상성 2배 규칙, 적 AI는 순수 함수 `decide()`로 의사결정한다.

**Tech Stack:** Phaser 3 + Vite + Vitest (JavaScript, ESM). 배포는 GitHub → Vercel (에이전트는 push까지만).

**전제:**
- 모든 명령은 `~/Projects/tyground/age-of-crowns/` 에서 실행 (git repo 이미 init됨, `docs/design.md` 커밋 존재).
- 그래픽은 자동 생성 사각형 텍스처로 시작. `public/assets/{키}.png` 파일이 있으면 자동으로 그걸 사용 (무료 팩 Tiny Swords 파일은 슈퍼천재/아빠가 나중에 수동으로 넣음 — 이 계획에서는 다운로드하지 않는다).
- 설계 문서 대비 1단계 의도적 단순화 (2·3단계에서 해소):
  - 성문 없음 → 자기 성벽도 통과 불가. 플레이어는 성벽에 입구 구멍을 남겨야 함 (2단계 성문에서 해결).
  - 영토 확장 없음 → 적 1개 함락 = 승리 화면 (3단계에서 확장).
  - 창고·광물·무기 사슬 없음 (2단계).

**설계 문서:** `docs/design.md` (자원 사슬·유닛 능력치·상성은 그 문서가 진실).

---

## 파일 구조 (최종 모습)

```
age-of-crowns/
├── index.html
├── package.json
├── vite.config.js
├── public/assets/            # (선택) 픽셀 팩 PNG — 있으면 자동 사용
├── src/
│   ├── main.js               # Phaser 부트스트랩
│   ├── data/
│   │   ├── config.js         # 전역 설정 (맵 크기, 속도, 칭호 등)
│   │   ├── unitData.js       # 유닛 밸런스 ★슈퍼천재 수정 파일
│   │   └── buildingData.js   # 건물 밸런스 ★슈퍼천재 수정 파일
│   ├── core/                 # 순수 로직 (테스트 대상)
│   │   ├── ResourceStore.js
│   │   ├── combat.js         # 데미지·상성·레벨·칭호
│   │   ├── MapGenerator.js
│   │   ├── Pathfinder.js     # A* (적 건물은 높은 비용 = 성벽 뚫기)
│   │   └── AiBrain.js        # 적 왕국 의사결정
│   ├── game/
│   │   ├── Kingdom.js        # 왕국 상태 묶음
│   │   └── textures.js       # 텍스처 자동 생성 + 에셋 폴백
│   └── scenes/
│       ├── TitleScene.js
│       ├── LoadingScene.js   # 5~8초 랜덤 로딩 + 맵 생성
│       ├── GameScene.js      # 맵·건설·주민·유닛·전투·AI·승패
│       └── UiScene.js        # HUD (자원·인구·칭호·건설/훈련 버튼)
└── tests/
    ├── data.test.js
    ├── resourceStore.test.js
    ├── combat.test.js
    ├── mapGenerator.test.js
    ├── pathfinder.test.js
    └── aiBrain.test.js
```

---

### Task 1: 프로젝트 뼈대 (Vite + Phaser + Vitest)

**Files:**
- Create: `package.json`, `vite.config.js`, `index.html`, `src/main.js`

- [ ] **Step 1: package.json 작성**

```json
{
  "name": "age-of-crowns",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest run"
  },
  "dependencies": {
    "phaser": "^3.90.0"
  },
  "devDependencies": {
    "vite": "^6.0.0",
    "vitest": "^3.0.0"
  }
}
```

- [ ] **Step 2: vite.config.js 작성** (`base: './'` — itch.io ZIP 배포 대비, game-dev-knowledge 규칙)

```js
import { defineConfig } from 'vite';
export default defineConfig({ base: './' });
```

- [ ] **Step 3: index.html 작성**

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <title>Age of Crowns 👑</title>
  <style>
    html, body { margin: 0; padding: 0; background: #0a0a14; }
    canvas { display: block; margin: 0 auto; }
  </style>
</head>
<body>
  <script type="module" src="/src/main.js"></script>
</body>
</html>
```

- [ ] **Step 4: src/main.js — 임시 부트 (씬은 Task 7에서 교체)**

```js
import Phaser from 'phaser';

new Phaser.Game({
  type: Phaser.AUTO,
  width: 1280,
  height: 720,
  backgroundColor: '#1a1a2e',
  pixelArt: true,
  scale: { mode: Phaser.Scale.FIT, autoCenter: Phaser.Scale.CENTER_BOTH },
  scene: {
    create() {
      this.add.text(640, 360, 'AGE OF CROWNS — 부팅 OK', { fontSize: '32px' }).setOrigin(0.5);
    },
  },
});
```

- [ ] **Step 5: 설치 + 실행 확인**

Run: `npm install` → 성공. `npm run dev` → 브라우저에서 "AGE OF CROWNS — 부팅 OK" 텍스트 확인 후 종료.
Run: `npm test` → "no test files found"는 정상 (vitest 종료 코드 확인용으로 `--passWithNoTests`를 아직 안 씀 — Task 2에서 첫 테스트 생김).

- [ ] **Step 6: .gitignore + 커밋**

`.gitignore`:
```
node_modules/
dist/
```

```bash
git add -A
git commit -m "chore: Vite + Phaser + Vitest 프로젝트 뼈대"
```

---

### Task 2: 밸런스 데이터 파일 (★슈퍼천재 수정 파일)

**Files:**
- Create: `src/data/config.js`, `src/data/unitData.js`, `src/data/buildingData.js`
- Test: `tests/data.test.js`

- [ ] **Step 1: 실패하는 테스트 작성** — `tests/data.test.js`

```js
import { describe, it, expect } from 'vitest';
import { CONFIG } from '../src/data/config.js';
import { UNITS } from '../src/data/unitData.js';
import { BUILDINGS } from '../src/data/buildingData.js';

const RESOURCE_KEYS = ['wood', 'stone', 'food'];

describe('밸런스 데이터', () => {
  it('1단계 유닛 3종이 있다', () => {
    expect(Object.keys(UNITS)).toEqual(['warrior', 'archer', 'spearman']);
  });
  it('유닛 스탯은 전부 양수이고 비용은 아는 자원만 쓴다', () => {
    for (const u of Object.values(UNITS)) {
      expect(u.hp).toBeGreaterThan(0);
      expect(u.atk).toBeGreaterThan(0);
      expect(u.range).toBeGreaterThan(0);
      expect(u.speed).toBeGreaterThan(0);
      for (const k of Object.keys(u.cost)) expect(RESOURCE_KEYS).toContain(k);
    }
  });
  it('건물 비용도 아는 자원만 쓴다', () => {
    for (const b of Object.values(BUILDINGS)) {
      expect(b.hp).toBeGreaterThan(0);
      for (const k of Object.keys(b.cost)) expect(RESOURCE_KEYS).toContain(k);
    }
  });
  it('왕성은 건설 불가, 성벽은 길을 막는다', () => {
    expect(BUILDINGS.castle.buildable).toBe(false);
    expect(BUILDINGS.wall.blocksPath).toBe(true);
  });
  it('칭호는 함락 수가 늘수록 올라가는 순서다', () => {
    for (let i = 1; i < CONFIG.titles.length; i++) {
      expect(CONFIG.titles[i].need).toBeGreaterThan(CONFIG.titles[i - 1].need);
    }
  });
});
```

- [ ] **Step 2: 실패 확인** — Run: `npm test` / Expected: FAIL (모듈 없음)

- [ ] **Step 3: src/data/config.js 작성**

```js
// ★ 슈퍼천재 밸런싱 파일 — 숫자를 바꾸면 게임이 바뀐다!
export const CONFIG = {
  tileSize: 32,
  mapWidth: 100,
  mapHeight: 100,
  loadingMinMs: 5000,          // 로딩 최소 (설계: 5~8초 랜덤)
  loadingMaxMs: 8000,
  startResources: { wood: 100, stone: 60, food: 60 },
  territoryRadius: 18,         // 왕성 중심 건설 가능 반경 (타일, 체비쇼프 거리)
  productionMs: 3000,          // 생산 건물 1회 생산 시간
  productionAmount: 1,
  villagersPerHouse: 3,        // 인구 상한 = 주택 수 × 이 값
  villagerSpawnMs: 20000,      // 주민 생성 주기 (식량 있으면 절반 + 식량 1 소모)
  aggroRange: 6,               // 적 자동 감지 거리 (타일)
  attackCooldownMs: 1000,
  counterMultiplier: 2,        // 상성 데미지 배율
  wallChewCost: 20,            // 길찾기에서 적 건물 통과 비용 (성벽 뚫기)
  xpPerKill: 1,
  levelUps: { maxLevel: 5, hpBonus: 0.2, atkBonus: 0.1, xpToLevel: [0, 2, 5, 9, 14] },
  aiTickMs: 2000,              // 적 AI 생각 주기
  aiAttackArmySize: 6,         // 이 수 모이면 공격
  titles: [
    { need: 0, name: '남작' },
    { need: 1, name: '자작' },
    { need: 2, name: '백작' },
    { need: 3, name: '후작' },
    { need: 4, name: '공작' },
    { need: 6, name: '왕' },
    { need: 8, name: '황제' },
  ],
};
```

- [ ] **Step 4: src/data/unitData.js 작성** (설계 문서 6장 표 그대로. speed = 픽셀/초)

```js
// ★ 슈퍼천재 밸런싱 파일 — 유닛 능력치표 v1 (docs/design.md 6장)
// counters: 이 유닛이 2배 데미지를 주는 상대. 2단계 유닛(knight 등) id 미리 적어둠.
export const UNITS = {
  warrior:  { name: '전사', hp: 100, atk: 12, range: 1,   speed: 60, cost: { food: 30 },           counters: ['archer', 'crossbow'] },
  archer:   { name: '궁수', hp: 60,  atk: 10, range: 5,   speed: 60, cost: { food: 20, wood: 20 }, counters: ['spearman'] },
  spearman: { name: '창병', hp: 80,  atk: 10, range: 1.5, speed: 45, cost: { food: 25, wood: 10 }, counters: ['warrior', 'knight'] },
};
```

- [ ] **Step 5: src/data/buildingData.js 작성** (설계 문서 5장 표의 1단계 부분)

```js
// ★ 슈퍼천재 밸런싱 파일 — 건물표 v1 (docs/design.md 5장)
// size: 한 변 타일 수 / near: 3타일 내에 이 지형 필요 / produces: 주민 배치 시 생산 자원
export const BUILDINGS = {
  castle:   { name: '왕성',   hp: 1000, cost: {},                     size: 2, buildable: false },
  house:    { name: '주택',   hp: 300,  cost: { wood: 30 },           size: 1, buildable: true },
  lumber:   { name: '벌목장', hp: 250,  cost: { wood: 20 },           size: 1, buildable: true, produces: 'wood',  near: 'forest' },
  quarry:   { name: '채석장', hp: 250,  cost: { wood: 30 },           size: 1, buildable: true, produces: 'stone', near: 'rock' },
  farm:     { name: '농장',   hp: 200,  cost: { wood: 25 },           size: 1, buildable: true, produces: 'food' },
  barracks: { name: '훈련소', hp: 400,  cost: { wood: 50, stone: 30 }, size: 2, buildable: true },
  wall:     { name: '성벽',   hp: 500,  cost: { stone: 10 },          size: 1, buildable: true, blocksPath: true },
};
```

- [ ] **Step 6: 통과 확인 + 커밋**

Run: `npm test` / Expected: PASS (5 tests)

```bash
git add -A
git commit -m "feat: 밸런스 데이터 파일 (config/unit/building) + 검증 테스트"
```

---

### Task 3: ResourceStore (자원 지갑)

**Files:**
- Create: `src/core/ResourceStore.js`
- Test: `tests/resourceStore.test.js`

- [ ] **Step 1: 실패하는 테스트 작성** — `tests/resourceStore.test.js`

```js
import { describe, it, expect } from 'vitest';
import { ResourceStore } from '../src/core/ResourceStore.js';

describe('ResourceStore', () => {
  it('초기값을 갖고 시작한다', () => {
    const r = new ResourceStore({ wood: 100 });
    expect(r.get('wood')).toBe(100);
    expect(r.get('stone')).toBe(0);
  });
  it('add로 쌓인다', () => {
    const r = new ResourceStore();
    r.add('wood', 5);
    r.add('wood', 3);
    expect(r.get('wood')).toBe(8);
  });
  it('충분하면 spend 성공, 부족하면 실패하고 아무것도 안 깎는다', () => {
    const r = new ResourceStore({ wood: 50, stone: 10 });
    expect(r.spend({ wood: 30, stone: 10 })).toBe(true);
    expect(r.get('wood')).toBe(20);
    expect(r.spend({ wood: 20, stone: 5 })).toBe(false); // stone 부족
    expect(r.get('wood')).toBe(20); // 그대로
  });
});
```

- [ ] **Step 2: 실패 확인** — Run: `npm test` / Expected: FAIL

- [ ] **Step 3: src/core/ResourceStore.js 구현**

```js
export class ResourceStore {
  constructor(initial = {}) {
    this.amounts = { wood: 0, stone: 0, food: 0, ...initial };
  }
  get(type) { return this.amounts[type] ?? 0; }
  add(type, n) { this.amounts[type] = this.get(type) + n; }
  canAfford(cost) {
    return Object.entries(cost).every(([t, n]) => this.get(t) >= n);
  }
  spend(cost) {
    if (!this.canAfford(cost)) return false;
    for (const [t, n] of Object.entries(cost)) this.amounts[t] -= n;
    return true;
  }
}
```

- [ ] **Step 4: 통과 확인 + 커밋**

Run: `npm test` / Expected: PASS

```bash
git add -A
git commit -m "feat: ResourceStore (자원 지갑)"
```

---

### Task 4: combat.js (데미지·상성·레벨·칭호)

**Files:**
- Create: `src/core/combat.js`
- Test: `tests/combat.test.js`

- [ ] **Step 1: 실패하는 테스트 작성** — `tests/combat.test.js`

```js
import { describe, it, expect } from 'vitest';
import { damage, levelStats, levelForXp, titleFor } from '../src/core/combat.js';
import { UNITS } from '../src/data/unitData.js';
import { CONFIG } from '../src/data/config.js';

describe('전투 규칙', () => {
  it('상성이면 2배 데미지 (창병 → 전사)', () => {
    expect(damage(UNITS.spearman, 'warrior')).toBe(UNITS.spearman.atk * 2);
  });
  it('상성 아니면 기본 데미지 (전사 → 창병)', () => {
    expect(damage(UNITS.warrior, 'spearman')).toBe(UNITS.warrior.atk);
  });
  it('건물 공격은 기본 데미지 (상성 없음)', () => {
    expect(damage(UNITS.warrior, 'building')).toBe(UNITS.warrior.atk);
  });
});

describe('레벨업 규칙', () => {
  it('레벨 1은 기본 스탯 그대로', () => {
    expect(levelStats(UNITS.warrior, 1)).toEqual({ hp: 100, atk: 12 });
  });
  it('레벨 5 전사는 체력 180, 공격 17 (설계: 레벨당 hp+20% atk+10%)', () => {
    expect(levelStats(UNITS.warrior, 5)).toEqual({ hp: 180, atk: 17 });
  });
  it('경험치 → 레벨 (xpToLevel 경계값)', () => {
    expect(levelForXp(0)).toBe(1);
    expect(levelForXp(2)).toBe(2);
    expect(levelForXp(14)).toBe(5);
    expect(levelForXp(999)).toBe(5); // 최대 레벨 캡
  });
});

describe('칭호 규칙', () => {
  it('함락 0회 = 남작, 1회 = 자작, 8회 = 황제', () => {
    expect(titleFor(0, CONFIG.titles)).toBe('남작');
    expect(titleFor(1, CONFIG.titles)).toBe('자작');
    expect(titleFor(5, CONFIG.titles)).toBe('공작'); // 5회는 아직 공작 (왕은 6회)
    expect(titleFor(8, CONFIG.titles)).toBe('황제');
  });
});
```

- [ ] **Step 2: 실패 확인** — Run: `npm test` / Expected: FAIL

- [ ] **Step 3: src/core/combat.js 구현**

```js
import { CONFIG } from '../data/config.js';

// attackerDef: UNITS의 항목. defenderTypeId: 상대 유닛 id 또는 'building'
export function damage(attackerDef, defenderTypeId) {
  const bonus = attackerDef.counters?.includes(defenderTypeId) ? CONFIG.counterMultiplier : 1;
  return Math.round(attackerDef.atk * bonus);
}

export function levelStats(baseDef, level) {
  const { hpBonus, atkBonus } = CONFIG.levelUps;
  return {
    hp: Math.round(baseDef.hp * (1 + (level - 1) * hpBonus)),
    atk: Math.round(baseDef.atk * (1 + (level - 1) * atkBonus)),
  };
}

export function levelForXp(xp) {
  const t = CONFIG.levelUps.xpToLevel;
  let lv = 1;
  for (let i = 0; i < t.length; i++) if (xp >= t[i]) lv = i + 1;
  return Math.min(lv, CONFIG.levelUps.maxLevel);
}

export function titleFor(conquests, titles) {
  let cur = titles[0].name;
  for (const t of titles) if (conquests >= t.need) cur = t.name;
  return cur;
}
```

- [ ] **Step 4: 통과 확인 + 커밋**

Run: `npm test` / Expected: PASS

```bash
git add -A
git commit -m "feat: 전투 규칙 (상성 2배, 레벨업, 칭호)"
```

---

### Task 5: MapGenerator (시드 랜덤 맵)

**Files:**
- Create: `src/core/MapGenerator.js`
- Test: `tests/mapGenerator.test.js`

- [ ] **Step 1: 실패하는 테스트 작성** — `tests/mapGenerator.test.js`

```js
import { describe, it, expect } from 'vitest';
import { generateMap, TILE } from '../src/core/MapGenerator.js';
import { CONFIG } from '../src/data/config.js';

describe('랜덤 맵 생성', () => {
  const map = generateMap(42);

  it('설정 크기의 2차원 배열을 만든다', () => {
    expect(map.terrain.length).toBe(CONFIG.mapHeight);
    expect(map.terrain[0].length).toBe(CONFIG.mapWidth);
  });
  it('같은 시드는 같은 맵 (재현 가능)', () => {
    expect(generateMap(42)).toEqual(generateMap(42));
  });
  it('다른 시드는 다른 맵', () => {
    expect(generateMap(1).terrain).not.toEqual(generateMap(2).terrain);
  });
  it('숲과 돌산이 존재한다', () => {
    const flat = map.terrain.flat();
    expect(flat).toContain(TILE.FOREST);
    expect(flat).toContain(TILE.ROCK);
  });
  it('플레이어와 적 시작점은 멀리 떨어져 있다 (50타일 이상)', () => {
    const p = map.playerStart, e = map.enemyStarts[0];
    expect(Math.hypot(p.x - e.x, p.y - e.y)).toBeGreaterThan(50);
  });
  it('시작점 주변 7×7은 잔디 (왕성 지을 자리)', () => {
    for (const s of [map.playerStart, ...map.enemyStarts]) {
      for (let dy = -3; dy <= 3; dy++)
        for (let dx = -3; dx <= 3; dx++)
          expect(map.terrain[s.y + dy][s.x + dx]).toBe(TILE.GRASS);
    }
  });
  it('각 시작점의 영토 반경 안에 숲과 돌산이 보장된다 (벌목장·채석장용)', () => {
    for (const s of [map.playerStart, ...map.enemyStarts]) {
      let forest = 0, rock = 0;
      const r = CONFIG.territoryRadius;
      for (let dy = -r; dy <= r; dy++)
        for (let dx = -r; dx <= r; dx++) {
          const y = s.y + dy, x = s.x + dx;
          if (y < 0 || y >= CONFIG.mapHeight || x < 0 || x >= CONFIG.mapWidth) continue;
          if (map.terrain[y][x] === TILE.FOREST) forest++;
          if (map.terrain[y][x] === TILE.ROCK) rock++;
        }
      expect(forest).toBeGreaterThan(0);
      expect(rock).toBeGreaterThan(0);
    }
  });
});
```

- [ ] **Step 2: 실패 확인** — Run: `npm test` / Expected: FAIL

- [ ] **Step 3: src/core/MapGenerator.js 구현**

```js
import { CONFIG } from '../data/config.js';

export const TILE = { GRASS: 0, FOREST: 1, ROCK: 2 };

// 시드 기반 난수 (같은 시드 = 같은 맵)
export function mulberry32(seed) {
  return function () {
    seed |= 0; seed = (seed + 0x6D2B79F5) | 0;
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}

export function generateMap(seed, cfg = CONFIG) {
  const rand = mulberry32(seed);
  const w = cfg.mapWidth, h = cfg.mapHeight;
  const terrain = Array.from({ length: h }, () => Array(w).fill(TILE.GRASS));

  // 랜덤 워크로 덩어리(blob) 배치
  const blob = (type, startX, startY, size) => {
    let x = startX, y = startY;
    for (let j = 0; j < size; j++) {
      if (x >= 0 && x < w && y >= 0 && y < h) terrain[y][x] = type;
      x += Math.floor(rand() * 3) - 1;
      y += Math.floor(rand() * 3) - 1;
    }
  };
  for (let i = 0; i < 40; i++) blob(TILE.FOREST, Math.floor(rand() * w), Math.floor(rand() * h), 30);
  for (let i = 0; i < 25; i++) blob(TILE.ROCK, Math.floor(rand() * w), Math.floor(rand() * h), 20);

  // 시작점: 플레이어 좌상단 근처, 적은 우하단 근처 (멀리 떨어지게)
  const playerStart = { x: 15 + Math.floor(rand() * 8), y: 15 + Math.floor(rand() * 8) };
  const enemyStarts = [{ x: w - 23 + Math.floor(rand() * 8), y: h - 23 + Math.floor(rand() * 8) }];

  for (const s of [playerStart, ...enemyStarts]) {
    // 영토 안에 숲·돌산 보장 (왕성에서 6~8타일 거리에 심음)
    blob(TILE.FOREST, s.x + 7, s.y - 6, 25);
    blob(TILE.ROCK, s.x - 6, s.y + 7, 18);
    // 왕성 주변 7×7 잔디 정리 (blob이 침범했을 수 있으니 마지막에)
    for (let dy = -3; dy <= 3; dy++)
      for (let dx = -3; dx <= 3; dx++) {
        const y = s.y + dy, x = s.x + dx;
        if (y >= 0 && y < h && x >= 0 && x < w) terrain[y][x] = TILE.GRASS;
      }
  }
  return { terrain, playerStart, enemyStarts };
}
```

- [ ] **Step 4: 통과 확인 + 커밋**

Run: `npm test` / Expected: PASS. (숲·돌산 보장 테스트가 불안정하면 blob 크기를 늘리지 말고 심는 위치를 왕성에 더 가깝게 조정할 것 — 반경 18 안이 조건.)

```bash
git add -A
git commit -m "feat: 시드 기반 랜덤 맵 생성 (숲·돌산 영토 보장)"
```

---

### Task 6: Pathfinder (A* — 적 성벽은 비용 높게 = 뚫고 간다)

**Files:**
- Create: `src/core/Pathfinder.js`
- Test: `tests/pathfinder.test.js`

- [ ] **Step 1: 실패하는 테스트 작성** — `tests/pathfinder.test.js`

```js
import { describe, it, expect } from 'vitest';
import { findPath } from '../src/core/Pathfinder.js';

// 5×5 테스트 그리드 도우미: 'X'=통과불가, 'W'=적 성벽(비용 20), '.'=평지(1)
function costFn(rows) {
  return (x, y) => {
    const c = rows[y][x];
    if (c === 'X') return Infinity;
    if (c === 'W') return 20;
    return 1;
  };
}

describe('A* 길찾기', () => {
  it('장애물 없으면 직선 경로 (맨해튼 거리 + 1 지점)', () => {
    const path = findPath(costFn(['.....', '.....', '.....', '.....', '.....']), { x: 0, y: 0 }, { x: 4, y: 0 }, 5, 5);
    expect(path.length).toBe(5);
    expect(path[0]).toEqual({ x: 0, y: 0 });
    expect(path[4]).toEqual({ x: 4, y: 0 });
  });
  it('통과불가(X)는 돌아간다', () => {
    const path = findPath(costFn(['..X..', '..X..', '..X..', '..X..', '.....']), { x: 0, y: 0 }, { x: 4, y: 0 }, 5, 5);
    expect(path).not.toBeNull();
    expect(path.some(p => p.x === 2 && p.y === 4)).toBe(true); // 아래로 우회
  });
  it('적 성벽(W)은 우회로가 없으면 뚫고 간다', () => {
    const path = findPath(costFn(['..W..', '..W..', '..W..', '..W..', '..W..']), { x: 0, y: 2 }, { x: 4, y: 2 }, 5, 5);
    expect(path).not.toBeNull();
    expect(path.some(p => p.x === 2)).toBe(true); // 성벽 칸을 지남
  });
  it('완전히 막히면 null', () => {
    const path = findPath(costFn(['..X..', '..X..', '..X..', '..X..', '..X..']), { x: 0, y: 2 }, { x: 4, y: 2 }, 5, 5);
    expect(path).toBeNull();
  });
});
```

- [ ] **Step 2: 실패 확인** — Run: `npm test` / Expected: FAIL

- [ ] **Step 3: src/core/Pathfinder.js 구현**

```js
// A* (4방향). costAt(x, y) → 1=평지, 큰 수=뚫어야 하는 적 건물, Infinity=통과 불가.
// 반환: [{x,y}, ...] (시작점 포함) 또는 null.
export function findPath(costAt, from, to, w, h) {
  const key = (x, y) => y * w + x;
  const open = new Map([[key(from.x, from.y), { x: from.x, y: from.y, g: 0, f: 0, parent: null }]]);
  const closed = new Set();

  while (open.size) {
    let cur = null;
    for (const n of open.values()) if (!cur || n.f < cur.f) cur = n;
    if (cur.x === to.x && cur.y === to.y) {
      const path = [];
      for (let n = cur; n; n = n.parent) path.unshift({ x: n.x, y: n.y });
      return path;
    }
    open.delete(key(cur.x, cur.y));
    closed.add(key(cur.x, cur.y));

    for (const [dx, dy] of [[1, 0], [-1, 0], [0, 1], [0, -1]]) {
      const x = cur.x + dx, y = cur.y + dy;
      if (x < 0 || x >= w || y < 0 || y >= h || closed.has(key(x, y))) continue;
      const c = costAt(x, y);
      if (c === Infinity) continue;
      const g = cur.g + c;
      const existing = open.get(key(x, y));
      if (existing && existing.g <= g) continue;
      const heuristic = Math.abs(to.x - x) + Math.abs(to.y - y);
      open.set(key(x, y), { x, y, g, f: g + heuristic, parent: cur });
    }
  }
  return null;
}
```

**성능 메모:** 유닛이 명령을 받을 때만 경로를 계산하고 매 프레임 재계산하지 않는다 (Task 11). 100×100에서 유닛 수십 기면 충분.

- [ ] **Step 4: 통과 확인 + 커밋**

Run: `npm test` / Expected: PASS

```bash
git add -A
git commit -m "feat: A* 길찾기 (적 성벽은 높은 비용으로 뚫기)"
```

---

### Task 7: 씬 뼈대 — 타이틀 → 로딩(5~8초) → 게임

**Files:**
- Create: `src/game/textures.js`, `src/scenes/TitleScene.js`, `src/scenes/LoadingScene.js`, `src/scenes/GameScene.js`(뼈대), `src/scenes/UiScene.js`(뼈대)
- Modify: `src/main.js` (전체 교체)

- [ ] **Step 1: src/game/textures.js — 에셋 있으면 사용, 없으면 사각형 자동 생성**

```js
// public/assets/{key}.png 이 로드돼 있으면 그대로 쓰고, 없으면 색 사각형을 만든다.
// → 4단계에서 슈퍼천재 에셋으로 교체할 때 코드 수정 없이 파일만 넣으면 됨.
export const TEXTURE_COLORS = {
  castle: 0xffd700, house: 0xc17a4a, lumber: 0x8b5a2b, quarry: 0x9a9a9a,
  farm: 0xd9c65b, barracks: 0xb04a4a, wall: 0x7d7d8e,
  warrior: 0xe63946, archer: 0x2a9d8f, spearman: 0x457b9d,
  villager: 0xf4a261, flag: 0xff2222,
};

export function ensureTextures(scene) {
  for (const [k, color] of Object.entries(TEXTURE_COLORS)) {
    if (scene.textures.exists(k)) continue;
    const g = scene.add.graphics();
    g.fillStyle(color, 1).fillRect(0, 0, 32, 32);
    g.lineStyle(2, 0x000000, 0.35).strokeRect(1, 1, 30, 30);
    g.generateTexture(k, 32, 32);
    g.destroy();
  }
  if (!scene.textures.exists('terrain')) {
    // 타일셋: [잔디, 숲, 돌산] 가로로 배치 → TILE 인덱스와 일치
    const g = scene.add.graphics();
    [[0, 0x4a7c3a], [32, 0x2d5a27], [64, 0x6e6e6e]].forEach(([x, c]) => {
      g.fillStyle(c, 1).fillRect(x, 0, 32, 32);
      g.fillStyle(0x000000, 0.08).fillRect(x, 0, 32, 2); // 픽셀 느낌 살짝
    });
    g.generateTexture('terrain', 96, 32);
    g.destroy();
  }
}
```

- [ ] **Step 2: src/scenes/TitleScene.js**

```js
import Phaser from 'phaser';

export class TitleScene extends Phaser.Scene {
  constructor() { super('TitleScene'); }

  create() {
    const cx = this.scale.width / 2;
    this.add.text(cx, 200, '👑 AGE OF CROWNS', { fontSize: '64px', fontStyle: 'bold', color: '#ffd700' }).setOrigin(0.5);
    this.add.text(cx, 280, '나만의 왕국을 짓고, 상대의 왕국을 함락시켜라', { fontSize: '20px', color: '#cccccc' }).setOrigin(0.5);

    const btn = this.add.rectangle(cx, 430, 320, 72, 0x2a9d8f).setInteractive({ useHandCursor: true });
    this.add.text(cx, 430, '게임 플레이 하기', { fontSize: '28px', fontStyle: 'bold' }).setOrigin(0.5);
    btn.on('pointerover', () => btn.setFillStyle(0x33b3a6));
    btn.on('pointerout', () => btn.setFillStyle(0x2a9d8f));
    btn.on('pointerdown', () => this.scene.start('LoadingScene'));
  }
}
```

- [ ] **Step 3: src/scenes/LoadingScene.js — 5~8초 랜덤 + 에셋 시도 로드 + 맵 생성**

```js
import Phaser from 'phaser';
import { CONFIG } from '../data/config.js';
import { generateMap } from '../core/MapGenerator.js';
import { TEXTURE_COLORS } from '../game/textures.js';

export class LoadingScene extends Phaser.Scene {
  constructor() { super('LoadingScene'); }

  preload() {
    // 에셋이 있으면 로드, 없으면 조용히 무시 (textures.js가 사각형으로 대체)
    this.load.on('loaderror', (file) => console.log(`에셋 없음(사각형으로 대체): ${file.key}`));
    for (const key of Object.keys(TEXTURE_COLORS)) this.load.image(key, `assets/${key}.png`);
    this.load.image('terrain', 'assets/terrain.png');
  }

  create() {
    const cx = this.scale.width / 2;
    this.add.text(cx, 300, '로딩 중...', { fontSize: '36px' }).setOrigin(0.5);
    this.add.rectangle(cx, 380, 500, 24).setStrokeStyle(2, 0xffffff);
    const bar = this.add.rectangle(cx - 248, 380, 1, 20, 0xffd700).setOrigin(0, 0.5);

    // 설계 규칙: 5~8초 사이 무작위
    const duration = Phaser.Math.Between(CONFIG.loadingMinMs, CONFIG.loadingMaxMs);
    const map = generateMap(Date.now() % 2147483647);

    this.tweens.add({
      targets: bar, width: 496, duration, ease: 'Sine.easeInOut',
      onComplete: () => {
        this.scene.start('GameScene', { map });
        this.scene.launch('UiScene');
      },
    });
  }
}
```

- [ ] **Step 4: src/scenes/GameScene.js — 뼈대 (Task 8부터 채움)**

```js
import Phaser from 'phaser';
import { ensureTextures } from '../game/textures.js';

export class GameScene extends Phaser.Scene {
  constructor() { super('GameScene'); }

  init(data) { this.map = data.map; }

  create() {
    ensureTextures(this);
    this.add.text(20, 20, 'GameScene OK — 맵 렌더는 Task 8', { fontSize: '20px' });
  }

  update(time, delta) {}
}
```

- [ ] **Step 5: src/scenes/UiScene.js — 뼈대 (Task 9부터 채움)**

```js
import Phaser from 'phaser';

export class UiScene extends Phaser.Scene {
  constructor() { super('UiScene'); }
  create() {}
}
```

- [ ] **Step 6: src/main.js 전체 교체**

```js
import Phaser from 'phaser';
import { TitleScene } from './scenes/TitleScene.js';
import { LoadingScene } from './scenes/LoadingScene.js';
import { GameScene } from './scenes/GameScene.js';
import { UiScene } from './scenes/UiScene.js';

new Phaser.Game({
  type: Phaser.AUTO,
  width: 1280,
  height: 720,
  backgroundColor: '#1a1a2e',
  pixelArt: true,
  scale: { mode: Phaser.Scale.FIT, autoCenter: Phaser.Scale.CENTER_BOTH },
  scene: [TitleScene, LoadingScene, GameScene, UiScene],
});
```

- [ ] **Step 7: 수동 확인 + 커밋**

Run: `npm run dev` → 확인 목록:
1. 타이틀 화면에 로고와 "게임 플레이 하기" 버튼
2. 버튼 클릭 → "로딩 중..." + 진행바 (5~8초 — 두 번 실행해서 시간이 달라지는지)
3. 로딩 끝 → "GameScene OK" 텍스트
4. 콘솔에 "에셋 없음" 로그 (정상 — 사각형 대체 동작 확인)

```bash
git add -A
git commit -m "feat: 타이틀 → 랜덤 로딩(5~8초) → 게임 씬 흐름"
```

---

### Task 8: 맵 렌더링 + 카메라

**Files:**
- Modify: `src/scenes/GameScene.js` (전체 교체)

- [ ] **Step 1: GameScene 교체 — 타일맵 렌더 + 카메라 조작**

```js
import Phaser from 'phaser';
import { CONFIG } from '../data/config.js';
import { ensureTextures } from '../game/textures.js';

export class GameScene extends Phaser.Scene {
  constructor() { super('GameScene'); }

  init(data) { this.map = data.map; }

  create() {
    ensureTextures(this);
    const ts = CONFIG.tileSize;

    // 지형 타일맵 (terrain 텍스처의 0=잔디,1=숲,2=돌산 인덱스와 TILE이 일치)
    const tilemap = this.make.tilemap({ data: this.map.terrain, tileWidth: ts, tileHeight: ts });
    const tileset = tilemap.addTilesetImage('terrain');
    tilemap.createLayer(0, tileset, 0, 0);

    // 카메라: 경계 + WASD/화살표 + 휠 줌
    const cam = this.cameras.main;
    cam.setBounds(0, 0, CONFIG.mapWidth * ts, CONFIG.mapHeight * ts);
    cam.centerOn(this.map.playerStart.x * ts, this.map.playerStart.y * ts);
    this.keys = this.input.keyboard.addKeys('W,A,S,D,UP,LEFT,DOWN,RIGHT');
    this.input.on('wheel', (p, o, dx, dy) => {
      cam.setZoom(Phaser.Math.Clamp(cam.zoom - dy * 0.001, 0.5, 2));
    });
  }

  // 좌표 변환 도우미 (이후 태스크 공용)
  worldToTile(wx, wy) { return { x: Math.floor(wx / CONFIG.tileSize), y: Math.floor(wy / CONFIG.tileSize) }; }
  tileToWorld(t) { return t * CONFIG.tileSize + CONFIG.tileSize / 2; }

  update(time, delta) {
    const cam = this.cameras.main;
    const speed = (600 / cam.zoom) * (delta / 1000);
    if (this.keys.A.isDown || this.keys.LEFT.isDown) cam.scrollX -= speed;
    if (this.keys.D.isDown || this.keys.RIGHT.isDown) cam.scrollX += speed;
    if (this.keys.W.isDown || this.keys.UP.isDown) cam.scrollY -= speed;
    if (this.keys.S.isDown || this.keys.DOWN.isDown) cam.scrollY += speed;
  }
}
```

- [ ] **Step 2: 수동 확인 + 커밋**

Run: `npm run dev` → 확인 목록:
1. 초록 잔디 위에 짙은 숲 덩어리와 회색 돌산 덩어리가 랜덤하게 보임
2. WASD/화살표로 카메라 이동, 맵 끝에서 멈춤
3. 마우스 휠로 줌 인/아웃 (0.5~2배)
4. 게임 재시작마다 다른 맵

```bash
git add -A
git commit -m "feat: 지형 타일맵 렌더 + 카메라 (이동·줌)"
```

---

### Task 9: Kingdom + 건설 시스템 (툴바·고스트·비용·영토)

**Files:**
- Create: `src/game/Kingdom.js`
- Modify: `src/scenes/GameScene.js`, `src/scenes/UiScene.js`

- [ ] **Step 1: src/game/Kingdom.js**

```js
import { CONFIG } from '../data/config.js';
import { ResourceStore } from '../core/ResourceStore.js';

export class Kingdom {
  constructor(id, name, color, isPlayer) {
    this.id = id;
    this.name = name;
    this.color = color;           // 팀 색 (스프라이트 틴트)
    this.isPlayer = isPlayer;
    this.resources = new ResourceStore({ ...CONFIG.startResources });
    this.buildings = [];          // {kind, tx, ty, size, hp, maxHp, kingdom, sprite, ring, hpBar, hasWorker, produceAcc}
    this.units = [];              // Task 11
    this.villagers = [];          // Task 10
    this.castle = null;
    this.conquests = 0;
    this.defeated = false;
    this.spawnAcc = 0;            // 주민 생성 타이머
  }
  popCap() { return this.buildings.filter(b => b.kind === 'house').length * CONFIG.villagersPerHouse; }
}
```

- [ ] **Step 2: GameScene에 왕국·점유 그리드·건설 로직 추가**

`create()` 끝에 추가:

```js
    // 점유 그리드 (건물 참조 or null)
    this.occ = Array.from({ length: CONFIG.mapHeight }, () => Array(CONFIG.mapWidth).fill(null));

    // 왕국 2개: 플레이어(파랑), 적(빨강)
    this.kingdoms = [
      new Kingdom(0, '나의 왕국', 0x4da6ff, true),
      new Kingdom(1, '붉은 왕국', 0xff4d4d, false),
    ];
    this.placeBuilding('castle', this.map.playerStart.x, this.map.playerStart.y, this.kingdoms[0], { free: true });
    this.placeBuilding('castle', this.map.enemyStarts[0].x, this.map.enemyStarts[0].y, this.kingdoms[1], { free: true });

    // 건설 모드
    this.buildMode = null; // 건물 kind 또는 null
    this.ghost = this.add.rectangle(0, 0, CONFIG.tileSize, CONFIG.tileSize, 0x00ff00, 0.35).setVisible(false);
    this.events.on('build-select', (kind) => { this.buildMode = kind; });
    this.input.keyboard.on('keydown-ESC', () => { this.buildMode = null; this.ghost.setVisible(false); });

    this.input.on('pointermove', (p) => {
      if (!this.buildMode) return;
      const w = p.positionToCamera(this.cameras.main);
      const t = this.worldToTile(w.x, w.y);
      const size = BUILDINGS[this.buildMode].size;
      const ok = this.canPlace(this.buildMode, t.x, t.y, this.kingdoms[0]);
      this.ghost.setVisible(true)
        .setPosition(t.x * CONFIG.tileSize + size * 16, t.y * CONFIG.tileSize + size * 16)
        .setSize(size * CONFIG.tileSize, size * CONFIG.tileSize)
        .setFillStyle(ok ? 0x00ff00 : 0xff0000, 0.35);
    });
    this.input.on('pointerdown', (p) => {
      if (!this.buildMode || p.rightButtonDown()) return;
      const w = p.positionToCamera(this.cameras.main);
      const t = this.worldToTile(w.x, w.y);
      if (this.canPlace(this.buildMode, t.x, t.y, this.kingdoms[0])) {
        this.placeBuilding(this.buildMode, t.x, t.y, this.kingdoms[0]);
        // 성벽은 연속 건설 편의를 위해 모드 유지, 나머지는 해제
        if (this.buildMode !== 'wall') { this.buildMode = null; this.ghost.setVisible(false); }
      }
    });
```

파일 상단 import에 추가: `import { BUILDINGS } from '../data/buildingData.js';`, `import { Kingdom } from '../game/Kingdom.js';`, `import { TILE } from '../core/MapGenerator.js';`

클래스에 메서드 추가:

```js
  canPlace(kind, tx, ty, kingdom) {
    const def = BUILDINGS[kind];
    if (!def?.buildable) return false;
    if (!kingdom.resources.canAfford(def.cost)) return false;
    // 영토: 왕성 중심 체비쇼프 거리
    const c = kingdom.castle;
    if (Math.max(Math.abs(tx - c.tx), Math.abs(ty - c.ty)) > CONFIG.territoryRadius) return false;
    // 칸 검사: 경계 + 잔디 + 비어있음
    for (let dy = 0; dy < def.size; dy++)
      for (let dx = 0; dx < def.size; dx++) {
        const x = tx + dx, y = ty + dy;
        if (x < 0 || x >= CONFIG.mapWidth || y < 0 || y >= CONFIG.mapHeight) return false;
        if (this.map.terrain[y][x] !== TILE.GRASS) return false;
        if (this.occ[y][x]) return false;
      }
    // near 조건: 3타일 내 해당 지형 (벌목장=숲, 채석장=돌산)
    if (def.near) {
      const want = def.near === 'forest' ? TILE.FOREST : TILE.ROCK;
      let found = false;
      for (let dy = -3; dy <= 3 && !found; dy++)
        for (let dx = -3; dx <= 3 && !found; dx++) {
          const x = tx + dx, y = ty + dy;
          if (x >= 0 && x < CONFIG.mapWidth && y >= 0 && y < CONFIG.mapHeight && this.map.terrain[y][x] === want) found = true;
        }
      if (!found) return false;
    }
    return true;
  }

  placeBuilding(kind, tx, ty, kingdom, { free = false } = {}) {
    const def = BUILDINGS[kind];
    if (!free && !kingdom.resources.spend(def.cost)) return null;
    const ts = CONFIG.tileSize;
    const px = tx * ts + (def.size * ts) / 2, py = ty * ts + (def.size * ts) / 2;
    const sprite = this.add.image(px, py, kind).setDisplaySize(def.size * ts, def.size * ts);
    const ring = this.add.rectangle(px, py, def.size * ts, def.size * ts).setStrokeStyle(2, kingdom.color);
    const b = { kind, tx, ty, size: def.size, hp: def.hp, maxHp: def.hp, kingdom, sprite, ring, hpBar: null, hasWorker: false, assigned: false, produceAcc: 0 };
    for (let dy = 0; dy < def.size; dy++)
      for (let dx = 0; dx < def.size; dx++) this.occ[ty + dy][tx + dx] = b;
    kingdom.buildings.push(b);
    if (kind === 'castle') kingdom.castle = b;
    return b;
  }
```

- [ ] **Step 3: UiScene — 자원 HUD + 건설 툴바**

`src/scenes/UiScene.js` 전체 교체:

```js
import Phaser from 'phaser';
import { BUILDINGS } from '../data/buildingData.js';

export class UiScene extends Phaser.Scene {
  constructor() { super('UiScene'); }

  create() {
    this.game_ = this.scene.get('GameScene');

    // 상단 자원 바
    this.hud = this.add.text(16, 12, '', { fontSize: '20px', backgroundColor: '#00000088', padding: { x: 10, y: 6 } });

    // 하단 건설 툴바
    const buildables = Object.entries(BUILDINGS).filter(([, d]) => d.buildable);
    buildables.forEach(([kind, def], i) => {
      const x = 90 + i * 170, y = this.scale.height - 44;
      const btn = this.add.rectangle(x, y, 160, 56, 0x333355).setInteractive({ useHandCursor: true }).setStrokeStyle(1, 0x8888aa);
      const costTxt = Object.entries(def.cost).map(([k, v]) => `${{ wood: '🪵', stone: '🪨', food: '🌾' }[k]}${v}`).join(' ');
      this.add.text(x, y, `${def.name}\n${costTxt}`, { fontSize: '15px', align: 'center' }).setOrigin(0.5);
      btn.on('pointerdown', () => this.game_.events.emit('build-select', kind));
    });
  }

  update() {
    const k = this.game_.kingdoms?.[0];
    if (!k) return;
    const r = k.resources;
    this.hud.setText(`🪵 ${r.get('wood')}   🪨 ${r.get('stone')}   🌾 ${r.get('food')}   👥 ${k.villagers.length}/${k.popCap()}`);
  }
}
```

- [ ] **Step 4: 수동 확인 + 커밋**

Run: `npm run dev` → 확인 목록:
1. 플레이어 왕성(파란 테두리 금색)과 멀리 적 왕성(빨간 테두리) 존재
2. 상단에 자원 🪵100 🪨60 🌾60, 하단에 건물 버튼 6개
3. 주택 버튼 → 초록/빨강 고스트, 잔디+영토 안에서만 초록
4. 배치하면 자원 차감. 자원 부족하면 고스트 빨강
5. 벌목장은 숲 3타일 이내에서만 초록, 채석장은 돌산 근처만
6. 성벽은 연속 클릭으로 줄줄이 건설, ESC로 해제

```bash
git add -A
git commit -m "feat: 왕국·건설 시스템 (툴바, 고스트, 영토, 지형 조건)"
```

---

### Task 10: 주민 + 생산 시스템

**Files:**
- Modify: `src/scenes/GameScene.js`

- [ ] **Step 1: 주민 스폰 + 배치 + 생산 로직 추가**

`create()` 끝에 추가:

```js
    // 시작 주민 3명
    for (let i = 0; i < 3; i++) this.spawnVillager(this.kingdoms[0]);
    for (let i = 0; i < 3; i++) this.spawnVillager(this.kingdoms[1]);
```

클래스에 메서드 추가:

```js
  spawnVillager(kingdom) {
    const c = kingdom.castle;
    const px = this.tileToWorld(c.tx) + Phaser.Math.Between(-20, 20);
    const py = this.tileToWorld(c.ty + c.size) + Phaser.Math.Between(0, 16);
    const sprite = this.add.image(px, py, 'villager').setDisplaySize(18, 18).setTint(kingdom.color);
    kingdom.villagers.push({ kingdom, sprite, job: null, tx: c.tx, ty: c.ty });
  }

  villagerSystem(delta) {
    for (const k of this.kingdoms) {
      if (k.defeated) continue;
      // 1) 주민 생성 (주택 있어야 함. 식량 있으면 2배 속도 + 식량 1 소모 — 설계 4장)
      const houses = k.buildings.filter(b => b.kind === 'house').length;
      if (houses > 0 && k.villagers.length < k.popCap()) {
        k.spawnAcc += delta;
        const need = k.resources.get('food') >= 1 ? CONFIG.villagerSpawnMs / 2 : CONFIG.villagerSpawnMs;
        if (k.spawnAcc >= need) {
          k.spawnAcc = 0;
          if (k.resources.get('food') >= 1 && need < CONFIG.villagerSpawnMs) k.resources.spend({ food: 1 });
          this.spawnVillager(k);
        }
      }
      // 2) 일 없는 주민 → 일꾼 없는 생산 건물에 배치
      for (const v of k.villagers) {
        if (v.job) continue;
        const target = k.buildings.find(b => BUILDINGS[b.kind].produces && !b.hasWorker && !b.assigned);
        if (target) {
          target.assigned = true;
          v.job = target;
          this.tweens.add({
            targets: v.sprite,
            x: this.tileToWorld(target.tx), y: this.tileToWorld(target.ty) + 20,
            duration: 1500,
            onComplete: () => { if (v.job === target) target.hasWorker = true; },
          });
        }
      }
    }
  }

  productionSystem(delta) {
    for (const k of this.kingdoms) {
      if (k.defeated) continue;
      for (const b of k.buildings) {
        const def = BUILDINGS[b.kind];
        if (!def.produces || !b.hasWorker) continue;
        b.produceAcc += delta;
        if (b.produceAcc >= CONFIG.productionMs) {
          b.produceAcc = 0;
          k.resources.add(def.produces, CONFIG.productionAmount);
        }
      }
    }
  }
```

`update()` 끝에 추가:

```js
    this.villagerSystem(delta);
    this.productionSystem(delta);
```

(주민이 죽거나 건물이 파괴될 때의 정리는 Task 12의 `destroyBuilding()`에서 처리.)

- [ ] **Step 2: 수동 확인 + 커밋**

Run: `npm run dev` → 확인 목록:
1. 왕성 옆에 작은 주민 3명
2. 벌목장(숲 옆) 건설 → 주민 1명이 걸어가 붙음 → 3초마다 🪵 +1 (HUD 숫자 증가)
3. 농장 건설 → 🌾 증가 시작
4. 주택 짓기 전엔 주민 안 늘어남 / 주택 + 식량 있으면 10초마다 주민 +1(식량 -1), 👥 상한 = 주택×3
5. 생산 건물이 주민 수보다 많으면 일부는 일꾼 없음 (생산 안 됨)

```bash
git add -A
git commit -m "feat: 주민 생성·배치 + 자원 생산 시스템"
```

---

### Task 11: 유닛 훈련 + 이동 (A*)

**Files:**
- Modify: `src/scenes/GameScene.js`, `src/scenes/UiScene.js`

- [ ] **Step 1: GameScene에 유닛 훈련·이동 추가**

상단 import 추가: `import { UNITS } from '../data/unitData.js';`, `import { findPath } from '../core/Pathfinder.js';`, `import { levelStats, levelForXp, damage } from '../core/combat.js';`

`create()` 끝에 추가:

```js
    this.events.on('train-unit', (type) => this.trainUnit(type, this.kingdoms[0]));
```

클래스에 메서드 추가:

```js
  trainUnit(type, kingdom) {
    const def = UNITS[type];
    const barracks = kingdom.buildings.find(b => b.kind === 'barracks');
    if (!barracks || !kingdom.resources.spend(def.cost)) return null;
    // 훈련소 주변 빈 칸에 스폰
    const spot = this.freeTileAround(barracks) ?? { x: barracks.tx, y: barracks.ty + barracks.size };
    const sprite = this.add.image(this.tileToWorld(spot.x), this.tileToWorld(spot.y), type)
      .setDisplaySize(24, 24).setTint(kingdom.color);
    const u = {
      type, def, kingdom, sprite,
      hp: def.hp, maxHp: def.hp, atk: def.atk, level: 1, xp: 0,
      tx: spot.x, ty: spot.y, path: null, pathIndex: 0,
      order: null,               // {tx, ty} 공격 깃발 목적지 (Task 13)
      target: null, attackAcc: 0, scanAcc: Math.random() * 300,
      hpBar: null,
    };
    kingdom.units.push(u);
    return u;
  }

  freeTileAround(b) {
    for (let dy = -1; dy <= b.size; dy++)
      for (let dx = -1; dx <= b.size; dx++) {
        const x = b.tx + dx, y = b.ty + dy;
        if (x < 0 || x >= CONFIG.mapWidth || y < 0 || y >= CONFIG.mapHeight) continue;
        if (this.map.terrain[y][x] === TILE.GRASS && !this.occ[y][x]) return { x, y };
      }
    return null;
  }

  // 길찾기 비용: 잔디 1 / 숲·돌산 불가 / 아군 건물 불가 / 적 건물 = wallChewCost(뚫는다)
  moveCostFn(kingdom) {
    return (x, y) => {
      if (this.map.terrain[y][x] !== TILE.GRASS) return Infinity;
      const b = this.occ[y][x];
      if (!b) return 1;
      return b.kingdom === kingdom ? Infinity : CONFIG.wallChewCost;
    };
  }

  orderMove(u, tx, ty) {
    const path = findPath(this.moveCostFn(u.kingdom), { x: u.tx, y: u.ty }, { x: tx, y: ty }, CONFIG.mapWidth, CONFIG.mapHeight);
    if (path) { u.path = path; u.pathIndex = 1; }
  }

  unitMoveSystem(delta) {
    for (const k of this.kingdoms) {
      for (const u of k.units) {
        if (!u.path || u.pathIndex >= u.path.length) continue;
        const next = u.path[u.pathIndex];
        const blocking = this.occ[next.y][next.x];
        if (blocking && blocking.kingdom !== u.kingdom) {
          // 다음 칸이 적 건물 → 이동 대신 그 건물을 공격 대상으로 (Task 12에서 처리)
          u.target = blocking;
          continue;
        }
        const wx = this.tileToWorld(next.x), wy = this.tileToWorld(next.y);
        const dist = Phaser.Math.Distance.Between(u.sprite.x, u.sprite.y, wx, wy);
        const step = u.def.speed * (delta / 1000);
        if (dist <= step) {
          u.sprite.setPosition(wx, wy);
          u.tx = next.x; u.ty = next.y;
          u.pathIndex++;
          if (u.pathIndex >= u.path.length) { u.path = null; u.order = null; }
        } else {
          const ang = Math.atan2(wy - u.sprite.y, wx - u.sprite.x);
          u.sprite.x += Math.cos(ang) * step;
          u.sprite.y += Math.sin(ang) * step;
        }
      }
    }
  }
```

`update()` 끝에 추가: `this.unitMoveSystem(delta);`

- [ ] **Step 2: UiScene에 훈련 버튼 추가** — `create()` 끝에 추가:

```js
    // 우측 하단 유닛 훈련 버튼
    import('../data/unitData.js').then(({ UNITS }) => {
      Object.entries(UNITS).forEach(([type, def], i) => {
        const x = this.scale.width - 90, y = this.scale.height - 260 + i * 70;
        const btn = this.add.rectangle(x, y, 150, 60, 0x553333).setInteractive({ useHandCursor: true }).setStrokeStyle(1, 0xaa8888);
        const costTxt = Object.entries(def.cost).map(([k, v]) => `${{ wood: '🪵', stone: '🪨', food: '🌾' }[k]}${v}`).join(' ');
        this.add.text(x, y, `${def.name}\n${costTxt}`, { fontSize: '15px', align: 'center' }).setOrigin(0.5);
        btn.on('pointerdown', () => this.game_.events.emit('train-unit', type));
      });
    });
```

(참고: UiScene 상단에서 정적 import를 써도 된다 — `import { UNITS } from '../data/unitData.js';` 후 바로 사용. 동적 import는 필수가 아님. 정적 import를 권장.)

- [ ] **Step 3: 수동 확인 + 커밋**

Run: `npm run dev` → 확인 목록:
1. 훈련소 없이 유닛 버튼 클릭 → 아무 일 없음
2. 훈련소 건설 후 전사 훈련 → 훈련소 옆에 파란 틴트 유닛 등장, 🌾 30 차감
3. (이동 확인은 Task 13 깃발에서 — 아직 명령 수단 없음, 정상)

```bash
git add -A
git commit -m "feat: 유닛 훈련 + A* 이동 시스템"
```

---

### Task 12: 전투 시스템 (타겟팅·공격·죽음·레벨업)

**Files:**
- Modify: `src/scenes/GameScene.js`

- [ ] **Step 1: 전투 로직 추가** — 클래스에 메서드 추가:

```js
  // 300ms마다 어그로 범위 내 가장 가까운 적 탐색
  combatSystem(delta) {
    for (const k of this.kingdoms) {
      for (const u of [...k.units]) {
        u.scanAcc += delta;
        if (u.scanAcc >= 300) {
          u.scanAcc = 0;
          if (!u.target || u.target.hp <= 0) u.target = this.findTarget(u);
        }
        if (!u.target || u.target.hp <= 0) continue;

        const t = u.target;
        const ttx = t.def ? t.tx : t.tx + (t.size - 1) / 2;   // 건물은 중심
        const tty = t.def ? t.ty : t.ty + (t.size - 1) / 2;
        const dist = Math.hypot(u.tx - ttx, u.ty - tty);
        const range = Math.max(u.def.range, 1.2);

        if (dist <= range + (t.size ? t.size / 2 : 0)) {
          u.path = null;                       // 사거리 안 → 멈추고 공격
          u.attackAcc += delta;
          if (u.attackAcc >= CONFIG.attackCooldownMs) {
            u.attackAcc = 0;
            this.dealDamage(u, t);
          }
        } else if (!u.path && !u.order) {
          this.orderMove(u, Math.round(ttx), Math.round(tty)); // 접근
        }
      }
    }
  }

  findTarget(u) {
    let best = null, bestD = CONFIG.aggroRange;
    for (const k of this.kingdoms) {
      if (k === u.kingdom || k.defeated) continue;
      for (const e of k.units) {
        const d = Math.hypot(u.tx - e.tx, u.ty - e.ty);
        if (d < bestD) { best = e; bestD = d; }
      }
      for (const b of k.buildings) {
        const d = Math.hypot(u.tx - b.tx, u.ty - b.ty);
        if (d < bestD) { best = b; bestD = d; }
      }
    }
    return best;
  }

  dealDamage(attacker, target) {
    const targetTypeId = target.def ? target.type : 'building';
    const dmg = Math.round(damage({ ...attacker.def, atk: attacker.atk }, targetTypeId));
    target.hp -= dmg;
    this.updateHpBar(target);
    if (target.hp <= 0) {
      if (target.def) this.killUnit(target, attacker);
      else this.destroyBuilding(target);
      attacker.target = null;
    }
  }

  updateHpBar(e) {
    const w = e.def ? 24 : e.size * CONFIG.tileSize;
    if (!e.hpBar) e.hpBar = this.add.rectangle(e.sprite.x, e.sprite.y - (e.def ? 18 : w / 2 + 6), w, 4, 0x00ff00);
    e.hpBar.setPosition(e.sprite.x, e.sprite.y - (e.def ? 18 : w / 2 + 6));
    const ratio = Math.max(e.hp / e.maxHp, 0);
    e.hpBar.width = w * ratio;
    e.hpBar.setFillStyle(ratio > 0.5 ? 0x00ff00 : ratio > 0.25 ? 0xffaa00 : 0xff0000);
  }

  killUnit(u, killer) {
    u.sprite.destroy(); u.hpBar?.destroy();
    u.kingdom.units = u.kingdom.units.filter(x => x !== u);
    if (killer?.def) {
      killer.xp += CONFIG.xpPerKill;
      const newLevel = levelForXp(killer.xp);
      if (newLevel > killer.level) {
        killer.level = newLevel;
        const s = levelStats(killer.def, newLevel);
        killer.hp += s.hp - killer.maxHp;   // 늘어난 만큼 회복
        killer.maxHp = s.hp;
        killer.atk = s.atk;
        killer.sprite.setDisplaySize(24 + newLevel * 2, 24 + newLevel * 2); // 레벨 표시(살짝 커짐)
      }
    }
  }

  destroyBuilding(b) {
    b.sprite.destroy(); b.ring.destroy(); b.hpBar?.destroy();
    for (let dy = 0; dy < b.size; dy++)
      for (let dx = 0; dx < b.size; dx++) this.occ[b.ty + dy][b.tx + dx] = null;
    b.kingdom.buildings = b.kingdom.buildings.filter(x => x !== b);
    // 일하던 주민 해고 → 다시 배치 대기
    for (const v of b.kingdom.villagers) if (v.job === b) { v.job = null; }
    if (b.kind === 'castle') this.onCastleDestroyed(b.kingdom); // Task 15
  }

  onCastleDestroyed(kingdom) { /* Task 15에서 구현 */ }
```

`update()` 끝에 추가: `this.combatSystem(delta);` 그리고 `unitMoveSystem` 안에서 유닛이 움직일 때 hp바가 따라오도록, `unitMoveSystem`의 위치 갱신 두 곳(순간이동/보간) 직후에 `if (u.hpBar) this.updateHpBar(u);` 추가.

- [ ] **Step 2: 수동 확인 + 커밋**

Run: `npm run dev` → 확인 (임시로 콘솔에서 적 유닛 생성해 테스트):
브라우저 콘솔에서 게임 씬 접근이 번거로우므로, **임시 테스트 코드**를 `create()` 끝에 넣고 확인 후 제거:

```js
    // TEST-ONLY: 적 전사 1기를 내 왕성 근처에 소환 (확인 후 삭제)
    const enemyBarracks = this.placeBuilding('barracks', this.map.playerStart.x + 6, this.map.playerStart.y + 6, this.kingdoms[1], { free: true });
    this.trainUnit('warrior', this.kingdoms[1]);
```

확인 목록:
1. 적 전사(빨간 틴트)가 내 주민/건물 쪽으로 접근해 공격, HP바 감소
2. 내 전사 훈련 → 서로 싸움 → 진 쪽 사라짐
3. 창병으로 전사 공격 시 더 빨리 죽임 (상성 2배 체감)
4. 여러 마리 잡은 유닛이 살짝 커짐 (레벨업)

**확인 후 TEST-ONLY 블록 삭제** 후 커밋:

```bash
git add -A
git commit -m "feat: 전투 시스템 (타겟팅, 상성 데미지, HP바, 레벨업)"
```

---

### Task 13: 공격 깃발

**Files:**
- Modify: `src/scenes/GameScene.js`, `src/scenes/UiScene.js`

- [ ] **Step 1: GameScene — 깃발 모드 + 전군 이동**

`create()` 끝에 추가:

```js
    this.flagSprite = this.add.image(0, 0, 'flag').setDisplaySize(24, 24).setVisible(false);
    this.flagMode = false;
    this.events.on('flag-mode', () => { this.flagMode = true; this.buildMode = null; this.ghost.setVisible(false); });
    this.input.keyboard.on('keydown-F', () => { this.flagMode = true; });
```

`pointerdown` 핸들러를 아래로 교체 (Task 9의 것을 대체 — 깃발 분기 추가):

```js
    this.input.on('pointerdown', (p) => {
      if (p.rightButtonDown()) { this.buildMode = null; this.flagMode = false; this.ghost.setVisible(false); return; }
      const w = p.positionToCamera(this.cameras.main);
      const t = this.worldToTile(w.x, w.y);
      if (this.flagMode) {
        this.flagMode = false;
        this.flagSprite.setVisible(true).setPosition(this.tileToWorld(t.x), this.tileToWorld(t.y));
        for (const u of this.kingdoms[0].units) {
          u.target = null;
          u.order = { tx: t.x, ty: t.y };
          this.orderMove(u, t.x, t.y);
        }
        return;
      }
      if (this.buildMode && this.canPlace(this.buildMode, t.x, t.y, this.kingdoms[0])) {
        this.placeBuilding(this.buildMode, t.x, t.y, this.kingdoms[0]);
        if (this.buildMode !== 'wall') { this.buildMode = null; this.ghost.setVisible(false); }
      }
    });
```

- [ ] **Step 2: UiScene — 깃발 버튼** — `create()` 끝에 추가:

```js
    const fx = this.scale.width - 90, fy = this.scale.height - 44;
    const flagBtn = this.add.rectangle(fx, fy, 150, 60, 0x882222).setInteractive({ useHandCursor: true }).setStrokeStyle(2, 0xff6666);
    this.add.text(fx, fy, '🚩 공격 깃발 (F)', { fontSize: '15px' }).setOrigin(0.5);
    flagBtn.on('pointerdown', () => this.game_.events.emit('flag-mode'));
```

- [ ] **Step 3: 수동 확인 + 커밋**

Run: `npm run dev` → 확인 목록:
1. 전사 3기 훈련 → 깃발 버튼(또는 F) → 맵 클릭 → 전원 그 지점으로 진군
2. 깃발을 적 왕성에 꽂으면 → 가는 길에 만나는 적과 교전(어그로), 도착하면 적 건물 공격
3. 적 성벽으로 둘러싸인 곳에 깃발 → 성벽을 부수고 진입 (A* 뚫기 비용 동작)
4. 우클릭으로 깃발 모드 취소

```bash
git add -A
git commit -m "feat: 공격 깃발 (전군 이동 명령)"
```

---

### Task 14: 적 AI 왕국 (AiBrain + 통합)

**Files:**
- Create: `src/core/AiBrain.js`
- Test: `tests/aiBrain.test.js`
- Modify: `src/scenes/GameScene.js`

- [ ] **Step 1: 실패하는 테스트 작성** — `tests/aiBrain.test.js`

```js
import { describe, it, expect } from 'vitest';
import { decide } from '../src/core/AiBrain.js';

// state: { counts: {house,lumber,quarry,farm,barracks,wall}, armySize, threatened, canAffordAll }
const base = { counts: { house: 0, lumber: 0, quarry: 0, farm: 0, barracks: 0, wall: 0 }, armySize: 0, threatened: false };

describe('적 AI 의사결정', () => {
  it('맨 처음엔 벌목장부터 (나무가 모든 건설의 시작)', () => {
    expect(decide(base)).toEqual({ type: 'build', building: 'lumber' });
  });
  it('건설 순서: 벌목장 → 농장 → 주택 → 채석장 → 훈련소', () => {
    let s = { ...base, counts: { ...base.counts, lumber: 1 } };
    expect(decide(s)).toEqual({ type: 'build', building: 'farm' });
    s = { ...s, counts: { ...s.counts, farm: 1 } };
    expect(decide(s)).toEqual({ type: 'build', building: 'house' });
    s = { ...s, counts: { ...s.counts, house: 1 } };
    expect(decide(s)).toEqual({ type: 'build', building: 'quarry' });
    s = { ...s, counts: { ...s.counts, quarry: 1 } };
    expect(decide(s)).toEqual({ type: 'build', building: 'barracks' });
  });
  it('기지 완성 후엔 군대 훈련 (전사→궁수→창병 순환)', () => {
    const s = { ...base, counts: { house: 1, lumber: 1, quarry: 1, farm: 1, barracks: 1, wall: 0 }, armySize: 0 };
    expect(decide(s).type).toBe('train');
    expect(decide({ ...s, armySize: 0 }).unit).toBe('warrior');
    expect(decide({ ...s, armySize: 1 }).unit).toBe('archer');
    expect(decide({ ...s, armySize: 2 }).unit).toBe('spearman');
  });
  it('군대가 6 이상 모이면 공격', () => {
    const s = { ...base, counts: { house: 1, lumber: 1, quarry: 1, farm: 1, barracks: 1, wall: 0 }, armySize: 6 };
    expect(decide(s)).toEqual({ type: 'attack' });
  });
  it('공격받는 중이면 무조건 방어', () => {
    const s = { ...base, threatened: true };
    expect(decide(s)).toEqual({ type: 'defend' });
  });
});
```

- [ ] **Step 2: 실패 확인** — Run: `npm test` / Expected: FAIL

- [ ] **Step 3: src/core/AiBrain.js 구현**

```js
import { CONFIG } from '../data/config.js';

const BUILD_ORDER = ['lumber', 'farm', 'house', 'quarry', 'barracks'];
const TRAIN_CYCLE = ['warrior', 'archer', 'spearman'];

// 순수 함수: 왕국 상태 스냅샷 → 다음 행동 하나
export function decide(state) {
  if (state.threatened) return { type: 'defend' };
  for (const b of BUILD_ORDER) {
    if ((state.counts[b] ?? 0) < 1) return { type: 'build', building: b };
  }
  if (state.armySize >= CONFIG.aiAttackArmySize) return { type: 'attack' };
  return { type: 'train', unit: TRAIN_CYCLE[state.armySize % TRAIN_CYCLE.length] };
}
```

- [ ] **Step 4: 통과 확인** — Run: `npm test` / Expected: PASS

- [ ] **Step 5: GameScene에 AI 실행기 통합**

상단 import 추가: `import { decide } from '../core/AiBrain.js';`

`create()` 끝에 추가: `this.aiAcc = 0;`

클래스에 메서드 추가:

```js
  aiSystem(delta) {
    const ai = this.kingdoms[1];
    if (ai.defeated) return;
    this.aiAcc += delta;
    if (this.aiAcc < CONFIG.aiTickMs) return;
    this.aiAcc = 0;

    const counts = {};
    for (const b of ai.buildings) counts[b.kind] = (counts[b.kind] ?? 0) + 1;
    const threatened = this.kingdoms[0].units.some(
      u => Math.max(Math.abs(u.tx - ai.castle.tx), Math.abs(u.ty - ai.castle.ty)) <= CONFIG.territoryRadius
    );
    const action = decide({ counts, armySize: ai.units.length, threatened });

    if (action.type === 'build') {
      const spot = this.findAiBuildSpot(action.building, ai);
      if (spot) this.placeBuilding(action.building, spot.x, spot.y, ai);
    } else if (action.type === 'train') {
      this.trainUnit(action.unit, ai);
    } else if (action.type === 'attack') {
      const target = this.kingdoms[0].castle;
      for (const u of ai.units) { u.order = { tx: target.tx, ty: target.ty }; this.orderMove(u, target.tx, target.ty); }
    } else if (action.type === 'defend') {
      for (const u of ai.units) {
        if (!u.target) { u.order = { tx: ai.castle.tx, ty: ai.castle.ty }; this.orderMove(u, ai.castle.tx, ai.castle.ty); }
      }
    }
  }

  // 왕성 주변 나선형 탐색으로 canPlace 되는 첫 칸
  findAiBuildSpot(kind, kingdom) {
    const c = kingdom.castle;
    for (let r = 2; r <= CONFIG.territoryRadius; r++) {
      for (let dy = -r; dy <= r; dy++)
        for (let dx = -r; dx <= r; dx++) {
          if (Math.max(Math.abs(dx), Math.abs(dy)) !== r) continue; // 링 테두리만
          const x = c.tx + dx, y = c.ty + dy;
          if (x < 0 || x >= CONFIG.mapWidth || y < 0 || y >= CONFIG.mapHeight) continue;
          if (this.canPlace(kind, x, y, kingdom)) return { x, y };
        }
    }
    return null;
  }
```

`update()` 끝에 추가: `this.aiSystem(delta);`

- [ ] **Step 6: 수동 확인 + 커밋**

Run: `npm run dev` → 카메라를 적 왕성으로 이동(WASD) 후 확인:
1. 적이 2초 간격으로 벌목장→농장→주택→채석장→훈련소를 스스로 지음
2. 이후 유닛을 순환 훈련, 6기 모이면 내 왕성으로 진군
3. 내가 먼저 쳐들어가면(깃발) 적 유닛들이 왕성으로 돌아와 방어
4. `npm test` 전체 PASS 유지

```bash
git add -A
git commit -m "feat: 적 AI 왕국 (건설·훈련·공격·방어 의사결정)"
```

---

### Task 15: 함락·승패·칭호 + 마무리

**Files:**
- Modify: `src/scenes/GameScene.js`, `src/scenes/UiScene.js`
- Create: `README.md`

- [ ] **Step 1: onCastleDestroyed 구현** — Task 12의 빈 메서드 교체:

```js
  onCastleDestroyed(kingdom) {
    kingdom.defeated = true;
    // 함락된 왕국의 모든 것 제거 (설계 3장: 왕성 파괴 = 함락)
    for (const u of [...kingdom.units]) this.killUnit(u, null);
    for (const v of kingdom.villagers) v.sprite.destroy();
    kingdom.villagers = [];
    for (const b of [...kingdom.buildings]) this.destroyBuilding(b);

    if (kingdom.isPlayer) {
      this.showEndScreen('💀 패배...', '왕성이 무너졌다. 다시 도전하자!', 0xff4444);
    } else {
      this.kingdoms[0].conquests += 1;
      this.showEndScreen('👑 함락!', `${kingdom.name}을(를) 정복했다!\n칭호 상승: ${titleFor(this.kingdoms[0].conquests, CONFIG.titles)}`, 0xffd700);
    }
  }

  showEndScreen(title, sub, color) {
    this.scene.pause();
    const ui = this.scene.get('UiScene');
    const cx = ui.scale.width / 2, cy = ui.scale.height / 2;
    ui.add.rectangle(cx, cy, ui.scale.width, ui.scale.height, 0x000000, 0.7);
    ui.add.text(cx, cy - 60, title, { fontSize: '72px', fontStyle: 'bold', color: `#${color.toString(16)}` }).setOrigin(0.5);
    ui.add.text(cx, cy + 20, sub, { fontSize: '24px', align: 'center' }).setOrigin(0.5);
    const btn = ui.add.rectangle(cx, cy + 120, 260, 60, 0x2a9d8f).setInteractive({ useHandCursor: true });
    ui.add.text(cx, cy + 120, '타이틀로', { fontSize: '24px' }).setOrigin(0.5);
    btn.on('pointerdown', () => { ui.scene.stop('GameScene'); ui.scene.stop('UiScene'); ui.scene.start('TitleScene'); });
  }
```

상단 import 추가: `import { titleFor } from '../core/combat.js';` (이미 combat import가 있으면 항목만 추가)

**주의:** `destroyBuilding()`이 왕성 파괴 시 `onCastleDestroyed`를 부르고, 그 안에서 다시 `destroyBuilding`을 호출한다. 무한 루프 방지를 위해 `destroyBuilding` 첫 줄에 가드 추가:

```js
    if (b.destroyed) return;
    b.destroyed = true;
```

- [ ] **Step 2: UiScene HUD에 칭호 표시** — `update()`의 setText를 교체:

```js
    const title = titleFor(k.conquests, CONFIG.titles);
    this.hud.setText(`🎖 ${title}   🪵 ${r.get('wood')}   🪨 ${r.get('stone')}   🌾 ${r.get('food')}   👥 ${k.villagers.length}/${k.popCap()}`);
```

상단 import 추가: `import { titleFor } from '../core/combat.js';`, `import { CONFIG } from '../data/config.js';`

- [ ] **Step 3: README.md 작성**

```markdown
# Age of Crowns 👑

나만의 왕국을 짓고, 지키고, 상대 왕국을 함락시키는 탑다운 2D 픽셀 왕국 샌드박스.

- 설계: `docs/design.md` / 1단계 구현 계획: `docs/plans/2026-07-05-phase1-kingdom-birth.md`
- 실행: `npm install && npm run dev`
- 테스트: `npm test`
- 밸런싱: `src/data/` 의 숫자를 고치면 게임이 바뀐다 (★슈퍼천재 파일)
- 에셋 교체: `public/assets/{키}.png` 를 넣으면 자동 적용 (없으면 색 사각형)

만든 사람: 슈퍼천재 (기획·밸런스) × 소니 (설계) × Sonnet (구현)
```

- [ ] **Step 4: 최종 통합 플레이테스트**

Run: `npm test` / Expected: 전체 PASS
Run: `npm run dev` → **엔드-투-엔드 시나리오**:
1. 타이틀 → 로딩(5~8초) → 게임 시작, 칭호 "남작" 표시
2. 벌목장·농장·주택·채석장 → 자원 성장 → 훈련소 → 성벽으로 입구만 남기고 두르기
3. 적 공격 웨이브 방어 (창병이 전사 상대로 유리한지 체감)
4. 병력 모아 깃발로 적 왕성 공격 → 성벽 뚫고 → 왕성 파괴
5. "👑 함락!" + "칭호 상승: 자작" → 타이틀로 → 재시작 시 새로운 랜덤 맵
6. (패배도 확인: 방어 안 하고 방치 → 왕성 파괴 → "💀 패배..." 화면)

- [ ] **Step 5: 커밋**

```bash
git add -A
git commit -m "feat: 함락·승패·칭호 시스템 + README — 1단계 완성"
```

---

## 완료 기준 (설계 문서 10장 1단계 대조)

- [x] 타이틀 + 로딩(5~8초 랜덤) — Task 7
- [x] 랜덤 맵 + 카메라 스크롤 — Task 5, 8
- [x] 자원 3종 + 건물 7종(왕성 포함) — Task 2, 9
- [x] 주민 자동 생성·자동 일하기 — Task 10
- [x] 유닛 3종 + 상성 삼각형 + 레벨업 — Task 2, 4, 11, 12
- [x] 공격 깃발 — Task 13
- [x] 적 왕국 1개 (건설·생산·공격 AI) — Task 14
- [x] 왕성 파괴 = 함락 + 칭호 상승 — Task 15

## 다음 단계 예고 (이 계획 범위 밖)

- GitHub repo 생성 + push (아빠 확인 후), Vercel 연동은 아빠가 직접
- 슈퍼천재 플레이테스트 → `src/data/` 밸런싱
- 2단계 계획: 광산·대장간·창고·성문·탑 + 유닛 6종
