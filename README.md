html
<html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ETHIC PAY - Multi-Sig Approval</title>
    <link rel="stylesheet" href="style.css">
    <style>
        .approval-container {
            background: linear-gradient(135deg, #1a1a2e, #2d3436);
            border-radius: 20px;
            padding: 30px;
            border: 1px solid #ffd700;
        }
        
        .request-card {
            background: #2a2a2a;
            border-radius: 15px;
            padding: 20px;
            margin: 20px 0;
            border: 1px solid #333;
        }
        
        .request-card.high-value {
            border-color: #ff4444;
        }
        
        .detail-row {
            display: flex;
            justify-content: space-between;
            padding: 10px 0;
            border-bottom: 1px solid #222;
        }
        
        .detail-row:last-child {
            border-bottom: none;
        }
        
        .detail-label {
            color: #888;
        }
        
        .detail-value {
            color: #fff;
            font-weight: bold;
        }
        
        .signature-progress {
            margin: 20px 0;
        }
        
        .progress-bar {
            height: 10px;
            background: #333;
            border-radius: 5px;
            overflow: hidden;
        }
        
        .progress-fill {
            height: 100%;
            background: linear-gradient(90deg, #ffd700, #ff6b35);
            transition: width 0.5s;
        }
        
        .sign-btn {
            width: 100%;
            padding: 15px;
            background: linear-gradient(135deg, #ffd700, #ff6b35);
            color: #1a1a1a;
            border: none;
            border-radius: 10px;
            font-size: 1.1em;
            font-weight: bold;
            cursor: pointer;
            transition: transform 0.3s;
        }
        
        .sign-btn:hover {
            transform: scale(1.02);
        }
        
        .sign-btn:disabled {
            background: #444;

color: #888;
            cursor: not-allowed;
        }
        
        .signer-list {
            display: flex;
            gap: 10px;
            margin: 20px 0;
        }
        
        .signer-item {
            padding: 10px;
            background: #333;
            border-radius: 10px;
            text-align: center;
            flex: 1;
        }
        
        .signer-item.signed {
            background: #00ff0020;
            border: 1px solid #00ff00;
        }
        
        .signer-item.pending {
            background: #ffd70020;
            border: 1px solid #ffd700;
        }
        
        .signer-icon {
            font-size: 2em;
            margin-bottom: 5px;
        }
        
        .signer-name {
            color: #fff;
            font-size: 0.9em;
        }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <div class="logo">
                <h1>🔐 ETHIC PAY</h1>
                <span class="subtitle">Multi-Signature Approval</span>
            </div>
            <div class="status">
                <span style="color: #ffd700;">🔑 2 of 3 Required</span>
            </div>
        </header>
        
        <div class="approval-container">
            <h2 style="color: #ffd700; margin-bottom: 20px;">Pending Approval Requests</h2>
            
            <!-- Request 1 -->
            <div class="request-card high-value">
                <div style="display: flex; justify-content: space-between; margin-bottom: 15px;">
                    <h3 style="color: #fff;">🚨 High Value Transaction</h3>
                    <span style="background: #ff4444; padding: 4px 8px; border-radius: 5px; font-size: 0.8em;">URGENT</span>
                </div>
                
                <div class="detail-row">
                    <span class="detail-label">Request ID</span>
                    <span class="detail-value">REQ-A1B2C3D4</span>
                </div>
                <div class="detail-row">
                    <span class="detail-label">Amount</span>
                    <span class="detail-value" style="color: #ffd700;">10,000 USDT</span>
                </div>
                <div class="detail-row">
                    <span class="detail-label">To Address</span>
                    <span class="detail-value" style="font-family: monospace; font-size: 0.85em;">TXYZ...abcd</span>
                </div>
                <div class="detail-row">
                    <span class="detail-label">Requested By</span>
                    <span class="detail-value">Golf 🏌️</span>
                </div>
                <div class="detail-row">
                    <span class="detail-label">Note</span>
                    <span class="detail-value">Payment for Q1 services</span>
                </div>
                <div class="detail-row">
                    <span class="detail-label">Created</span>
                    <span class="detail-value">2025-02-01 14:30</span>
                </div>
                <div class="detail-row">
                    <span class="detail-label">Expires</span>
                    <span class="detail-value" style="color: #ff4444;">In 12 hours</span>
                </div>
                
                <div class="signature-progress">
                    <div style="display: flex; justify-content: space-between; margin-bottom: 8px;">
                        <span style="color: #888;">Signatures:</span>
                        <span style="color: #ffd700;">1 / 2</span>
                    </div>
                    <div class="progress-bar">
                        <div class="progress-fill" style="width: 50%;"></div>
                    </div>
                </div>
                
                <div class="signer-list">

<div class="signer-item signed">
                        <div class="signer-icon">👑</div>
                        <div class="signer-name">Boss</div>
                        <div style="color: #00ff00; font-size: 0.8em;">✅ Signed</div>
                    </div>
                    <div class="signer-item pending">
                        <div class="signer-icon">🏌️</div>
                        <div class="signer-name">Golf</div>
                        <div style="color: #ffd700; font-size: 0.8em;">⏳ Pending</div>
                    </div>
                    <div class="signer-item pending">
                        <div class="signer-icon">⚖️</div>
                        <div class="signer-name">Lawyer</div>
                        <div style="color: #ffd700; font-size: 0.8em;">⏳ Pending</div>
                    </div>
                </div>
                
                <button class="sign-btn" onclick="signTransaction('REQ-A1B2C3D4')">
                    🔐 Approve & Sign
                </button>
            </div>
            
            <!-- Request 2 -->
            <div class="request-card">
                <h3 style="color: #fff; margin-bottom: 15px;">Standard Transaction</h3>

/ Start countdown
        countdownInterval = setInterval(updateTimer, 1000);

          // Initial load
        updateQR();
    </script>
</body>
</html>
```

---

## 📜 ไฟล์ใหม่ที่ 5: `frontend/multisig_approve.html

