# Changelog

## [1.0.0] - 2025-05-26

### Thêm mới
- Pipeline phân tích RFM hoàn chỉnh (Recency, Frequency, Monetary)
- Phát hiện sản phẩm quá hạn mua theo chu kỳ mua từng khách
- Phân khúc khách hàng A/B/C tự động
- Lập lịch gọi điện tự động (30 KH/ngày, bỏ CN)
- AI Agent (Gemini) tạo Next Best Action từ RAG
- Groq batch processing cho kịch bản framework
- Auto-ingest tài liệu từ Google Drive vào Pinecone
- Script generation trigger khi có row mới
- Thời tiết theo vùng miền inject vào prompt

### Bảo mật
- Ẩn tất cả credentials, document IDs, folder IDs
- Thêm `.env.example` làm template
- Thêm `.gitignore` cho file nhạy cảm

### Tài liệu
- README.md với kiến trúc tổng quan
- docs/SETUP.md hướng dẫn cài đặt
- docs/ARCHITECTURE.md chi tiết data flow
- docs/RFM_METHODOLOGY.md phương pháp phân tích
- CONTRIBUTING.md quy trình đóng góp
