# 🧠 1. KONSEP FINAL (ARSITEKTUR SISTEM)

graph TD
    subgraph External [Sumber Eksternal]
        WS_Indodax[WebSocket Indodax<br>wss://ws3.indodax.com]
        REST_Indodax[REST API Indodax<br>https://indodax.com/tapi]
    end

    subgraph Core [Inti Sistem - Komputer Lokal]
        subgraph Go_Services [Go Services]
            Ingestor[Ingestor<br>WebSocket Client]
            Executor[Executor<br>Order & Risk Manager]
        end

        subgraph Python_Services [Python Services]
            Strategy[Strategy Engine<br>Feature Engineering]
            subgraph Rust_Accel [Rust Acceleration Layer]
                Calc[Fast Math Lib<br>RSI/MACD/Imbalance]
            end
            Strategy -- FFI Call --> Calc
        end

        subgraph Node_Services [Node.js Services]
            WS_Server[WebSocket Server<br>Socket.IO / ws]
            API_Server[REST API Server<br>Express]
        end

        subgraph Data_Layer [Data & State Layer]
            Redis[(Redis Server)]
        end

        subgraph UI [User Interface]
            Browser[Dashboard Browser<br>React / Vue / Svelte]
            Telegram[Notifikasi Telegram]
        end
    end

    %% Koneksi
    WS_Indodax -- Data Stream --> Ingestor
    Ingestor -- Publish market:BTCIDR.* --> Redis
    Redis -- Subscribe market:* --> Strategy
    Strategy -- Publish signal:BTCIDR --> Redis
    Redis -- Subscribe signal:* --> Executor
    Executor -- Order Request --> REST_Indodax
    Executor -- Publish order:status --> Redis
    Executor -- Update State --> Redis

    Redis -- Subscribe * --> WS_Server
    WS_Server -- Push Data --> Browser
    API_Server -- Query --> Redis
    Browser -- REST API --> API_Server
    Strategy -- Error/Alert --> Telegram


## 🔷 PARADIGMA UTAMA

Sistem kamu bukan “bot trading biasa”, tapi:

> **Event-driven distributed trading system (mini quant infrastructure)**

Semua komunikasi berbasis **event streaming**, bukan polling.

---

## ⚙️ ALUR BESAR SISTEM

```text id="flow1"
[ Market Data (Indodax WebSocket) ]
                ↓
        ┌─────────────────┐
        │ Ingestor (Go)   │  → normalisasi tick data
        └─────────────────┘
                ↓
        ┌─────────────────┐
        │ Redis (Event Bus│  ← pusat sistem
        └─────────────────┘
                ↓
   ┌──────────────────────────┐
   │ Strategy Engine (Python) │ → analisis indikator
   └──────────────────────────┘
                ↓
        signal: BUY / SELL
                ↓
   ┌──────────────────────────┐
   │ Execution Engine (Go)    │ → risk check + order
   └──────────────────────────┘
                ↓
        Indodax REST API
                ↓
   ┌──────────────────────────┐
   │ State Manager            │ → posisi, PnL, recovery
   └──────────────────────────┘
                ↓
   ┌──────────────────────────┐
   │ Dashboard (Node.js)      │ → monitoring realtime
   └──────────────────────────┘
```

---

# 🧩 2. PRINSIP DESAIN FINAL

## 🔥 1. SEPARATION OF CONCERNS

* Go → execution + ingestion (low latency)
* Python → logic / intelligence
* Node.js → UI + realtime display
* Redis → satu-satunya sumber event

---

## ⚡ 2. EVENT-DRIVEN ONLY

Tidak ada service yang “saling call langsung”.

Semua lewat Redis:

```text id="event1"
market.tick.BTCIDR
signal.BTCIDR
order.executed
position.updated
```

---

## 🧠 3. STATELESS SERVICE (kecuali state manager)

* Ingestor = stateless
* Strategy = stateless
* Executor = semi-stateful
* State Manager = satu-satunya sumber truth

---

## 🛡️ 4. RISK FIRST DESIGN

Semua order harus lewat:

```text id="risk1"
Signal → Risk Check → Position Sizing → Execution
```

---

# 📁 3. STRUKTUR FINAL GITHUB REPOSITORY

Ini versi clean dan production-ready:

```text id="repo1"
bot-scalping/
│
├── README.md
├── .gitignore
├── docker-compose.yml
├── .env.example
│
├── config/
│   ├── settings.yaml        # strategy + risk config
│   ├── pairs.yaml           # trading pairs
│
├── infra/
│   ├── redis/
│   │   └── redis.conf
│   ├── docker/
│   │   └── services.yml
│
├── state/
│   ├── positions.json       # open positions
│   ├── equity.json          # balance history
│
├── logs/
│   ├── ingestor/
│   ├── strategy/
│   ├── executor/
│
├── services/
│   │
│   ├── ingestor-go/
│   │   ├── cmd/main.go
│   │   ├── internal/
│   │   │   ├── websocket/
│   │   │   ├── redis/
│   │   │   └── parser/
│   │
│   ├── strategy-python/
│   │   ├── main.py
│   │   ├── engine/
│   │   │   ├── indicators.py
│   │   │   ├── signals.py
│   │   │   ├── orderbook.py
│   │
│   ├── executor-go/
│   │   ├── cmd/main.go
│   │   ├── internal/
│   │   │   ├── risk/
│   │   │   ├── order/
│   │   │   ├── state/
│   │
│   ├── dashboard-node/
│   │   ├── server.js
│   │   ├── public/
│   │   │   ├── index.html
│   │   │   ├── app.js
│
├── libs/
│   ├── shared-types/
│   ├── utils/
│
├── tests/
│   ├── backtest/
│   ├── paper-trade/
│
└── docs/
    ├── architecture.md
    ├── trading-strategy.md
```

---

# 🧠 4. PERAN SETIAP SERVICE (FINAL LOGIC)

## 🟢 Ingestor (Go)

* connect WebSocket Indodax
* normalize tick
* publish ke Redis

---

## 🟣 Strategy Engine (Python)

* hitung:

  * RSI
  * MACD
  * VWAP
  * orderbook imbalance
* generate signal

---

## 🔴 Executor (Go)

* receive signal
* risk validation:

  * max loss
  * position size
  * cooldown
* execute order

---

## 🔵 State Manager

* track:

  * posisi aktif
  * PnL
  * recovery state

---

## 🟡 Dashboard (Node.js)

* realtime chart
* signal stream
* posisi aktif
* alert system

---

# ⚙️ 5. INFRASTRUCTURE FINAL

## Docker stack:

```text id="infra1"
- Redis
- Ingestor service
- Strategy service
- Executor service
- Dashboard service
```

---

# 🧭 6. DATA FLOW FINAL

```text id="dataflow1"
Market → Ingestor → Redis → Strategy → Signal → Executor → Order → State → Dashboard
```

---

# 🧠 7. KARAKTER SISTEM (IMPORTANT DESIGN INTENT)

Sistem ini:

✔ real-time event system
✔ modular microservices
✔ fault tolerant
✔ restart safe (state recovery)
✔ scalable (bisa dipisah VPS nanti)

---

# 🚀 8. ROADMAP IMPLEMENTASI

## PHASE 1 (foundation)

* setup repo
* Redis + Docker
* ingestor

## PHASE 2 (logic)

* strategy engine
* signal system

## PHASE 3 (execution)

* executor + risk system

## PHASE 4 (monitoring)

* dashboard + logs

## PHASE 5 (testing)

* backtest + paper trading

## PHASE 6 (live)

* small capital deployment

---

# 💡 INSIGHT TERPENTING

👉 Kunci sistem ini bukan complexity
👉 tapi **event flow + risk control + state consistency**

---
