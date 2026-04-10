# Architecture — コードの構造と設計

---

## 現在のファイル構成

```
battle-sim/
├── CLAUDE.md                    ← Claudeへの指示書（AIアシスタント向け）
├── src/
│   └── index.html               ← プロトタイプ本体（全機能がここに）
└── docs/
    ├── discussions/             ← 設計議論の記録
    ├── issues/                  ← 課題・バックログ
    └── planning/                ← 開発手順（このディレクトリ）
```

---

## index.html の内部構造

シングルファイルの中を役割ごとに整理すると以下の通りです。

### 定数・設定（変更頻度: 低）

```js
const FD = [...]         // 施設定義（id, name, col, row）
const CONN = [...]       // 施設間の接続関係
const CX = [...]         // SVGのX座標（列ごと）
const RY = [...]         // SVGのY座標（行ごと）
const R = 40             // 施設円のRadius
const BASE_TICK = 900    // 1ティックのミリ秒（速度×1時）
const UNIT_HP = 10       // 1ユニットのHP（損失計算に使用）
const MAX_U = 15         // バー表示の上限ユニット数
const EP = [...]         // 敵のプリセット配置（施設IDインデックス）
```

---

### 状態（State）

```js
let phase = 'setup'      // 'setup' | 'running' | 'paused' | 'done'
let tick = 0             // 経過ティック数
let spd = 1              // 再生速度倍率
let tid = null           // setIntervalのID（タイマー管理）
let aStrV = 3            // 味方ユニット強度
let eStrV = 3            // 敵ユニット強度
let facs = []            // 現在の施設状態の配列
let alloc = {}           // セットアップ時の味方配置数 { [facilityId]: count }
```

#### 施設オブジェクト（facs の要素）

```js
{
  id: 0,               // 施設ID (0-8)
  name: '北西砦',       // 表示名
  col: 0,              // 列番号 (0=左, 1=中, 2=右)
  row: 0,              // 行番号 (0=上, 1=中, 2=下)
  x: 110,              // SVG X座標
  y: 100,              // SVG Y座標
  owner: 'ally',       // 占領状態 'ally' | 'enemy' | 'neutral'
  ally: {
    count: 6,          // 残存兵数（小数あり）
    str: 3             // 攻撃強度
  },
  enemy: {
    count: 0,
    str: 3
  }
}
```

---

### 関数一覧

| 関数名 | 役割 | 変更の影響範囲 |
|--------|------|----------------|
| `mkFacs()` | 施設の初期状態を生成 | 全体 |
| `mkAlloc()` | セットアップの初期配置を生成 | セットアップUI |
| `doStep(facs)` | **1ティック分の戦闘計算**（コア） | シミュレーション結果 |
| `draw()` | SVGを現在の状態で再描画 | 表示のみ |
| `svgEl(tag, attrs, text)` | SVG要素を生成するユーティリティ | 描画処理 |
| `buildSetup()` | セットアップUIを構築 | セットアップUI |
| `updBtns()` | ボタンの表示/非表示を更新 | ボタンUI |
| `startTick()` | タイマーを開始（setInterval） | タイマー |
| `doStart()` | 開始ボタンのハンドラ | 状態遷移 |
| `doPause()` | 一時停止のハンドラ | 状態遷移 |
| `doResume()` | 再開のハンドラ | 状態遷移 |
| `doReset()` | リセットのハンドラ | 状態遷移 |
| `setSpd(v)` | 速度変更のハンドラ | タイマー |

---

### 戦闘計算の詳細（`doStep`）

```js
function doStep(facs) {
  return facs.map(f => {
    // 片方が0なら計算不要（変化なし）
    if (f.ally.count <= 0 || f.enemy.count <= 0) return f;

    // ランチェスター線形則
    const aLoss = (f.enemy.count * f.enemy.str) / UNIT_HP;  // 味方の損失
    const eLoss = (f.ally.count * f.ally.str) / UNIT_HP;    // 敵の損失

    const newAlly = Math.max(0, f.ally.count - aLoss);
    const newEnemy = Math.max(0, f.enemy.count - eLoss);

    // 占領状態を更新
    let owner = f.owner;
    if (newAlly > 0 && newEnemy <= 0) owner = 'ally';
    else if (newEnemy > 0 && newAlly <= 0) owner = 'enemy';

    return { ...f, ally: {...f.ally, count: newAlly}, enemy: {...f.enemy, count: newEnemy}, owner };
  });
}
```

**戦闘モデルを変更したい場合は、この関数のみを修正します。**
他の関数には触れる必要はありません。

---

## 将来のReact移行を見越した設計方針

`doStep()` はpure function（副作用なし）として実装されています。
React移行時にそのまま `useCombatStep` カスタムフックに移植できます。

```jsx
// 将来的な移植イメージ
function useBattleSimulator() {
  const [facs, setFacs] = useState(mkFacs());
  const [phase, setPhase] = useState('setup');

  const tick = useCallback(() => {
    setFacs(prev => doStep(prev));  // doStep はそのまま使える
  }, []);

  useEffect(() => {
    if (phase !== 'running') return;
    const id = setInterval(tick, BASE_TICK / spd);
    return () => clearInterval(id);
  }, [phase, spd, tick]);

  return { facs, phase, ... };
}
```

---

## SVGレイアウトの座標系

```
viewBox="0 0 680 470"

        col=0    col=1    col=2
        x=110    x=340    x=570
row=0   [0]------[1]------[2]    y=100
 y=100   |        |        |
         |        |        |
row=1   [3]------[4]------[5]    y=240
 y=240   |        |        |
         |        |        |
row=2   [6]------[7]------[8]    y=380
 y=380

各施設: 半径R=40の円
左バー (味方): x = cx-R-20, width=11
右バー (敵):  x = cx+R+9,  width=11
バー高さ: count/MAX_U × (R×2-6) px
```

