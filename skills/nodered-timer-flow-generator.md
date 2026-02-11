# Node-RED タイマーフロー自動生成スキル

> 統合済み: nodered-timeslot-generator の内容を含む

Node-REDタイマーフローを自動生成する。日出/日入連動、シーケンス制御対応。
灌水シーケンス、換気制御、照明制御などの時間制御フローに対応。

---

## 1. 概要

### 1.1 目的

施設園芸で使用されるタイマー制御フローをNode-REDで実装する。
シーケンス制御（複数動作の順次実行）と日出/日入連動に対応。

### 1.2 対応する制御パターン

| パターン | 説明 | 例 |
|---------|------|-----|
| 単発タイマー | 指定時刻に1回実行 | 6:00に換気扇ON |
| 周期タイマー | 一定間隔で繰り返し | 30分毎に灌水 |
| シーケンス | 複数動作を順次実行 | 灌水→待機→液肥→待機 |
| 日出/日入連動 | 日出/日入基準の相対時刻 | 日出30分後にカーテン開 |
| 条件付きタイマー | 条件成立時のみ実行 | 晴天時のみ遮光 |

### 1.3 シーケンス制御とは

複数のアクチュエータを順番に制御する方式。

```
例: 灌水シーケンス
  1. 主バルブON
  2. 3秒待機
  3. 系統1バルブON
  4. 60秒灌水
  5. 系統1バルブOFF
  6. 系統2バルブON
  7. 60秒灌水
  8. 系統2バルブOFF
  9. 主バルブOFF
```

---

## 2. 使用方法

### 2.1 入力パラメータ

```yaml
# タイマー定義
timer:
  name: "朝灌水"
  type: sequence  # single | periodic | sequence
  trigger:
    type: sunrise_relative  # fixed | sunrise_relative | sunset_relative | periodic
    offset: 30  # 分（日出30分後）

  # シーケンス定義
  sequence:
    - action: relay_on
      target: main_valve
      delay: 0

    - action: wait
      duration: 3  # 秒

    - action: relay_on
      target: zone1_valve
      delay: 0

    - action: wait
      duration: 60

    - action: relay_off
      target: zone1_valve
      delay: 0

    - action: relay_on
      target: zone2_valve
      delay: 0

    - action: wait
      duration: 60

    - action: relay_off
      target: zone2_valve
      delay: 0

    - action: relay_off
      target: main_valve
      delay: 0

# アクチュエータ定義
actuators:
  main_valve:
    type: relay
    mqtt_topic: "farm/actuator/main_valve"
    on_value: 1
    off_value: 0

  zone1_valve:
    type: relay
    mqtt_topic: "farm/actuator/zone1"
    on_value: 1
    off_value: 0

  zone2_valve:
    type: relay
    mqtt_topic: "farm/actuator/zone2"
    on_value: 1
    off_value: 0
```

### 2.2 出力形式

- Node-REDフローJSON
- シーケンス状態管理用DBスキーマ
- MQTTトピック設計

---

## 3. Node-REDフロー設計

### 3.1 シーケンス制御フロー構成

```
[トリガー]
    │
    ▼
[シーケンス開始]
    │
    ▼
[ステップ1実行] → [MQTT送信]
    │
    ▼
[delay] ────→ [ステップ2実行] → [MQTT送信]
                  │
                  ▼
              [delay] ────→ [ステップN実行]
                                │
                                ▼
                          [シーケンス完了]
```

### 3.2 必要ノード

| ノード | 用途 |
|--------|------|
| inject / cron-plus | タイマートリガー |
| function | シーケンス制御ロジック |
| delay | 待機時間 |
| switch | 条件分岐 |
| MQTT out | アクチュエータ制御 |
| link in/out | フロー間連携 |

---

## 4. シーケンス制御コード

### 4.1 シーケンスコントローラー functionノード

```javascript
// シーケンスコントローラー
// グローバル変数でシーケンス状態を管理

const sequence = [
    { action: "relay_on", target: "main_valve", topic: "farm/actuator/main_valve", value: 1 },
    { action: "wait", duration: 3000 },
    { action: "relay_on", target: "zone1_valve", topic: "farm/actuator/zone1", value: 1 },
    { action: "wait", duration: 60000 },
    { action: "relay_off", target: "zone1_valve", topic: "farm/actuator/zone1", value: 0 },
    { action: "relay_on", target: "zone2_valve", topic: "farm/actuator/zone2", value: 1 },
    { action: "wait", duration: 60000 },
    { action: "relay_off", target: "zone2_valve", topic: "farm/actuator/zone2", value: 0 },
    { action: "relay_off", target: "main_valve", topic: "farm/actuator/main_valve", value: 0 }
];

// 状態取得
let state = flow.get("sequence_state") || { running: false, step: 0 };

// シーケンス開始
if (msg.payload === "start" && !state.running) {
    state = { running: true, step: 0, startTime: Date.now() };
    flow.set("sequence_state", state);
    node.status({ fill: "green", shape: "dot", text: "実行中" });
}

// シーケンス停止（緊急停止）
if (msg.payload === "stop") {
    state.running = false;
    flow.set("sequence_state", state);
    // 全アクチュエータOFF
    return [
        { topic: "farm/actuator/main_valve", payload: 0 },
        { topic: "farm/actuator/zone1", payload: 0 },
        { topic: "farm/actuator/zone2", payload: 0 }
    ];
}

// シーケンス次ステップ
if (msg.payload === "next" && state.running) {
    if (state.step >= sequence.length) {
        // シーケンス完了
        state.running = false;
        flow.set("sequence_state", state);
        node.status({ fill: "grey", shape: "ring", text: "完了" });
        return { payload: "sequence_complete" };
    }

    const currentStep = sequence[state.step];
    state.step++;
    flow.set("sequence_state", state);

    if (currentStep.action === "wait") {
        // 待機: delayノードへ
        return [null, { payload: currentStep.duration, delay: currentStep.duration }];
    } else {
        // アクチュエータ制御: MQTTへ
        node.status({ fill: "green", shape: "dot", text: `Step ${state.step}: ${currentStep.target}` });
        return [{ topic: currentStep.topic, payload: currentStep.value }, null];
    }
}

return null;
```

### 4.2 シーケンス実行フロー

```javascript
// シーケンス実行管理
// inject → このノード → [MQTT, delay] → link back

const SEQUENCE_NAME = "朝灌水";

// シーケンス定義
const sequences = {
    "朝灌水": {
        steps: [
            { type: "on", relay: "main_valve", mqtt: "farm/act/main" },
            { type: "delay", ms: 3000 },
            { type: "on", relay: "zone1", mqtt: "farm/act/zone1" },
            { type: "delay", ms: 60000 },
            { type: "off", relay: "zone1", mqtt: "farm/act/zone1" },
            { type: "on", relay: "zone2", mqtt: "farm/act/zone2" },
            { type: "delay", ms: 60000 },
            { type: "off", relay: "zone2", mqtt: "farm/act/zone2" },
            { type: "off", relay: "main_valve", mqtt: "farm/act/main" }
        ]
    }
};

// 現在のステップを取得
let step = context.get("step") || 0;
const seq = sequences[SEQUENCE_NAME];

if (msg.topic === "start") {
    step = 0;
    context.set("step", step);
    context.set("running", true);
}

if (!context.get("running")) {
    return null;
}

if (step >= seq.steps.length) {
    context.set("running", false);
    context.set("step", 0);
    node.status({ fill: "grey", shape: "ring", text: "完了" });
    return { payload: { status: "complete", sequence: SEQUENCE_NAME } };
}

const current = seq.steps[step];
context.set("step", step + 1);

if (current.type === "delay") {
    // delayノードへ送信し、完了後link backで戻る
    return [null, { delay: current.ms, payload: "continue" }];
}

// リレー制御
const value = current.type === "on" ? 1 : 0;
node.status({ fill: "green", shape: "dot", text: `${current.relay}: ${current.type}` });

// MQTT送信後、即座に次ステップへ
return [{ topic: current.mqtt, payload: value }, { payload: "continue" }];
```

---

## 5. 日出/日入連動タイマー

### 5.1 日出/日入計算＋スケジュール更新

```javascript
// 毎日0:05に実行し、当日のスケジュールを設定

const lat = 35.6762;
const lon = 139.6503;

// 簡易日出計算（省略版）
function calcSun(date, latitude, longitude) {
    // ... 計算ロジック（前スキル参照）
    return { sunrise: new Date(), sunset: new Date() };
}

const today = new Date();
const sun = calcSun(today, lat, lon);

// タイマー設定
const timers = [
    {
        name: "朝灌水",
        base: "sunrise",
        offset: 30,  // 日出30分後
        action: "start_sequence",
        sequence: "morning_irrigation"
    },
    {
        name: "夕方灌水",
        base: "sunset",
        offset: -60,  // 日入60分前
        action: "start_sequence",
        sequence: "evening_irrigation"
    },
    {
        name: "カーテン開",
        base: "sunrise",
        offset: 0,
        action: "relay_on",
        target: "curtain"
    },
    {
        name: "カーテン閉",
        base: "sunset",
        offset: 30,  // 日入30分後
        action: "relay_off",
        target: "curtain"
    }
];

// 実行時刻を計算
const scheduledTimers = timers.map(t => {
    const baseTime = t.base === "sunrise" ? sun.sunrise : sun.sunset;
    const execTime = new Date(baseTime.getTime() + t.offset * 60000);
    return {
        ...t,
        execTime: execTime,
        cron: `${execTime.getMinutes()} ${execTime.getHours()} * * *`
    };
});

// cron-plusへスケジュール送信
msg.payload = {
    command: "replace-all",
    schedules: scheduledTimers.map(t => ({
        name: t.name,
        expression: t.cron,
        payload: {
            action: t.action,
            sequence: t.sequence,
            target: t.target
        }
    }))
};

// ログ出力
node.warn(`日出: ${sun.sunrise.toTimeString().slice(0,5)}, 日入: ${sun.sunset.toTimeString().slice(0,5)}`);
node.warn(`スケジュール更新: ${scheduledTimers.length}件`);

return msg;
```

---

## 6. DBスキーマ（状態管理用）

### 6.1 SQLite スキーマ

```sql
-- タイマー定義テーブル
CREATE TABLE timers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    type TEXT NOT NULL,  -- 'single', 'periodic', 'sequence'
    trigger_type TEXT NOT NULL,  -- 'fixed', 'sunrise', 'sunset', 'periodic'
    trigger_time TEXT,  -- HH:MM または offset分
    enabled INTEGER DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- シーケンス定義テーブル
CREATE TABLE sequences (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timer_id INTEGER NOT NULL,
    step_order INTEGER NOT NULL,
    action_type TEXT NOT NULL,  -- 'relay_on', 'relay_off', 'wait'
    target TEXT,  -- アクチュエータID
    duration INTEGER,  -- 待機時間(ms)
    FOREIGN KEY (timer_id) REFERENCES timers(id)
);

-- 実行ログテーブル
CREATE TABLE execution_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timer_id INTEGER NOT NULL,
    started_at DATETIME NOT NULL,
    completed_at DATETIME,
    status TEXT,  -- 'running', 'completed', 'error', 'cancelled'
    error_message TEXT,
    FOREIGN KEY (timer_id) REFERENCES timers(id)
);

-- アクチュエータ定義テーブル
CREATE TABLE actuators (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    type TEXT NOT NULL,  -- 'relay', 'valve', 'motor'
    mqtt_topic TEXT NOT NULL,
    on_value TEXT DEFAULT '1',
    off_value TEXT DEFAULT '0'
);

-- インデックス
CREATE INDEX idx_timers_enabled ON timers(enabled);
CREATE INDEX idx_sequences_timer ON sequences(timer_id, step_order);
CREATE INDEX idx_logs_timer ON execution_logs(timer_id, started_at);
```

### 6.2 初期データ例

```sql
-- アクチュエータ登録
INSERT INTO actuators (id, name, type, mqtt_topic) VALUES
('main_valve', '主バルブ', 'valve', 'farm/actuator/main_valve'),
('zone1', '系統1バルブ', 'valve', 'farm/actuator/zone1'),
('zone2', '系統2バルブ', 'valve', 'farm/actuator/zone2'),
('curtain', 'カーテン', 'motor', 'farm/actuator/curtain');

-- タイマー登録
INSERT INTO timers (name, type, trigger_type, trigger_time) VALUES
('朝灌水', 'sequence', 'sunrise', '30'),
('夕方灌水', 'sequence', 'sunset', '-60');

-- シーケンス登録（朝灌水）
INSERT INTO sequences (timer_id, step_order, action_type, target, duration) VALUES
(1, 1, 'relay_on', 'main_valve', NULL),
(1, 2, 'wait', NULL, 3000),
(1, 3, 'relay_on', 'zone1', NULL),
(1, 4, 'wait', NULL, 60000),
(1, 5, 'relay_off', 'zone1', NULL),
(1, 6, 'relay_on', 'zone2', NULL),
(1, 7, 'wait', NULL, 60000),
(1, 8, 'relay_off', 'zone2', NULL),
(1, 9, 'relay_off', 'main_valve', NULL);
```

---

## 7. 完全なフローJSON

### 7.1 シーケンス灌水フロー

```json
[
    {
        "id": "irrigation_trigger",
        "type": "inject",
        "name": "灌水開始",
        "props": [{"p": "topic", "vt": "str", "v": "start"}],
        "repeat": "",
        "crontab": "30 06 * * *",
        "wires": [["sequence_controller"]]
    },
    {
        "id": "sequence_controller",
        "type": "function",
        "name": "シーケンス制御",
        "func": "/* 上記のシーケンスコントローラーコード */",
        "outputs": 2,
        "wires": [["mqtt_out"], ["delay_node"]]
    },
    {
        "id": "delay_node",
        "type": "delay",
        "name": "待機",
        "pauseType": "delayv",
        "timeout": "1",
        "timeoutUnits": "milliseconds",
        "rate": "1",
        "wires": [["link_back"]]
    },
    {
        "id": "link_back",
        "type": "link out",
        "name": "次ステップへ",
        "links": ["link_in_next"],
        "wires": []
    },
    {
        "id": "link_in_next",
        "type": "link in",
        "name": "次ステップ受信",
        "links": ["link_back"],
        "wires": [["sequence_next"]]
    },
    {
        "id": "sequence_next",
        "type": "change",
        "name": "次ステップ指示",
        "rules": [{"t": "set", "p": "payload", "pt": "msg", "to": "next", "tot": "str"}],
        "wires": [["sequence_controller"]]
    },
    {
        "id": "mqtt_out",
        "type": "mqtt out",
        "name": "アクチュエータ制御",
        "topic": "",
        "broker": "mqtt_broker",
        "wires": [["sequence_next"]]
    },
    {
        "id": "emergency_stop",
        "type": "inject",
        "name": "緊急停止",
        "props": [{"p": "payload", "vt": "str", "v": "stop"}],
        "wires": [["sequence_controller"]]
    }
]
```

---

## 8. MQTTトピック設計

### 8.1 トピック構造

```
farm/
├── actuator/           # アクチュエータ制御
│   ├── main_valve      # 主バルブ (0/1)
│   ├── zone1           # 系統1 (0/1)
│   ├── zone2           # 系統2 (0/1)
│   └── curtain         # カーテン (0-100)
├── sequence/           # シーケンス制御
│   ├── start           # シーケンス開始
│   ├── stop            # シーケンス停止
│   └── status          # シーケンス状態
└── timer/              # タイマー管理
    ├── enable          # タイマー有効化
    ├── disable         # タイマー無効化
    └── status          # タイマー状態
```

### 8.2 ペイロード形式

```json
// シーケンス開始
{
    "topic": "farm/sequence/start",
    "payload": {
        "sequence": "morning_irrigation",
        "triggered_by": "cron",
        "timestamp": "2026-02-04T06:30:00"
    }
}

// シーケンス状態
{
    "topic": "farm/sequence/status",
    "payload": {
        "sequence": "morning_irrigation",
        "status": "running",
        "step": 3,
        "total_steps": 9,
        "current_action": "zone1灌水中"
    }
}
```

---

## 9. エラーハンドリング

### 9.1 タイムアウト処理

```javascript
// シーケンス実行タイムアウト監視
const MAX_SEQUENCE_TIME = 10 * 60 * 1000;  // 10分

const state = flow.get("sequence_state");
if (state && state.running) {
    const elapsed = Date.now() - state.startTime;
    if (elapsed > MAX_SEQUENCE_TIME) {
        // タイムアウト: 強制停止
        flow.set("sequence_state", { running: false, error: "timeout" });
        node.error("シーケンスタイムアウト: 強制停止");

        // 全アクチュエータOFF
        return [
            { topic: "farm/actuator/main_valve", payload: 0 },
            { topic: "farm/actuator/zone1", payload: 0 },
            { topic: "farm/actuator/zone2", payload: 0 }
        ];
    }
}
```

### 9.2 重複実行防止

```javascript
// シーケンス開始時の重複チェック
const state = flow.get("sequence_state");

if (msg.payload === "start") {
    if (state && state.running) {
        node.warn("シーケンス実行中のため開始をスキップ");
        return null;
    }
    // 新規開始
    flow.set("sequence_state", { running: true, step: 0, startTime: Date.now() });
}
```

---

## 10. 注意事項

### 10.1 安全対策

| 項目 | 対策 |
|------|------|
| 緊急停止 | 物理スイッチ + ソフトウェア停止の2重化 |
| タイムアウト | 最大実行時間を設定し、超過時は強制停止 |
| 重複実行 | 同一シーケンスの重複実行を防止 |
| 電源断 | 再起動時は全アクチュエータOFFから開始 |

### 10.2 Node-REDの制限

| 項目 | 注意点 |
|------|--------|
| メモリ | 大量のシーケンスを同時実行しない |
| delay | delayノードは再起動でリセットされる |
| context | flow/global contextはメモリ上（永続化設定推奨） |

---

## 11. 参考資料

- Node-RED: https://nodered.org/
- node-red-contrib-cron-plus: https://flows.nodered.org/node/node-red-contrib-cron-plus
- MQTT: https://mqtt.org/

---

## 統合元からの補足: nodered-timeslot-generator

### 農業用8時間帯タイマー設定の生成パターン

nodered-timeslot-generator は、農業特有の「8時間帯変温管理」に特化したフロー生成機能を提供します。

#### 8時間帯制御とは

1日を8つの時間帯に分割し、各時間帯で異なる目標値（温度、湿度、CO2等）を設定する制御方式。

| # | 時間帯 | 開始基準 | 農業的意義 |
|---|--------|----------|-----------|
| 1 | 夜間前半 | 日入 | 呼吸抑制期 - 日中の光合成産物を消費しない |
| 2 | 夜間後半 | 日入+4h | 低温管理期 - 徒長防止、花芽分化促進 |
| 3 | 明け方 | 日出-2h | 最低温度期 - 1日の最低温度を維持 |
| 4 | 早朝 | 日出 | 暖房開始期 - 光合成開始に合わせて加温 |
| 5 | 午前 | 日出+2h | 光合成活発期 - 最も活発な光合成 |
| 6 | 真昼 | 南中-2h | 高温管理期 - CO2施用効果大 |
| 7 | 午後 | 南中+2h | 転流促進期 - 同化産物を果実へ転流 |
| 8 | 夕方 | 日入-2h | 暖房停止準備 - 徐々に温度を下げる |

#### 8時間帯フロー構成

```
[1分毎inject]
      │
      ▼
[sun-position] ─── 日出/日入/南中時刻取得
      │
      ▼
[時間帯判定function] ─── msg.timeslot = 1-8
      │
      ▼
[switch分岐] ─┬─ 時間帯1 → [目標値1設定] ──┐
              ├─ 時間帯2 → [目標値2設定] ──┤
              ├─ ...                       │
              └─ 時間帯8 → [目標値8設定] ──┤
                                           │
                                           ▼
                                    [MQTT送信]
```

#### 時間帯判定ロジック（functionノード）

```javascript
// 8時間帯判定（日出/日入/南中連動）
const now = new Date();
const sunData = msg.payload[0]; // sun_times
const timeslots = msg.timeslots; // SQLiteから取得した時間帯設定

// sun_timesから時刻をパース
function parseTime(timeStr) {
    const [h, m, s] = timeStr.split(':').map(Number);
    const d = new Date();
    d.setHours(h, m, s || 0, 0);
    return d;
}

const sunrise = parseTime(sunData.sunrise);
const sunset = parseTime(sunData.sunset);
const solarNoon = parseTime(sunData.solar_noon);

// オフセットを適用した実際の開始時刻を計算
function getSlotStartTime(slot) {
    let baseTime;
    switch (slot.start_type) {
        case 'sunrise':
            baseTime = new Date(sunrise);
            break;
        case 'sunset':
            baseTime = new Date(sunset);
            break;
        case 'solar_noon':
            baseTime = new Date(solarNoon);
            break;
        case 'fixed':
            return parseTime(slot.start_time);
        default:
            return parseTime('00:00:00');
    }
    return new Date(baseTime.getTime() + slot.start_offset * 60000);
}

// 全スロットの開始時刻を計算
const slotTimes = timeslots.map(slot => ({
    ...slot,
    actualStart: getSlotStartTime(slot)
})).sort((a, b) => a.actualStart - b.actualStart);

// 現在の時間帯を判定
let currentSlot = slotTimes[slotTimes.length - 1];
for (let i = 0; i < slotTimes.length; i++) {
    const nextIdx = (i + 1) % slotTimes.length;
    const start = slotTimes[i].actualStart;
    const end = slotTimes[nextIdx].actualStart;

    if (end < start) {
        if (now >= start || now < end) {
            currentSlot = slotTimes[i];
            break;
        }
    } else {
        if (now >= start && now < end) {
            currentSlot = slotTimes[i];
            break;
        }
    }
}

msg.timeslot = currentSlot.slot_number;
msg.targetTemp = currentSlot.target_temp;
msg.targetHumidity = currentSlot.target_humidity;
msg.payload = {
    slot: currentSlot.slot_number,
    name: currentSlot.slot_name,
    targets: {
        temperature: currentSlot.target_temp,
        humidity: currentSlot.target_humidity,
        co2: currentSlot.target_co2
    }
};

return msg;
```

#### 目標値設定テンプレート（トマト夏期）

```yaml
# トマト夏期設定
crop: tomato
season: summer
timeslots:
  - slot: 1  # 夜間前半（呼吸抑制）
    temp: 22
    humid: 70
    co2: 400
    note: "日中の光合成産物を消費しないよう、やや高めの温度"

  - slot: 2  # 夜間後半（低温管理）
    temp: 20
    humid: 75
    co2: 400
    note: "徒長防止、花芽分化促進"

  - slot: 3  # 明け方（最低温度）
    temp: 18
    humid: 80
    co2: 400
    note: "1日の最低温度維持"

  - slot: 4  # 早朝（暖房開始）
    temp: 22
    humid: 70
    co2: 600
    note: "光合成開始に合わせて加温・CO2施用開始"

  - slot: 5  # 午前（光合成活発）
    temp: 26
    humid: 65
    co2: 1000
    note: "最も光合成が活発、CO2施用効果大"

  - slot: 6  # 真昼（高温管理）
    temp: 28
    humid: 60
    co2: 800
    note: "高温ストレス回避しつつ光合成維持"

  - slot: 7  # 午後（転流促進）
    temp: 26
    humid: 65
    co2: 600
    note: "同化産物を果実へ転流促進"

  - slot: 8  # 夕方（暖房停止準備）
    temp: 24
    humid: 70
    co2: 400
    note: "徐々に温度を下げ、夜間管理へ移行"
```

#### SQLiteスキーマ（タイマー設定永続化）

```sql
-- timeslotsテーブル
CREATE TABLE IF NOT EXISTS timeslots (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    name            TEXT NOT NULL,           -- 設定名（例: 夏期設定）
    slot_number     INTEGER NOT NULL CHECK (slot_number BETWEEN 1 AND 8),
    slot_name       TEXT,                    -- 時間帯名（例: 夜間前半）

    -- 開始時刻設定
    start_type      TEXT NOT NULL DEFAULT 'fixed',
    start_time      TEXT,                    -- HH:MM (fixed時)
    start_offset    INTEGER DEFAULT 0,       -- 分単位オフセット

    -- 目標値
    target_temp     REAL,                    -- 目標温度（℃）
    target_humidity REAL,                    -- 目標湿度（%）
    target_co2      INTEGER,                 -- 目標CO2（ppm）

    -- 有効/無効
    enabled         INTEGER DEFAULT 1,
    description     TEXT,
    created_at      TEXT DEFAULT (datetime('now', 'localtime')),
    updated_at      TEXT DEFAULT (datetime('now', 'localtime')),

    UNIQUE (name, slot_number)
);

-- sun_timesテーブル
CREATE TABLE IF NOT EXISTS sun_times (
    date        TEXT PRIMARY KEY,
    sunrise     TEXT NOT NULL,
    sunset      TEXT NOT NULL,
    solar_noon  TEXT NOT NULL,
    latitude    REAL,
    longitude   REAL,
    updated_at  TEXT DEFAULT (datetime('now', 'localtime'))
);
```

#### 必要なNode-REDノード

```bash
cd ~/.node-red

# スケジューラ
npm install node-red-contrib-cron-plus

# 太陽位置計算
npm install node-red-contrib-sun-position

# SQLite（タイマー設定永続化）
npm install node-red-node-sqlite

# Node-RED再起動
node-red-restart
```

#### 注意事項

**日出/日入計算の精度**
- sun-positionノードは高精度（±1分以内）
- 簡易計算式使用時は±2分程度の誤差あり

**季節変動**
- 冬至/夏至で日出時刻は約3時間変動
- 日出/日入連動により自動対応

**時間帯境界でのチャタリング防止**
- 1分間隔での判定でヒステリシスを確保
- 必要に応じて判定間隔を調整

この補足により、一般的なタイマー制御（シーケンス制御）と農業特化の8時間帯タイマーの両方に対応可能となります。
