# EMS 問題排查流程

建立日期：2026/08/15（← 0815 新增）

## 一、固定排查順序

```mermaid
flowchart LR
    A["① 現場設備 / Modbus Poll"] --> B["② Reader / 批次 Log"]
    B --> C["③ MongoDB / 歷史檔案"]
    C --> D["④ Backend API"]
    D --> E["⑤ Frontend 畫面"]
```

原則：從最接近資料來源的位置開始確認，不要直接在前端猜問題。

## 二、常見問題快速表

| 現象 | 先檢查 | 下一步 |
|---|---|---|
| 前端沒有資料 | API 是否有回傳資料 | API 有資料：查前端欄位；API 無資料：查資料庫 |
| API 回傳空資料 | MongoDB／歷史檔是否有資料 | 無資料：查 Reader、批次與設備通訊 |
| 設備讀不到 | IP、網路線、控制櫃燈號、Modbus 設定 | 核對 Address、Quantity、設備點位表 |
| Network Error | Reverse Proxy、Docker network、服務名稱 | 查 `access.log`、Container 狀態與 Port |
| 報表沒有資料 | 報表批次 Log、來源檔、報表 Collection | 確認案場版本及 API 查詢日期 |
| 首頁投標資料異常 | `cases_info`、`sbspm` 與首頁 API | 不要只查 `job_10` 的 `sr_schedule` |
| 通知群組收不到 | 雲端群組、地端 Token、批次 Token | 分別測試兩個群組及對應 Job／Service |
| PCS／BMS 狀態異常 | 現場實值、保護點位、批次 Log | 確認是否有批次下達控制命令 |

## 三、硬體通訊檢查

### ABB REX615

- 前方 RJ-45 綠燈：網路線已成功連接。
- 黃燈閃爍：正在與連接設備通訊。
- 若看到的是交換器燈號，需依交換器型號判斷，不可直接套用 REX615 定義。

### Modbus Poll

1. `Connection → Connection Setup`：確認設備／Proxy IP。
2. `Setup → Read/Write Definition`：確認 Address 與 Quantity。
3. 按下 Apply 後，比對現場、Modbus Poll、API 與前端數值。
4. 寫入前再次確認設備、功能碼、Address、允許值與停止方式。

## 四、設備測試紀錄格式

| 欄位 | 紀錄內容 |
|---|---|
| 測試設備 | PCS／BMS／電表／VCB 等 |
| 測試時間 | 開始與結束時間 |
| 測試條件 | 模式、前置狀態、現場允許範圍 |
| 設定值 | Address、功率、SOC、Quantity 等 |
| 實際值 | 設備、Modbus、API、前端顯示值 |
| 結果 | 通過／失敗／待確認 |
| 復原確認 | 功率歸零、模式恢復、告警解除 |

## 五、安全提醒

- `job_00`、`job_01` 與充放電控制類批次可能寫入 PCS／BMS。
- 不確定點位或設備狀態時，只做讀取測試。
- 測試前確認停止條件，測試後確認設備已回到安全狀態。

相關文件：

- [系統整體架構](./SYSTEM_ARCHITECTURE.md)
- [批次、API 與畫面對照](./BATCH_API_UI_MAPPING.md)
- [完整批次程式盤點](./EMS_BATCH_OVERVIEW.md)
