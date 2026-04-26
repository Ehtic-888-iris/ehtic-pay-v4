func (h *AdminHandler) GetPendingApprovals(w http.ResponseWriter, r *http.Request) {
 rows, err := h.DB.Query(
  SELECT 
   ua.id,
   ua.payment_intent_id,
   ua.expected_amount,
   ua.actual_amount,
   ua.short_amount,
   ua.tx_hash,
   ua.wallet_address,
   ua.created_at,
   pi.status as intent_status
  FROM underpaid_approvals ua
  JOIN payment_intents pi ON pi.id = ua.payment_intent_id
  WHERE ua.status = 'pending'
  ORDER BY ua.created_at DESC
 )
 if err != nil {
  http.Error(w, "Query failed", http.StatusInternalServerError)
  return
 }
 defer rows.Close()

 type Approval struct {
  ID             string    json:"id"
  IntentID       string    json:"intent_id"
  ExpectedAmount string    json:"expected_amount"
  ActualAmount   string    json:"actual_amount"
  ShortAmount    string    json:"short_amount"
  TxHash         string    json:"tx_hash"
  WalletAddress  string    json:"wallet_address"
  CreatedAt      time.Time json:"created_at"
  Status         string    json:"status"
 }

 var approvals []Approval
 for rows.Next() {
  var a Approval
  rows.Scan(&a.ID, &a.IntentID, &a.ExpectedAmount, &a.ActualAmount, 
   &a.ShortAmount, &a.TxHash, &a.WalletAddress, &a.CreatedAt, &a.Status)
  approvals = append(approvals, a)
 }

 w.Header().Set("Content-Type", "application/json")
 json.NewEncoder(w).Encode(approvals)
}

## **6. Notification Service (notification/alert.go)**
go
package notification

import (
 "bytes"
 "fmt"
 "log"
 "net/smtp"
 "text/template"

 "payment-system/internal/config"

 "github.com/shopspring/decimal"
)

type AlertService struct {
 Config *config.Config
}

func NewAlertService(cfg *config.Config) *AlertService {
 return &AlertService{
  Config: cfg,
 }
}

func (s *AlertService) SendUnderpaidAlert(intentID string, expected, actual decimal.Decimal) {
 short := expected.Sub(actual)

 // Slack notification
 if s.Config.SlackWebhookURL != "" {
  message := fmt.Sprintf(
🚨 *แจ้งเตือนยอดเงินไม่ครบ* 🚨

📋 Intent ID: %s
💰 ยอดที่คาดหวัง: %s USDT
💵 ยอดที่ได้รับ: %s USDT
📉 ยอดขาด: %s USDT

🔗 อนุมัติ: http://%s/admin/approve/%s
❌ ปฏิเสธ: http://%s/admin/reject/%s
, intentID, expected.String(), actual.String(), short.String(),
   s.Config.AdminPort, intentID, s.Config.AdminPort, intentID)

  s.sendSlack(message)
 }

 // Email notification
 if s.Config.SMTPHost != "" {
  subject := fmt.Sprintf("🚨 ต้องการอนุมัติ: ยอดชำระไม่ครบ %s USDT", short.String())
  body := fmt.Sprintf(
<html>
<body>
<h2>แจ้งเตือนยอดเงินไม่ครบ</h2>
<p>Intent ID: %s</p>
<p>ยอดที่คาดหวัง: %s USDT</p>
<p>ยอดที่ได้รับ: %s USDT</p>
<p>ยอดขาด: %s USDT</p>
<a href="http://%s/admin/approve/%s">✅ อนุมัติ</a>
<a href="http://%s/admin/reject/%s">❌ ปฏิเสธ</a>
</body>
</html>
, intentID, expected.String(), actual.String(), short.String(),
   s.Config.Admin
