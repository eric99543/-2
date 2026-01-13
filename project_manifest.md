Global Context (全局背景):
硬體環境 (13900K 192.168.2.3/2696v4 192.168.2.4)
Tech Stack Constraints (技術棧約束): 1. 核心連接庫選擇 Rust 生態（性能首選） toml # Cargo.toml 關鍵依賴 [dependencies] tokio = { version = "1.37", features = ["full"] } tungstenite = "0.20" # WebSocket 客戶端 reqwest = { version = "0.11", features = ["rustls-tls"] } serde = { version = "1.0", features = ["derive"] } serde_json = "1.0" tokio-util = { version = "0.7", features = ["codec"] } tracing = "0.1" # 高性能日誌 tokio-tracing = "0.3" rdkafka = "0.36" # Kafka 生產者 Go 生態（折衷選擇） go // go.mod 關鍵依賴 require ( github.com/gorilla/websocket v1.5.1 github.com/go-resty/resty/v2 v2.11.0 github.com/confluentinc/confluent-kafka-go v1.9.2 github.com/json-iterator/go v1.1.12 github.com/prometheus/client_golang v1.18.0 ) Python 生態（快速原型） python # requirements.txt websockets==12.0 aiohttp==3.9.1 aiokafka==0.8.0 uvloop==0.19.0 # Linux專用，性能接近Rust orjson==3.9.10 # 最快的JSON解析 msgpack==1.0.7 2. 連接架構模式 模式一：每交易對獨立連接 text 優點：隔離故障，獨立重連 缺點：連接數多，管理複雜 適用：核心交易對（BTC/ETH） 模式二：聚合連接 text 優點：管理簡單，資源節省 缺點：單點故障，延遲不均 適用：長尾交易對 推薦方案：混合模式 rust // Rust 示例架構 struct ExchangeConnector { // 核心交易對：獨立連接 major_pairs: HashMap<String, IndividualWebSocket>, // 長尾交易對：聚合連接 minor_pairs: AggregatedWebSocket, // 故障轉移連接 backup_connections: Vec<WebSocketBackup>, } 3. 數據接收優化 協議層優化 WebSocket壓縮：啟用 permessage-deflate 壓縮 二進制協議：優先使用交易所二進制格式（如幣安Futures） 增量更新：使用增量depth book，非全量 心跳優化：自定義心跳間隔，非固定時間 網路層優化 TCP_NODELAY：禁用Nagle算法 SO_REUSEPORT：多進程綁定同一端口 連接池：HTTP/1.1 keep-alive + 連接復用 DNS緩存：本地DNS緩存，避免解析延遲 🧹 數據清洗層（13900K） 1. 標準化庫選擇 Rust 實現（推薦） toml [dependencies] prost = "0.12" # Protobuf 編解碼 prost-types = "0.12" bytes = "1.5" chrono = { version = "0.4", features = ["serde"] } uuid = { version = "1.6", features = ["v4", "serde"] } 數據標準化流程 rust // 統一的數據結構 #[derive(Serialize, Deserialize, Clone)] struct UnifiedTrade { pub id: String, // UUID v4 pub exchange: Exchange, // 枚舉 pub symbol: String, // 標準化格式 pub price: Decimal, // 定點小數 pub quantity: Decimal, pub side: Side, // BUY/SELL pub timestamp_ns: i64, // Unix納秒 pub received_ns: i64, // 本地接收時間 pub raw_data: Vec<u8>, // 原始數據（Protobuf） } // 清洗管道 async fn cleaning_pipeline(raw: RawData) -> UnifiedTrade { // 1. 驗證數據完整性 // 2. 標準化符號格式 // 3. 轉換價格精度 // 4. 生成唯一ID // 5. 添加接收時間戳 // 6. 序列化原始數據 } 2. 異常檢測與處理 實時檢測規則 價格異常：偏離市場平均價 ±5% 數量異常：超過日平均量的 10x 時間戳異常：未來時間或過期數據 序列號異常：trade_id 不連續 處理策略 rust enum DataQuality { Perfect, // 直接使用 Repaired, // 修復後使用 Discarded, // 丟棄但記錄 Suspicious, // 標記可疑，人工審核 } 🚀 數據傳輸層（13900K → 2696v4） 1. 消息隊列選擇 方案A：Apache Kafka（推薦） toml [dependencies] rdkafka = { version = "0.36", features = ["cmake-build"] } 配置要點： ini # producer.properties compression.type=zstd linger.ms=0 # 立即發送 batch.size=65536 acks=1 # 平衡可靠性和延遲 方案B：NATS（更低延遲） toml [dependencies] async-nats = "0.32" 方案C：ZeroMQ（極致性能） toml [dependencies] zmq = "0.10" 2. 傳輸協議設計 Protobuf 消息定義 protobuf syntax = "proto3"; package market_data; message MarketDataBatch { uint64 batch_id = 1; repeated UnifiedTrade trades = 2; repeated UnifiedOrderBook orderbooks = 3; uint64 send_timestamp_ns = 4; } // 主題設計 // trades.binance.spot.btcusdt // orderbook.okx.swap.ethusdt // funding.bybit.perp.solusdt 壓縮策略 實時數據：不壓縮或Zstd Level 1 批量數據：Zstd Level 3 歸檔數據：Zstd Level 10 + 字典訓練 💾 數據存儲層（2696v4） 1. 存儲引擎選型矩陣 數據類型 主要存儲 輔助存儲 保留策略 逐筆交易 TimescaleDB Parquet (S3) 原始：30天，聚合：永久 深度數據 Redis + TimescaleDB ClickHouse L2：7天，快照：永久 K線數據 PostgreSQL DuckDB 1min以上：永久 資金費率 PostgreSQL CSV歸檔 全部永久 爆倉數據 ClickHouse S3 全部永久 2. 數據庫配置要點 TimescaleDB 優化 sql -- 創建超表 CREATE TABLE trades ( time TIMESTAMPTZ NOT NULL, exchange_id SMALLINT, symbol_id INTEGER, price NUMERIC(24,8), volume NUMERIC(24,8), side CHAR(1) ) USING columnar; -- 分區策略 SELECT create_hypertable('trades', 'time', chunk_time_interval => INTERVAL '1 day', create_default_indexes => FALSE ); -- 自定義索引 CREATE INDEX idx_trades_composite ON trades (symbol_id, time DESC) INCLUDE (price, volume, side); ClickHouse 配置 xml <!-- config.xml --> <merge_tree> <max_partitions_per_insert_block>1000</max_partitions_per_insert_block> <min_bytes_for_wide_part>1073741824</min_bytes_for_wide_part> </merge_tree> <compression> <case> <min_part_size>1073741824</min_part_size> <method>zstd</method> <level>1</level> </case> </compression> Redis 作為緩存層 python # 緩存策略 CACHE_STRATEGY = { 'ticker': {'ttl': 1, 'max_size': 10000}, 'orderbook_l1': {'ttl': 0.1, 'max_size': 1000}, 'orderbook_l2': {'ttl': 0.5, 'max_size': 5000}, } 3. 分片與備份策略 時間分片 text /data/timescaledb/ ├── 2024/ │ ├── Q1/ │ │ ├── trades_202401.parquet │ │ └── trades_202402.parquet │ └── Q2/ 交易所分片 text /data/by_exchange/ ├── binance/ ├── okx/ ├── bybit/ ⚡ 計算層（2696v4 - 回測與分析） 1. 計算引擎選擇 實時計算 rust // Rust + SIMD 向量化計算 use packed_simd::f64x4; struct VectorizedCalculator { // SIMD優化的指標計算 fn calculate_sma(&self, prices: &[f64], window: usize) -> Vec<f64> { // SIMD實現 } } 批量計算 python # Python 但使用高效庫 import polars as pl # 替代pandas，性能更好 import duckdb # 內存OLAP import numba # JIT編譯 # 配置Polars pl.Config.set_streaming_chunk_size(100000) pl.Config.set_fmt_str_lengths(100) 2. 回測框架架構 python class BacktestEngine: def __init__(self): self.data_source = UnifiedDataSource() # 統一的數據接口 self.strategy_loader = StrategyLoader() self.performance = MetricsCalculator() async def run(self, strategy_id: str, start: datetime, end: datetime): # 從統一數據源讀取 data = await self.data_source.load( symbols=["BTC-USDT-PERP"], start=start, end=end, data_types=["trade", "orderbook", "funding"] ) # 執行策略 results = await self.strategy_loader.run(data) # 計算指標 metrics = self.performance.calculate(results) return metrics 📊 可視化層（2680v2） 1. 技術棧選擇 後端 API 服務 python # FastAPI + WebSocket from fastapi import FastAPI, WebSocket import psycopg2 import aioredis app = FastAPI() @app.websocket("/ws/market") async def market_websocket(websocket: WebSocket): # 實時市場數據推送 pass @app.get("/api/analysis") async def get_analysis(): # 分析結果API pass 前端可視化 javascript // React + TypeScript + 高效圖表 import { useWebSocket } from './hooks/useWebSocket' import { RealTimeChart } from './components/RealTimeChart' import { OrderBookVisualizer } from './components/OrderBookVisualizer' // 關鍵庫 // - recharts: 通用圖表 // - uPlot: 超高性能時間序列圖 // - D3.js: 自定義可視化 // - WebGL: 大規模數據渲染 2. 數據流架構 text TimescaleDB/ClickHouse → Materialized Views → Redis Cache → WebSocket Server → Browser WebSocket → React State → Visualization 🔧 監控與運維 1. 系統監控 Prometheus：指標收集 Grafana：儀表板 Loki：日誌聚合 AlertManager：告警管理 2. 關鍵指標 yaml latency: websocket_receive_ms: <10ms processing_ms: <5ms storage_write_ms: <20ms end_to_end_ms: <50ms throughput: trades_per_second: >100,000 messages_per_second: >1,000,000 storage_iops: >50,000 reliability: uptime: >99.99% data_loss: <0.001% recovery_time: <30s 🚀 實施路線圖 第1階段：基礎設施（2週） 部署TimescaleDB + ClickHouse 搭建Kafka集群 配置監控系統 建立開發環境 第2階段：核心採集（3週） 實現幣安現貨WebSocket 實現數據標準化管道 建立基礎存儲 簡單可視化 第3階段：擴展與優化（4週） 添加OKX、Bybit等交易所 實現合約數據採集 優化性能瓶頸 添加異常檢測 第4階段：高級功能（持續） 對敲檢測算法 回測框架 機器學習模型 自動交易接口 💡 關鍵決策點 必須自研的核心組件 數據標準化格式：你的"通用語言" 交易所協議適配器：避免抽象層開銷 實時計算引擎：SIMD優化的指標計算 統一數據接口：所有上層應用共用 可以使用現成方案的組件 數據庫：TimescaleDB, ClickHouse 消息隊列：Kafka 監控系統：Prometheus + Grafana 前端框架：React + FastAPI 性能關鍵路徑優化順序 網路延遲：交易所→採集器（最關鍵） 序列化開銷：二進制 > JSON 存儲IO：批量寫入 + 適當索引 計算瓶頸：向量化 + 並行化 📦 最終推薦技術棧 text 語言層： - 數據採集：Rust（性能）+ Go（快速開發交易所SDK） - 數據處理：Rust（核心管道）+ Python（數據科學） - 後端API：Python FastAPI（開發效率） - 前端：TypeScript + React 存儲層： - 主存儲：TimescaleDB（交易數據） - 分析存儲：ClickHouse（聚合查詢） - 緩存：Redis（實時數據） - 歸檔：MinIO + Parquet 消息層： - 主消息隊列：Apache Kafka（可靠） - 實時推送：NATS（低延遲） 部署層： - 容器化：Docker - 編排：Docker Compose（單機）或 Kubernetes（未來） - 配置管理：Ansible 🎯 給Claude Code的提示模板 當你開始構建時，可以這樣提示： text "請為我創建一個Rust項目，用於採集幣安現貨WebSocket數據。 要求： 1. 使用tokio和tungstenite 2. 實現BTC/USDT交易對的實時交易數據採集 3. 數據結構包含：價格、數量、方向、時間戳 4. 將數據發送到本地的Kafka topics 'trades.binance.spot.btcusdt' 5. 包含錯誤處理和重連機制 6. 添加性能指標輸出 這個技術棧選擇能讓你在性能、開發效率和可維護性之間取得最佳平衡，並且完全可控，沒有第三方框架的抽象層開銷。
這是全局約束
「這是一個長期演進的量化市場數據平台
所有組件必須模組化、可替換、不可耦合
不允許為了方便引入黑箱框架
不做任何一次性 Demo 設計」

Architecture Diagram (架構圖):核心架構與技術選型指南 總體設計哲學：分層模塊化 text 原始數據 → 採集層 → 標準化層 → 存儲層 → 計算層 → 可視化層 🔌 數據採集層（13900K - 極致低延遲）
Phase Roadmap (階段路線圖):
phase1
目標機器：13900K（只考慮本機，不涉及分佈式）
你要做的事情：
1. 設計一套「交易所無關」的統一市場數據模型
2. 這套模型只負責「語義與結構」，不負責採集、不負責存儲

數據範圍（僅限）：
- trade
- orderbook（L1 / L2 可區分）
- funding
- liquidation

強制約束：
- 不允許硬編碼任何交易所（Binance / OKX 等只能以 enum 或 tag 表示）
- 不允許假設某一交易所一定有某欄位
- 不允許直接使用 JSON 作為內部表示
- 必須支持二進制序列化（例如 Protobuf / FlatBuffers 類型）

演進要求：
- 必須有版本號
- 新版本不得破壞舊版本反序列化
- 允許未來新增交易所私有欄位（extension / metadata）

邊界限制（非常重要）：
- 不要連接任何交易所
- 不要寫 Kafka
- 不要設計資料庫 schema
- 不要設計 API

交付結果：
- 一個清晰的數據模型設計
- 模型之間的關係說明
- 為什麼這樣設計可以覆蓋主流幣圈交易所
Phase 2 —— 數據清洗與標準化管道
執行機器：13900K
👉 直接貼給 Claude Code
目標機器：13900K（單機，本地）
前置條件：
- 已存在一套「交易所無關」的統一市場數據模型
- 本階段只能使用該模型，不允許重新定義模型
你要做的事情：
1. 基於統一數據模型，設計一個「數據清洗與標準化管道」
2. 管道的輸入是「原始市場數據（已反序列化，但未清洗）」
3. 管道的輸出是「可直接下游使用的統一數據實例」
清洗流程必須包含（順序不可亂）：
- 結構完整性驗證
- 欄位合法性驗證（價格、數量、時間）
- 符號標準化（例如 BTCUSDT → BTC/USDT）
- 精度修正（禁止浮點誤差）
- 本地接收時間戳補齊
- 異常標記（非直接丟棄）
異常處理策略：
- 區分：可修復 / 不可修復 / 可疑
- 不允許在此階段直接丟棄所有異常數據
強制約束：
- 不允許連接任何交易所
- 不允許 Kafka / MQ
- 不允許資料庫
- 不允許網路 IO
交付結果：
- 一個清晰的清洗管道設計
- 每一步清洗的責任說明
- 為什麼這個設計適合高頻市場數據
Phase 3 —— 消息傳輸層（Producer）
執行機器：13900K
目標機器：13900K
前置條件：
- 已完成統一數據模型
- 已完成清洗與標準化管道
你要做的事情：
1. 設計一個「市場數據消息生產模組」
2. 該模組只負責：將清洗後的統一數據發送到 Kafka
消息設計要求：
- 使用二進制序列化
- 必須攜帶模型版本資訊
- 不允許在消息層重新修改數據語義
Topic 設計規則：
- 類型.交易所.市場.標的
- 例如：trade.binance.spot.btcusdt
性能與可靠性約束：
- 低延遲優先
- 允許 at-least-once
- 不在此層處理回補或重放
嚴格禁止：
- 不允許 Consumer
- 不允許存儲
- 不允許業務邏輯
- 不允許數據聚合
交付結果：
- Kafka Producer 設計
- Topic 命名規範
- 為什麼這樣不會成為瓶頸
Phase 4 —— 第一個交易所適配器
執行機器：13900K
目標機器：13900K
前置條件：
- 統一數據模型已確定
- 清洗管道存在
- Kafka Producer 存在
你要做的事情：
1. 實作「第一個交易所的數據接入適配器」
2. 僅限一個交易所（幣安）
3. 僅限一個市場（現貨）
4. 僅限一個數據類型（trade）
5. 僅限一個交易對（BTC/USDT）
數據流向（不可改）：
交易所 WebSocket
→ 原始數據解析
→ 統一模型映射
→ 清洗管道
→ Kafka Producer
強制約束：
- 不允許支持第二個交易所
- 不允許設計通用 SDK
- 不允許「為未來擴展」提前抽象
交付結果：
- 一個乾淨、單一職責的交易所適配器
- 為什麼這個適配器未來可以被複製而不是修改
Phase 5 —— 消息消費與存儲寫入
執行機器：2696v4
目標機器：2696v4（雙路，多核心）
前置條件：
- Kafka 中已存在統一市場數據
你要做的事情：
1. 設計 Kafka Consumer
2. 將數據寫入 TimescaleDB / ClickHouse
3. 僅實現「寫入」
存儲原則：
- 不做即時查詢
- 不做聚合
- 不做分析
- 不優化索引（先保證吞吐）
強制約束：
- 不允許回推數據到上游
- 不允許業務邏輯
- 不允許 API
交付結果：
- 消費與存儲寫入流程
- 為什麼這樣寫不會阻塞 Kafka
Phase 6 —— 統一數據訪問語義層
執行機器：2696v4
目標機器：2696v4
你要做的事情：
1. 設計一個「統一數據訪問接口」
2. 對上層屏蔽 TimescaleDB / ClickHouse 差異
3. 查詢條件只允許：時間 / 標的 / 類型
強制約束：
- 不暴露 SQL
- 不暴露具體資料庫
- 不做計算
交付結果：
- 為回測 / API / 可視化準備的數據語義層
Phase 7 —— 計算與回測框架
執行機器：2696v4
目標機器：2696v4
你要做的事情：
1. 基於統一數據訪問接口
2. 設計批量分析與回測框架
3. 偏向列式、向量化、批處理
強制約束：
- 不處理實時
- 不接 Web
- 不依賴前端
交付結果：
- 一個純計算層設計
Phase 8 —— API / Dashboard
執行機器：2696v4（或未來獨立）
目標機器：2696v4
你要做的事情：
1. 基於既有數據訪問接口
2. 提供 API 與 WebSocket
3. 前端只顯示，不做決策
強制約束：
- 不允許業務邏輯進前端
- 不允許繞過數據語義層
交付結果：
- 一個完全被動的可視化系統
