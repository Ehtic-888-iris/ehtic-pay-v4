ht-pay/
├── main.go
├── go.mod
├── go.sum
├── config/
│   └── config.go
├── database/
│   ├── database.go
│   ├── migrations/
│   │   ├── 001_create_users.sql
│   │   ├── 002_create_payments.sql
│   │   └── 003_create_transactions.sql
│   └── seed.go
├── handlers/
│   ├── auth.go
│   ├── payments.go
│   ├── websocket.go
│   └── transactions.go
├── middleware/
│   ├── auth.go
│   ├── cors.go
│   └── logging.go
├── models/
│   ├── user.go
│   ├── payment.go
│   └── transaction.go
├── services/
│   ├── auth_service.go
│   ├── payment_service.go
│   └── blockchain_service.go
└── utils/
    ├── encryption.go
    ├── validator.go
    └── logger.go
```

## 1. **ไฟล์หลัก - main.go (อัปเกรดแล้ว)**

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "ht-pay/config"
    "ht-pay/database"
    "ht-pay/handlers"
    "ht-pay/middleware"
    "github.com/gorilla/mux"
    "go.uber.org/zap"
)

var logger *zap.Logger

func init() {
    var err error
    logger, err = zap.NewProduction()
    if err != nil {
        log.Fatal("Failed to initialize logger:", err)
    }
}

func main() {
    // โหลด config
    cfg := config.Load()

    // เชื่อมต่อ Database
    db, err := database.Connect(cfg)
    if err != nil {
        logger.Fatal("Database connection failed", zap.Error(err))
    }
    defer db.Close()

    // ทำ Migration
    database.Migrate(db)

    // สร้าง Router
    router := mux.NewRouter()

    // Middleware Chain
    router.Use(middleware.CORS)
    router.Use(middleware.RequestLogging(logger))
    router.Use(middleware.RateLimiter(100, time.Minute))

    // Public Routes
    router.HandleFunc("/api/v1/auth/signin", handlers.SignIn(db, cfg)).Methods("POST")
    router.HandleFunc("/api/v1/auth/signup", handlers.SignUp(db, cfg)).Methods("POST")
    router.HandleFunc("/api/v1/payment_intents", handlers.CreatePaymentIntent(db)).Methods("POST")
    router.HandleFunc("/api/v1/payment_intents/{id}", handlers.GetPaymentIntent(db)).Methods("GET")
    router.HandleFunc("/api/v1/payment_intents/{id}/submit_tx", handlers.SubmitTransaction(db)).Methods("POST")

    // Protected Routes
    protected := router.PathPrefix("/api/v1").Subrouter()
    protected.Use(middleware.Authenticate(cfg.JWTSecret))
    protected.HandleFunc("/dashboard", handlers.GetDashboard(db)).Methods("GET")
    protected.HandleFunc("/payments", handlers.GetUserPayments(db)).Methods("GET")
    protected.HandleFunc("/transactions", handlers.GetUserTransactions(db)).Methods("GET")

    // WebSocket Route
    router.HandleFunc("/ws", handlers.HandleWebSocket)

    // Health Check
    router.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"status":"healthy"}`))
    }).Methods("GET")

    // Start WebSocket Broadcast Handler
    go handlers.HandleBroadcast()

    // สร้าง HTTP Server
    server := &http.Server{
        Addr:         ":" + cfg.Port,
        Handler:      router,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Graceful Shutdown
    go func() {
        sigChan := make(chan os.Signal, 1)
        signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
        <-sigChan
        
        logger.Info("Shutting down server...")
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        
        if err := server.Shutdown(ctx); err != nil {
            logger.Fatal("Server shutdown failed", zap.Error(err))
        }
    }()

    logger.Info("Server started", zap.String("port", cfg.Port))
    if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        logger.Fatal("Server failed to start", zap.Error(err))
    }
}
```

## 2. **Config - config/config.go**

```go
package config

import (
    "os"
    "strconv"
)

type Config struct {
    Port        string
    JWTSecret   string
    DatabaseURL string
    RedisURL    string
    Environment string
    RateLimit   int
    LogLevel    string
}

func Load() *Config {
    return &Config{
        Port:        getEnv("PORT", "8080"),
        JWTSecret:   getEnv("JWT_SECRET", "your-secret-key-change-in-production"),
        DatabaseURL: getEnv("DATABASE_URL", "postgres://user:password@localhost:5432/htpay"),
        RedisURL:    getEnv("REDIS_URL", "redis://localhost:6379"),
        Environment: getEnv("ENVIRONMENT", "development"),
        RateLimit:   getEnvInt("RATE_LIMIT", 100),
        LogLevel:    getEnv("LOG_LEVEL", "info"),
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if intVal, err := strconv.Atoi(value); err == nil {
            return intVal
        }
    }
    return defaultValue
}
```

## 3. **Database - database/database.go**

```go
package database

import (
    "database/sql"
    "encoding/json"
    "fmt"
    "log"
    "os"
    "path/filepath"
    "sort"
    "time"

    "ht-pay/config"
    _ "github.com/lib/pq"
    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

type Database struct {
    *sql.DB
}

func Connect(cfg *config.Config) (*Database, error) {
    db, err := sql.Open("postgres", cfg.DatabaseURL)
    if err != nil {
        return nil, fmt.Errorf("failed to open database: %w", err)
    }

    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)

    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }

    log.Println("Database connected successfully")
    return &Database{db}, nil
}

func (db *Database) RunMigrations(migrationsDir string) error {
    driver, err := postgres.WithInstance(db.DB, &postgres.Config{})
    if err != nil {
        return fmt.Errorf("failed to create migration driver: %w", err)
    }

    m, err := migrate.NewWithDatabaseInstance(
        "file://"+migrationsDir,
        "postgres",
        driver,
    )
    if err != nil {
        return fmt.Errorf("failed to create migration instance: %w", err)
    }

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return fmt.Errorf("failed to run migrations: %w", err)
    }

    log.Println("Database migrations completed successfully")
    return nil
}

// Cache using Redis-like pattern
type Cache struct {
    data map[string]CacheItem
}

type CacheItem struct {
    Value      interface{}
    Expiration time.Time
}

func NewCache() *Cache {
    return &Cache{
        data: make(map[string]CacheItem),
    }
}

func (c *Cache) Get(key string) (interface{}, bool) {
    item, exists := c.data[key]
    if !exists {
        return nil, false
    }
    if time.Now().After(item.Expiration) {
        delete(c.data, key)
        return nil, false
    }
    return item.Value, true
}

func (c *Cache) Set(key string, value interface{}, duration time.Duration) {
    c.data[key] = CacheItem{
        Value:      value,
        Expiration: time.Now().Add(duration),
    }
}

func (c *Cache) Delete(key string) {
    delete(c.data, key)
}
```

## 4. **Models - models/payment.go**

```go
package models

import (
    "time"
)

type PaymentIntent struct {
    ID                 string    `json:"id"`
    UserID             string    `json:"user_id,omitempty"`
    Chain              string    `json:"chain"`
    TokenSymbol        string    `json:"token_symbol"`
    TokenContract      string    `json:"token_contract"`
    TokenDecimals      int       `json:"token_decimals"`
    ToAddress          string    `json:"to_address"`
    AmountRequested    string    `json:"amount_requested"`
    AmountPaid         string    `json:"amount_paid"`
    Status             string    `json:"status"`
    ExpiresAt          time.Time `json:"expires_at"`
    CreatedAt          time.Time `json:"created_at"`
    UpdatedAt          time.Time `json:"updated_at"`
}

type PaymentRequest struct {
    Chain              string `json:"chain" validate:"required"`
    TokenSymbol        string `json:"token_symbol" validate:"required"`
    TokenContract      string `json:"token_contract" validate:"required"`
    TokenDecimals      int    `json:"token_decimals" validate:"required"`
    ToAddress          string `json:"to_address" validate:"required,eth_addr"`
    AmountRequested    string `json:"amount_requested" validate:"required"`
    ExpiresInMinutes   int    `json:"expires_in_minutes" validate:"min=1"`
}

type Transaction struct {
    ID              string    `json:"id"`
    PaymentIntentID string    `json:"payment_intent_id"`
    TxHash          string    `json:"tx_hash"`
    FromAddress     string    `json:"from_address"`
    Amount          string    `json:"amount"`
    BlockNumber     int64     `json:"block_number"`
    Status          string    `json:"status"`
    CreatedAt       time.Time `json:"created_at"`
}
```

## 5. **Handlers - handlers/payments.go**

```go
package handlers

import (
    "database/sql"
    "encoding/json"
    "log"
    "net/http"
    "time"

    "github.com/google/uuid"
    "github.com/gorilla/mux"
    "ht-pay/database"
    "ht-pay/models"
)

type PaymentHandler struct {
    db    *database.Database
    cache *database.Cache
}

func NewPaymentHandler(db *database.Database) *PaymentHandler {
    return &PaymentHandler{
        db:    db,
        cache: database.NewCache(),
    }
}

func CreatePaymentIntent(w http.ResponseWriter, r *http.Request) {
    var req models.PaymentRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, `{"error":"invalid request body"}`, http.StatusBadRequest)
        return
    }

    // Validate request
    if err := validatePaymentRequest(req); err != nil {
        http.Error(w, `{"error":"`+err.Error()+`"}`, http.StatusBadRequest)
        return
    }

    // Create payment intent
    payment := models.PaymentIntent{
        ID:              uuid.New().String(),
        Chain:           req.Chain,
        TokenSymbol:     req.TokenSymbol,
        TokenContract:   req.TokenContract,
        TokenDecimals:   req.TokenDecimals,
        ToAddress:       req.ToAddress,
        AmountRequested: req.AmountRequested,
        Status:          "pending",
        ExpiresAt:       time.Now().Add(time.Duration(req.ExpiresInMinutes) * time.Minute),
        CreatedAt:       time.Now(),
        UpdatedAt:       time.Now(),
    }

    // Save to database
    query := `INSERT INTO payment_intents (id, chain, token_symbol, token_contract, token_decimals, to_address, amount_requested, status, expires_at, created_at, updated_at) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)`
    _, err := db.Exec(query, payment.ID, payment.Chain, payment.TokenSymbol, payment.TokenContract, payment.TokenDecimals, payment.ToAddress, payment.AmountRequested, payment.Status, payment.ExpiresAt, payment.CreatedAt, payment.UpdatedAt)
    if err != nil {
        log.Printf("Error creating payment intent: %v", err)
        http.Error(w, `{"error":"failed to create payment intent"}`, http.StatusInternalServerError)
        return
    }

    // Send real-time update
    SendPaymentUpdate(PaymentUpdate{
        Type:    "new_payment",
        Payment: payment,
    })

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(payment)
}

func GetPaymentIntent(w http.ResponseWriter, r *http.Request) {
    params := mux.Vars(r)
    paymentID := params["id"]

    // Check cache first
    if cachedPayment, found := cache.Get("payment_" + paymentID); found {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(cachedPayment)
        return
    }

    // Query database
    var payment models.PaymentIntent
    query := `SELECT id, chain, token_symbol, token_contract, token_decimals, to_address, amount_requested, amount_paid, status, expires_at, created_at, updated_at FROM payment_intents WHERE id = $1`
    err := db.QueryRow(query, paymentID).Scan(&payment.ID, &payment.Chain, &payment.TokenSymbol, &payment.TokenContract, &payment.TokenDecimals, &payment.ToAddress, &payment.AmountRequested, &payment.AmountPaid, &payment.Status, &payment.ExpiresAt, &payment.CreatedAt, &payment.UpdatedAt)
    if err == sql.ErrNoRows {
        http.Error(w, `{"error":"payment intent not found"}`, http.StatusNotFound)
        return
    } else if err != nil {
        log.Printf("Error fetching payment intent: %v", err)
        http.Error(w, `{"error":"failed to fetch payment intent"}`, http.StatusInternalServerError)
        return
    }

    // Cache the result
    cache.Set("payment_"+paymentID, payment, 5*time.Minute)

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(payment)
}

func validatePaymentRequest(req models.PaymentRequest) error {
    if req.Chain == "" {
        return fmt.Errorf("chain is required")
    }
    if req.TokenSymbol == "" {
        return fmt.Errorf("token symbol is required")
    }
    if req.TokenContract == "" {
        return fmt.Errorf("token contract is required")
    }
    if req.ToAddress == "" {
        return fmt.Errorf("to address is required")
    }
    if req.AmountRequested == "" {
        return fmt.Errorf("amount requested is required")
    }
    if req.ExpiresInMinutes <= 0 {
        req.ExpiresInMinutes = 15 // Default 15 minutes
    }
    return nil
}
```

## 6. **Middleware - middleware/auth.go**

```go
package middleware

import (
    "context"
    "net/http"
    "strings"
    "time"

    "github.com/golang-jwt/jwt"
)

type contextKey string

const UserContextKey contextKey = "user"

func Authenticate(jwtSecret string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                http.Error(w, `{"error":"missing authorization header"}`, http.StatusUnauthorized)
                return
            }

            tokenString := strings.Replace(authHeader, "Bearer ", "", 1)
            token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
                if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                    return nil, jwt.ErrSignatureInvalid
                }
                return []byte(jwtSecret), nil
            })

            if err != nil || !token.Valid {
                http.Error(w, `{"error":"invalid token"}`, http.StatusUnauthorized)
                return
            }

            claims, ok := token.Claims.(jwt.MapClaims)
            if !ok || !token.Valid {
                http.Error(w, `{"error":"invalid token claims"}`, http.StatusUnauthorized)
                return
            }

            userID := claims["user_id"].(string)
            ctx := context.WithValue(r.Context(), UserContextKey, userID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

func RateLimiter(requests int, window time.Duration) func(http.Handler) http.Handler {
    // Simple in-memory rate limiter
    visitors := make(map[string]*rateLimiter)
    
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := r.RemoteAddr
            limiter, exists := visitors[ip]
            if !exists {
                limiter = &rateLimiter{
                    tokens:    requests,
                    lastReset: time.Now(),
                }
                visitors[ip] = limiter
            }

            if !limiter.Allow() {
                http.Error(w, `{"error":"rate limit exceeded"}`, http.StatusTooManyRequests)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

type rateLimiter struct {
    tokens    int
    lastReset time.Time
}

func (l *rateLimiter) Allow() bool {
    if time.Since(l.lastReset) > time.Minute {
        l.tokens = 100
        l.lastReset = time.Now()
    }

    if l.tokens > 0 {
        l.tokens--
        return true
    }
    return false
}
```

## 7. **WebSocket - handlers/websocket.go (อัปเกรด)**

```go
package handlers

import (
    "encoding/json"
    "log"
    "net/http"
    "sync"
    "time"

    "github.com/gorilla/websocket"
)

var (
    clients   = make(map[*websocket.Conn]bool)
    broadcast = make(chan PaymentUpdate, 100)
    mutex     = &sync.RWMutex{}
)

type PaymentUpdate struct {
    Type    string      `json:"type"`
    Payment interface{} `json:"payment"`
    Time    time.Time   `json:"time"`
}

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        return true // ควรปรับใน production
    },
}

func HandleWebSocket(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("WebSocket upgrade error: %v", err)
        return
    }

    mutex.Lock()
    clients[conn] = true
    mutex.Unlock()

    // Set connection timeout
    conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    conn.SetPongHandler(func(string) error {
        conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    defer func() {
        mutex.Lock()
        delete(clients, conn)
        mutex.Unlock()
        conn.Close()
    }()

    for {
        messageType, p, err := conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseNormalClosure) {
                log.Printf("WebSocket error: %v", err)
            }
            break
        }

        // Echo back for ping/pong
        if messageType == websocket.TextMessage {
            var msg map[string]interface{}
            if err := json.Unmarshal(p, &msg); err == nil {
                if msg["type"] == "ping" {
                    conn.WriteJSON(map[string]string{"type": "pong"})
                }
            }
        }
    }
}

func SendPaymentUpdate(update PaymentUpdate) {
    update.Time = time.Now()
    select {
    case broadcast <- update:
    default:
        log.Println("Broadcast channel full, dropping message")
    }
}

func HandleBroadcast() {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case update := <-broadcast:
            mutex.RLock()
            for client := range clients {
                err := client.SetWriteDeadline(time.Now().Add(10 * time.Second))
                if err != nil {
                    log.Printf("WebSocket write deadline error: %v", err)
                    client.Close()
                    mutex.RUnlock()
                    mutex.Lock()
                    delete(clients, client)
                    mutex.Unlock()
                    mutex.RLock()
                    continue
                }

                err = client.WriteJSON(update)
                if err != nil {
                    log.Printf("WebSocket send error: %v", err)
                    client.Close()
                    mutex.RUnlock()
                    mutex.Lock()
                    delete(clients, client)
                    mutex.Unlock()
                    mutex.RLock()
                }
            }
            mutex.RUnlock()

        case <-ticker.C:
            // Send heartbeat to all clients
            mutex.RLock()
            for client := range clients {
                err := client.WriteJSON(map[string]string{"type": "heartbeat"})
                if err != nil {
                    log.Printf("WebSocket heartbeat error: %v", err)
        
