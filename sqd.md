```mermaid
sequenceDiagram
    autonumber
    actor Customer as 🧑 Customer
    participant VueApp as 🖥️ Vue Frontend (Browser)
    participant IndexedDB as 📦 IndexedDB (Local Storage)
    participant LocalBridge as ⚙️ Local Hardware Bridge
    participant OdooAPI as ☁️ Odoo 19 API (Backend)
    participant BankGW as 🏦 Thai Payment GW (PromptPay)
    participant MDB as 🔩 MDB Hardware (Motors)

    note over Customer, MDB: === PHASE 1: STARTUP & SYNCHRONIZATION (Machine Boot) ===
    
    VueApp->>+OdooAPI: POST /vending/sync/products (Request latest master data)
    alt Internet is available
        OdooAPI-->>-VueApp: Return JSON [Products, Prices, Images, Categories]
        VueApp->>IndexedDB: Bulk Put (Save master data locally)
        VueApp->>VueApp: Set State to "Ready"
    else Internet unavailable
        VueApp->>VueApp: Show "Out of Service / Cached Mode" Warning
    end

    note over Customer, MDB: === PHASE 2: BROWSING & CART (Offline Capable) ===

    Customer->>VueApp: Touches Screen (Start Shopping)
    VueApp->>+IndexedDB: Query all products
    IndexedDB-->>-VueApp: Return local product list instantly
    VueApp-->>Customer: Display Product Grid UI
    
    Customer->>VueApp: Taps "Add Coke (Slot A1)"
    VueApp->>VueApp: Update local cart state (Total: ฿20)

    note over Customer, MDB: === PHASE 3: PAYMENT PROCESSING (Requires Internet) ===

    Customer->>VueApp: Taps "Checkout with PromptPay"
    
    alt Internet is OFF
        VueApp-->>Customer: Show Error "Cannot connect for payment. Cash only."
    else Internet is ON
        VueApp->>+BankGW: API Request Dynamic QR (Amount: 20.00, Ref: ORD-123)
        BankGW-->>-VueApp: Return QR Payload/String
        VueApp-->>Customer: Display QR Code on Screen
        
        par Parallel Actions: Payment & Monitoring
            Customer->>BankGW: Scans QR with Mobile Banking App & Pays
            
            loop Every 3 seconds (Polling)
                VueApp->>+OdooAPI: Check payment status for Ref ORD-123
                note right of OdooAPI: Odoo waits for Bank Webhook
                OdooAPI-->>-VueApp: Status: "Pending"
            end
        and Bank Webhook
            BankGW->>+OdooAPI: Webhook POST: Payment Success (Ref: ORD-123)
            OdooAPI->>OdooAPI: Mark internal order as "Paid"
            OdooAPI-->>-BankGW: Acknowledge (200 OK)
        end
        
        VueApp->>+OdooAPI: Final Poll Check for Ref ORD-123
        OdooAPI-->>-VueApp: Status: "PAID" (Success confirmed)
    end

    note over Customer, MDB: === PHASE 4: DISPENSING & RECORDING ===

    VueApp->>VueApp: Show "Dispensing..." UI
    VueApp->>+LocalBridge: POST http://localhost:port/dispense {slot: "A1"}
    LocalBridge->>+MDB: Send Serial Command to Turn Motor A1
    MDB->>MDB: Turn Motor, Drop Product
    MDB-->>-LocalBridge: Signal: Vend Success
    LocalBridge-->>-VueApp: HTTP 200 OK: Dispense Successful
    
    VueApp-->>Customer: Show "Please Collect Item Below" Screen
    
    note right of VueApp: Async Background Task
    VueApp->>OdooAPI: POST /vending/order/finalize {items: ["A1"], total: 20.00}
    OdooAPI->>OdooAPI: Create permanent POS Order record & Deduct Inventory
