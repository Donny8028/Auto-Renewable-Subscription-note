# Server to Server Notification
關於自動續訂, 由Apple在2017年釋出的功能, 能夠在apple store connect 設定自己server的endpoint, 以做到能接收到由Apple提供的通知, 並從通知中瞭解User的訂閱狀態。

### Type

| Name | Purpose |
| ------ | ------ |
| INITIAL_BUY | 用戶第一次購買的通知 |
| INTERACTIVE_RENEWAL | 用戶在使用App時或在設定選擇訂閱 |
| DID_CHANGE_RENEWAL_PREF | 用戶降級時 |
| CANCEL | 用戶取消或請求退款 |
| DID_CHANGE_RENEWAL_STATUS | 用戶是否將自動續訂關閉 |
| DID_FAIL_TO_RENEW | <span style="color:green;font-weight:bold">首次</span>自動續訂失敗 |
| DID_RECOVER | 用戶恢復購買/續訂|
| PRICE_INCREASE_CONSENT | 當價格調漲時, 用戶是否同意 |

*PRICE_INCREASE_CONSENT包含 price_increase_effective_date, 此欄位為用戶必須同意漲價的最後日期
*PRICE_INCREASE_CONSENT會依訂閱日期在過期7/10/30 天之前, Apple會檢查是否漲價
*DID_RECOVER 將取代 RENEWAL 通知, 目前兩者都會發送

### Comunicating

當Apple嘗試通知你時, 你的server只需要回應Status code 200, 不然不然Apple server 最多會嘗試三次重試, 或者你能回應Status code 50X / 40X 讓Apple server知道需要再重新嘗試。

情境：首次購買
通知：INITIAL_BUY
```json
{
latest_receipt_info: {
    purchase_date_ms: String //訂閱日
    original_transaction_id: String //此訂閱者的訂閱的唯一標識符
    web_order_line_item_id: String // 每個訂閱週期的唯一標識符, 可將此對應到varifyReceipt作為entry
    product_id: String //產品id
    }
}
```

情境：用戶升級
通知：CANCEL -> INTERACTIVE_RENEWAL 
```json
{
latest_receipt_info: {
    purchase_date_ms: String //訂閱日
    original_transaction_id: String //此訂閱者的訂閱的唯一標識符
    web_order_line_item_id: String // 每個訂閱週期的唯一標識符, 可將此對應到varifyReceipt作為entry
    product_id: String //產品id
    }
}
```

情境：用戶重新訂閱
通知：INTERACTIVE_RENEWAL
```json
{
latest_receipt_info: {
    purchase_date_ms: String //訂閱日
    original_transaction_id: String //此訂閱者的訂閱的唯一標識符
    web_order_line_item_id: String // 每個訂閱週期的唯一標識符, 可將此對應到varifyReceipt作為entry
    product_id: String //產品id
    }
}
```
通知：DID_CHANGE_RENEWAL_STATUS

```json
{
auto_renew_status_change_date_ms: String
auto_renew_status: String
latest_receipt_info: {
    original_transaction_id: String //此訂閱者的訂閱的唯一標識符
    product_id: String //產品id
    }
}
```

情境：用戶關閉自動續訂功能
通知：DID_CHANGE_RENEWAL_STATUS

```json
{
auto_renew_status_change_date_ms: String //狀態更改日期
auto_renew_status: String //是否關閉自動續訂
latest_receipt_info: {
    original_transaction_id: String //此訂閱者的訂閱的唯一標識符
    product_id: String //產品id
    }
}
```

情境：用戶降級
通知：DID_CHANGE_RENEWAL_PREF
```json
{
auto_renew_product_id: String // 下一次訂閱生效的產品id
latest_receipt_info: {
    original_transaction_id: String //此訂閱者的訂閱的唯一標識符
}
}
```

情境：用戶取消申請退款/用戶升級
通知：CANCEL
```json
{
cancellation_date_ms: String // 取消日期
latest_receipt_info: {
    original_transaction_id: String //此訂閱者的訂閱的唯一標識符
    product_id: String //產品id
}
}
```

情境：用戶第一次續訂失敗
通知：DID_FAIL_TO_RENEW

```json
{
pending_renewal_info: {
    is_in_billing_retry_period: String // 是不是正在Apple嘗試扣款的期間
}
latest_receipt_info: {
    original_transaction_id: String //此訂閱者的訂閱的唯一標識符
    expires_date_ms: String //到期日
}
}
```

情境：用戶續訂失敗後, 又成功扣款續訂（在is_in_billing_retry_period期間）
通知：DID_RECOVER

```json
{
latest_receipt_info: {
    original_transaction_id: String //此訂閱者的訂閱的唯一標識符
    expires_date_ms: String //到期日
    purchase_date_ms: String // 購買日
}
}
```

情境：用戶是否同意漲價/一更動價格時/ 當用戶進入價格調漲流程時
通知：PRICE_INCREASE_CONSENT

```json
// 當一更動價格時, 會馬上發出這個通知, 則price_consent_status都會是flase(代表用戶尚未回覆)
{
pending_renewal_info: {
    price_consent_status: String // 是否同意漲價
}
latest_receipt_info: {
    original_transaction_id: String //此訂閱者的訂閱的唯一標識符
    expires_date_ms: String //到期日
}
}
```


#### 規範

**EndPoint**
- 需要遵守ATS
- 證書需要由受信任的證書機構發行
- TLS 1.2
- 使用其中一種對稱加密算法(AES-128/AES-256)
- 證書須由SHA-256或更佳的算法來簽署
- 需要遵守ATS

**Receipt Data**
- 將此當成Token
- 以此來驗證收據
- 需儲存在Server


#### 通知欄位

- unified_receipt_info (包含所有購買的資料與信息/只有最新的100筆)
- enviroment (Sandbox/Production)
- latest_receipt
- latest_receipt_info
- pending_renewal_info
- status

#### 寬限期
可以設定16天的寬限期, 讓訂閱天數不會中斷
在/verifyReceipt中
多了新欄位`grace_period_expires_date`

#### Apple Review
Review時, apple測試人員將會嘗試以SandBox環境購買, 此時server如果以Production的/verifyReceipt打此次購買, 將會收到status: 21007, 代表驗證的data為SandBox.

#### 增加付款體驗(optional)
- 能新增可點擊的button, 導向調整信用卡設定的頁面
- 告知用戶正在嘗試扣款, 但失敗
- 向用戶告知已在寬限期內, 寬限期過後服務將會終止

------------

最後編輯日期: 2020/06/02
