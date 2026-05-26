# Kiến trúc hệ thống

## Tổng quan pipeline

Hệ thống gồm 3 workflow chính chạy trên n8n, phối hợp với nhau qua Google Sheets làm trung gian:

```
┌─────────────────────────────────────────────────────────┐
│                    WORKFLOW 1: DATA PIPELINE             │
│                                                         │
│  [Manual Trigger]                                       │
│       │                                                 │
│       ├──► [Sheet: Saleorder] ──┐                       │
│       ├──► [Sheet: Account]  ───┼──► [Merge] ──► [Code] │
│       └──► [Sheet: Product]  ──┘         │              │
│                                          ▼              │
│                                   [Tổng hợp → Sheet]    │
│                                          │              │
│                                          ▼              │
│                                   [RFM (Python)]        │
│                                          │              │
│                                          ▼              │
│                                   [Sort + Assign]       │
│                                          │              │
│                                          ▼              │
│                                   [Sheet: RFM output]   │
│                                          │              │
│                                          ▼              │
│                                   [Filter: hôm nay]     │
│                                       │     │           │
│                                       ▼     ▼           │
│                              [Tạo prompt] [Merge1]      │
│                                  │              │       │
│                                  ▼              │       │
│                           [AI Agent + RAG]      │       │
│                                  │              │       │
│                                  ▼              │       │
│                              [Merge1] ◄─────────┘       │
│                                  │                      │
│                           ┌──────┴──────┐               │
│                           ▼             ▼               │
│                    [Sheet output]  [Groq batch]          │
│                                        │                │
│                                        ▼                │
│                                 [Merge2 → Sheet]        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                 WORKFLOW 2: DOCUMENT INGESTION            │
│                                                         │
│  [Google Drive Trigger: new file]                       │
│       │                                                 │
│       ▼                                                 │
│  [Search files in folder]                               │
│       │                                                 │
│       ▼                                                 │
│  [Download file]                                        │
│       │                                                 │
│       ▼                                                 │
│  [Loop: for each file]                                  │
│       │                                                 │
│       ▼                                                 │
│  [Default Data Loader (binary)]                         │
│       │                                                 │
│       ▼                                                 │
│  [Recursive Text Splitter]                              │
│       │                                                 │
│       ▼                                                 │
│  [Gemini Embedding]                                     │
│       │                                                 │
│       ▼                                                 │
│  [Pinecone Vector Store: insert]                        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│               WORKFLOW 3: SCRIPT GENERATION              │
│                                                         │
│  [Google Sheets Trigger: new row in dsgoi_homnay]       │
│       │                                                 │
│       ▼                                                 │
│  [Get all rows]                                         │
│       │                                                 │
│       ├──► [Loop: AI Agent 2 + RAG]                     │
│       │         │                                       │
│       │         ▼                                       │
│       │    [3 bước kịch bản gọi]                        │
│       │         │                                       │
│       └──► [Merge3]                                     │
│              │                                          │
│              ▼                                          │
│         [Code: extract fields]                          │
│              │                                          │
│              ▼                                          │
│         [Sheet: kichbangoi]                             │
└─────────────────────────────────────────────────────────┘
```

## Chi tiết xử lý RFM

### Input
- Dữ liệu đơn hàng từ 3 nguồn (Google Sheets), merge theo `customer_id`

### Tính toán (Python node)

**Recency:** Số ngày từ đơn hàng gần nhất đến hiện tại
```
recency = (today - last_order_date).days
```

**Frequency:** Số đơn hàng trong 90 ngày gần nhất
```
frequency = count(orders where order_date >= today - 90)
```

**Monetary:** Tổng giá trị đơn hàng trong 90 ngày
```
monetary = sum(order_value where order_date >= today - 90)
```

**RFM Score:** Trọng số kết hợp
```
RFM_score = 5 * freq_rank + 3 * monetary_rank + 2 * recency_score
```
- `freq_rank`, `monetary_rank`: percentile rank (0-1)
- `recency_score`: qcut 5 nhóm (1-5, recency thấp = score cao)

### Phân khúc

| Segment | Điều kiện |
|---|---|
| A (VIP) | recency ≤ 30 HOẶC monetary_rank ≥ 0.8 HOẶC frequency ≥ 3 |
| B (Tiềm năng) | 31 ≤ recency ≤ 60 HOẶC 0.2 ≤ monetary_rank < 0.5 |
| C (Cần chăm sóc) | Còn lại |

### Phát hiện quá hạn mua (Purchase Cycle Warning)

Với mỗi cặp (customer, SKU):
1. Tính median khoảng cách giữa các lần mua → `purchase_cycle`
2. Ngưỡng cảnh báo: `warn_due = last_purchase + 0.2 * purchase_cycle`
3. Nếu `today > warn_due` → đánh dấu warning

## AI Agent System

### Layer 1: Next Best Action (Gemini + RAG)
- Input: dữ liệu RFM + thời tiết vùng miền + sản phẩm quá hạn
- RAG: truy vấn tài liệu nội bộ từ Pinecone (chương trình KM, catalog sản phẩm)
- Output: gợi ý hành động (cross-sell, up-sell, core-repeat)

### Layer 2: Kịch bản framework (Groq / Llama 3.1)
- Input: NBA output + dữ liệu KH
- Áp dụng framework: Up-sell / Cross-sell / Core-Product-Repeat
- Output: prompt chi tiết cho Layer 3

### Layer 3: Kịch bản gọi điện (Gemini + RAG)
- Input: dữ liệu tổng hợp từ Layer 1 + 2
- Tạo 3 bước kịch bản cụ thể, tự nhiên
- Output: kịch bản sẵn sàng cho nhân viên CSKH

## Rate Limiting

- Groq API: batch 5 requests, chờ 2s giữa mỗi batch
- AI Agent (Gemini): loop 1 item/lần, chờ 30s giữa mỗi vòng
- Script Generation: loop 1 item/lần, chờ 30s

## Luồng dữ liệu thời tiết

Hệ thống có sẵn bảng pattern thời tiết theo tháng cho 3 miền (Bắc/Trung/Nam), gồm 63 tỉnh thành. Thông tin thời tiết được inject vào prompt để AI gợi ý sản phẩm phù hợp theo mùa.
