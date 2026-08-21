# EMS 即時資料與 VueFlow 拓撲整理

## 1. 文件目的

本文件整理目前 EMS WebSocket JSON 資料，以及兩份 VueFlow 圖形程式的資料來源、設備節點、連線與狀態顏色邏輯。

目前兩份程式分別代表：

1. **電力／設備拓撲圖**：高壓併接點、VCB、變壓器、PCS、BMS、Rack、DCP 與太陽能設備。
2. **通訊／網路拓撲圖**：EMS 主機、Switch、Mgate、IO、Meter、PCS、BMS、逆變器及各種通訊線路。

---

## 2. 即時 JSON 的主要結構

目前 JSON 最外層包含以下資料區塊：

| 區塊 | 用途 | 目前資料量 |
|---|---|---:|
| `pcs` | PCS 逆變器即時資料 | 2 |
| `bms` | BMU/BMS 電池管理資料 | 4 |
| `meter` | 電表資料 | 8 |
| `io` | IO、ATS、UPS 數位訊號 | 3 |
| `fan` | PCS 風扇訊號 | 1 |
| `relay` | 保護電驛與斷路器 | 7 |
| `tr` | 變壓器溫度 | 3 |
| `ups` | UPS 狀態 | 1 |
| `dcu` | 太陽能 DCU | 0 |
| `solar_exposure` | 日照計 | 0 |
| `alarm` | 通訊或設備告警 | 0 |
| `env_status` | 環境狀態 | 0 |
| `ems_schedule` | EMS 充放電排程 | 12 |
| `sr_schedule` | 即時備轉排程 | 160 |
| `other` | EMS 模式與系統狀態 | 1 |

設備資料通常共用以下欄位：

| 欄位 | 說明 |
|---|---|
| `device_type` | 設備類型，例如 `pcs`、`bmu`、`meter` |
| `equipment_id` | 設備識別碼，例如 `PCS_1`、`BMU_1` |
| `parent_id` | 上層設備識別碼，例如 BMS 對應 PCS |
| `field_id` | 場域或站點識別碼 |
| `resource_id` | 資源或租戶識別碼 |
| `case_id` | 案件、情境或排程識別碼 |
| `abnormal` | 是否被判斷為異常 |
| `warning` | 結構化告警清單，`[]` 表示目前沒有告警 |
| `data_time` | 該設備資料的時間 |

`null` 應解讀為「未提供或不適用」，不要直接當成 0。

---

## 3. 電力拓撲圖：第一份 VueFlow 程式

### 3.1 程式用途

第一份程式透過 `elements` computed property，動態建立 VueFlow 的 nodes 與 edges。

使用的自訂節點包括：

| VueFlow node type | 元件 | 用途 |
|---|---|---|
| `teleportable` | `TeleportableNode` | 一般可移動設備節點 |
| `PCSportable` | `PCStableNode` | PCS 資訊節點 |
| `TRportable` | `TRtableNode` | 變壓器資訊節點 |
| `BMSportable` | `BMStableNode` | BMS 資訊節點 |
| `RACKportable` | `RACKtableNode` | Rack 資訊節點 |
| `INVportable` | `INVtableNode` | 太陽能逆變器節點 |
| `DCUportable` | `DCUtableNode` | DCU 節點 |
| `Relayportable` | `RelaytableNode` | 保護電驛節點 |
| `custom` | `CustomNode` | 自訂狀態節點 |
| `color` | `CustomColorNode` | 顏色或外框節點 |

資料來源是 Vuex 的：

```js
SocketDataALL
```

### 3.2 儲能電力路徑

程式建立的主要電力路徑如下：

```text
廠內高壓併接點
    ↓
VCB-BESS盤
    ↓
BESS-TR
    ↓
MP-BESS盤
    ↓
PCS
    ↓
DCP_1 / DCP_2
    ↓
BMS_1 / BMS_2
    ↓
Rack1～Rack6
```

其中：

- `PCS` 節點實際讀取 `pcs` 中的 `equipment_id = PCS_1`。
- `BMS_1` 讀取 `BMU_1`。
- `BMS_2` 讀取 `BMU_2`。
- 每個 BMS 節點再往下連接 `rack_1`～`rack_6`。
- Rack 顏色由 `rack_state`、`warning` 與 `abnormal` 決定。

### 3.3 太陽能路徑

程式另外建立太陽能路徑：

```text
廠內低壓盤 MP1
    ↓
AC METER
    ↓
DCU 1
    ↓
INV1 → INV2 → INV3
```

太陽能流向使用：

```js
METER_SUN.total_active_power
```

這份程式的判斷是：

```js
total_active_power > 0 ? 'normal' : 'reverse'
```

### 3.4 儲能功率流向

儲能流向使用：

```js
METER_CABINET.total_active_power
```

目前程式判斷：

```js
total_active_power < 0 ? 'reverse' : 'normal'
```

依目前資料的慣例：

| `total_active_power` | 意義 |
|---:|---|
| 正值 | 從電網取電，通常代表充電方向 |
| 負值 | 向電網送電，通常代表放電方向 |
| `0`、`null` | 不顯示流向動畫 |

### 3.5 設備顏色邏輯

PCS、BMS、Rack 的顏色來自：

```js
EquipmentStatusColor.json
```

主要規則如下：

1. 告警等級大於或等於 2，顯示紅色。
2. 設備狀態對應 `danger`，顯示紅色。
3. `abnormal = true` 時，PCS/BMS 通常顯示黃色或紅色，依函式而定。
4. 正常充電、放電、運作中通常顯示綠色。
5. 找不到設備資料時，顯示深色或關閉狀態。

---

## 4. 通訊拓撲圖：第二份 VueFlow 程式

### 4.1 程式用途

第二份程式不是主要呈現電力流向，而是呈現設備通訊架構及通訊異常狀態。

使用的主要節點類型：

| node type | 用途 |
|---|---|
| `color` | 靜態設備、文字、IP、外框與圖例 |
| `communicate` | 有通訊狀態的設備節點 |

使用的自訂線路：

```js
customLine → CustomLine.vue
```

### 4.2 `statusMap` 產生方式

第二份程式目前只從：

```js
SocketDataALL.value.alarm
```

建立 `statusMap`。

當告警內容包含「通訊異常」時，會設定：

```js
statusMap[equipment_id].abnormal = true
statusMap[equipment_id].comm = true
```

因此第二份圖的紅色狀態主要來自 `alarm`，不是直接讀取 PCS、BMS、IO 或 Meter 物件本身的 `abnormal` 欄位。

### 4.3 通訊線路顏色

| 顏色 | 變數 | 線路用途 |
|---|---|---|
| `#0057A0` | `color_cat6` | Cat6 網路線 |
| `#008000` | `color_fiber` | 多模光纖 |
| `#000000` | `color_rs485` | RS485 |
| `#A0A0A0` | `color_vga` | VGA、PS/2 |
| `#DDA0DD` | `color_dio` | DO/DI |
| `#CC0000` | - | 通訊異常圖例 |

### 4.4 通訊架構概念

```text
EMS 主機／資料收集設備
        ↓
Switch／光電 Hub／Mgate
        ↓
PCS、BMS、Meter、Relay、UPS、DCU、IO
        ↓
現場設備、太陽能逆變器與即時備轉 PLC
```

第二份圖還包含：

- Fortinet 防火牆。
- 中華電信外部網路。
- 2F 圖控室。
- 即時備轉廠內 PLC 接點。
- IP 位址標示。
- Cat6、光纖、RS485、VGA+PS/2、DO/DI 圖例。

---

## 5. JSON 與 VueFlow 的設備 ID 對照

### 5.1 目前可以直接對應

| VueFlow 程式使用的 ID | JSON 對應 |
|---|---|
| `PCS_1` | `pcs[].equipment_id = PCS_1` |
| `BMU_1` | `bms[].equipment_id = BMU_1` |
| `BMU_2` | `bms[].equipment_id = BMU_2` |
| `METER_CABINET` | `meter[].equipment_id = METER_CABINET` |
| `METER_MAIN` | `meter[].equipment_id = METER_MAIN` |
| `METER_SUN` | `meter[].equipment_id = METER_SUN` |
| `IO_INTERNET` | `io[].equipment_id = IO_INTERNET` |
| `IO_TCP` | `io[].equipment_id = IO_TCP` |
| `UPS` | `ups[].equipment_id = UPS` |

### 5.2 目前對不起來或沒有資料的 ID

| 程式使用的 ID | 目前 JSON 情況 | 影響 |
|---|---|---|
| `TR` | JSON 是 `TR_MP_BESS_1`、`TR_MP_1`、`TR_MP_BESS_2` | `TRdata` 找不到資料 |
| `METER_VCB` | JSON 是 `METER_VCB_1`、`METER_VCB_2` | VCB Meter 找不到資料 |
| `METER_FACTORY` | JSON 沒有此 ID | 低壓盤 Meter 找不到資料 |
| `VCB` | JSON 是 `RELAY_VCB_3`、`RELAY_VCB_BESS_1` 等 | VCB Relay 找不到資料 |
| `DCP` | 目前 JSON 沒有 `dcp` 區塊 | DCP 狀態無法由目前資料更新 |
| `DCU_1` | `dcu` 目前是空陣列 | DCU 狀態無法更新 |
| `EXPOSURE` | `solar_exposure` 目前是空陣列 | 日照計狀態無法更新 |
| `INVERTER_1`～`INVERTER_3` | 目前 JSON 沒有對應資料 | 太陽能逆變器狀態只能靠告警或靜態節點 |
| `VDF_1`、`VDF_2` | 目前 JSON 沒有對應資料 | 變頻器狀態只能靠告警 |
| `DCP_DEHU` | 目前 JSON 沒有對應資料 | 除濕機狀態只能靠告警 |
| `FORTI`、`SWITCH` | 目前 JSON 沒有對應資料 | 網路設備狀態只能靠告警 |

### 5.3 目前最重要的狀態來源問題

目前 JSON 中：

```text
IO_INTERNET.abnormal = true
IO_TCP.abnormal      = true
alarm = []
```

但第二份程式只從 `alarm` 建立 `statusMap`，所以即使 IO 物件本身已經是 `abnormal = true`，第二份通訊圖仍可能顯示為正常。

建議將設備本身的 `abnormal` 也納入 `statusMap`，例如：

```js
const statusMap = {};

for (const group of ['pcs', 'bms', 'meter', 'io', 'relay', 'tr', 'ups', 'fan']) {
  for (const item of SocketDataALL.value?.[group] ?? []) {
    if (!item.equipment_id) continue;
    statusMap[item.equipment_id] = {
      abnormal: item.abnormal === true,
      comm: item.abnormal === true,
    };
  }
}
```

再用 `alarm` 補充較詳細的通訊告警原因。

---

## 6. 目前 JSON 的狀態摘要

從目前貼出的資料可整理為：

| 項目 | 狀態 |
|---|---|
| PCS | `PCS_1`、`PCS_2` 都是 `abnormal = false` |
| PCS 故障 | `active_fault = 0`、故障與告警狀態皆為 OFF |
| PCS 運轉 | `running_status = 電池已連接` |
| BMS | 4 台皆為 `abnormal = false` |
| BMS 運轉 | `status = 放電` |
| BMS SOC | 約 11% |
| BMS SOH | 100% |
| IO | `IO_INTERNET`、`IO_TCP` 為異常；`IO_ATS` 正常 |
| 排程模式 | `other.operation_mode = 0`，EMS 控制 |
| EMS 狀態 | `other.ems_status = 5`，排程充放電 |
| 未提供區塊 | `dcu`、`solar_exposure`、`alarm`、`env_status` 目前為空 |

另外，PCS 的 `dc_input_current_m1` 中重複出現 `-3276.8`，應視為無效資料或未接入通道，不宜直接納入電流計算。

---

## 7. 建議的資料層整理方式

建議將設備 ID 與資料區塊集中管理，避免在 VueFlow 中散落大量字串：

```js
const EQUIPMENT = {
  pcs: {
    main: 'PCS_1',
  },
  bms: {
    left: 'BMU_1',
    right: 'BMU_2',
  },
  meter: {
    cabinet: 'METER_CABINET',
    main: 'METER_MAIN',
    solar: 'METER_SUN',
    vcb1: 'METER_VCB_1',
    vcb2: 'METER_VCB_2',
  },
  transformer: {
    bess1: 'TR_MP_BESS_1',
    mp1: 'TR_MP_1',
    bess2: 'TR_MP_BESS_2',
  },
};
```

並統一使用查找函式：

```js
function getEquipment(group, equipmentId) {
  return SocketDataALL.value?.[group]
    ?.find(item => item.equipment_id === equipmentId);
}
```

這樣可以降低以下問題：

- `TR` 與 `TR_MP_BESS_1` 命名不一致。
- `METER_VCB` 與 `METER_VCB_1`、`METER_VCB_2` 命名不一致。
- 程式讀取不存在的 `dcp`、`METER_FACTORY` 或 `VCB`。
- JSON 增加設備時需要修改很多 VueFlow 節點。

---

## 8. 程式碼檢查清單

1. 將第二份程式的通訊判斷整合設備物件的 `abnormal`。
2. 統一 `TR`、`VCB`、`METER_VCB`、`METER_FACTORY` 等設備 ID。
3. 確認 `dcp` 是否應該改成 `dcu`，或後端是否需要補回 `dcp` 區塊。
4. 確認太陽能逆變器是否應在 JSON 增加 `INVERTER_1`～`INVERTER_3`。
5. 第二份程式有重複的 edge ID：`1U 機架型LCD KVM__EMS主機2`，應改成唯一 ID。
6. 所有通訊設備最好同時提供：`abnormal`、`data_time`、`warning` 與通訊錯誤原因。
7. `dc_input_current_m1` 的 `-3276.8` 建議在資料轉換層轉成 `null`，不要讓前端自行判斷。

