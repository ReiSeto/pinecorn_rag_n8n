# Pinecone RAG + n8n Automation

Hệ thống tự động phân tích khách hàng và hỗ trợ CSKH dựa trên dữ liệu bán hàng thực tế, kết hợp RFM scoring, RAG (Retrieval-Augmented Generation) với Pinecone vector database và AI Agent trên nền tảng n8n.

## Tổng quan

Dự án giải quyết bài toán: **"Làm sao để nhân viên CSKH biết nên gọi ai, nói gì, vào lúc nào?"**

Pipeline xử lý:
1. Thu thập dữ liệu đơn hàng từ nhiều Google Sheets (Saleorder, Account, Product)
2. Tính toán RFM (Recency - Frequency - Monetary) và phát hiện sản phẩm quá hạn mua
3. Phân khúc khách hàng (A/B/C) theo mức độ ưu tiên
4. Lập lịch gọi điện tự động theo ngày làm việc
5. Tạo kịch bản gọi điện cá nhân hóa bằng AI (Gemini + Groq + RAG từ tài liệu nội bộ)
6. Xuất danh sách gọi hàng ngày kèm kịch bản chi tiết

## Kiến trúc hệ thống

```
Google Sheets (3 nguồn)
        │
        ▼
  Merge & Làm sạch dữ liệu
        │
        ▼
  RFM Scoring (Python)
  ├── Recency: ngày kể từ đơn hàng gần nhất
  ├── Frequency: số đơn trong 90 ngày
  └── Monetary: tổng giá trị 90 ngày
        │
        ▼
  Phân khúc A/B/C + Phát hiện quá hạn mua
        │
        ▼
  Lập lịch gọi (30 KH/ngày, bỏ Chủ nhật)
        │
        ▼
  AI Agent (Gemini) ──── RAG (Pinecone)
  ├── Next Best Action                │
  └── Cross-sell / Up-sell           Tài liệu nội bộ
        │
        ▼
  Groq (Llama 3.1) → Kịch bản gọi chi tiết
        │
        ▼
  Google Sheets (output)
```

## Công nghệ sử dụng

| Thành phần | Công nghệ |
|---|---|
| Orchestration | n8n (self-hosted) |
| Data source | Google Sheets API v4 |
| Vector DB | Pinecone |
| Embedding | Google Gemini Embedding |
| LLM chính | Google Gemini |
| LLM phụ (batch) | Groq API (Llama 3.1 8B) |
| RAG | Pinecone Vector Store + Recursive Character Text Splitter |
| Document ingestion | Google Drive → auto-chunking → Pinecone |
| Scheduling | n8n Cron + Google Sheets Trigger |

## Cấu trúc file

```
├── data_ana_final.json    # n8n workflow export (đã ẩn credentials)
├── .env.example           # Template biến môi trường
├── .gitignore
├── docs/
│   ├── SETUP.md           # Hướng dẫn cài đặt chi tiết
│   ├── ARCHITECTURE.md    # Chi tiết kiến trúc và data flow
│   └── RFM_METHODOLOGY.md # Phương pháp tính RFM
└── README.md
```

## Bắt đầu nhanh

1. Clone repo và copy file env:
```bash
git clone https://github.com/ReiSeto/pinecorn_rag_n8n.git
cp .env.example .env
```

2. Điền credentials vào `.env` (xem chi tiết tại [docs/SETUP.md](docs/SETUP.md))

3. Import `data_ana_final.json` vào n8n instance

4. Cấu hình lại các credentials trong n8n theo hướng dẫn

> Chi tiết cài đặt đầy đủ: [docs/SETUP.md](docs/SETUP.md)

## Workflow chính

### Flow 1: Data Pipeline + RFM (Manual trigger)
- Kéo dữ liệu từ 3 sheets → Merge → Tính RFM → Phân khúc → Lập lịch gọi
- AI Agent tạo Next Best Action cho từng khách
- Groq tạo kịch bản gọi chi tiết theo framework Up-sell / Cross-sell / Core-repeat

### Flow 2: Document Ingestion (Drive trigger)
- Khi có file mới trong Google Drive folder → tự động download → chunk → embed → lưu vào Pinecone
- Giúp AI Agent luôn có dữ liệu nội bộ mới nhất để tham khảo

### Flow 3: Script Generation (Sheet trigger)
- Khi có row mới trên sheet danh sách gọi → tự động tạo 3 bước kịch bản gọi điện
- Kịch bản được cá nhân hóa theo dữ liệu khách, thời tiết vùng miền, và chương trình khuyến mãi hiện hành

## Ghi chú bảo mật

- Tất cả credentials đã được thay bằng placeholder `<...>` 
- Không commit file `.env` chứa thông tin thật
- Xem `.env.example` để biết danh sách biến cần cấu hình
