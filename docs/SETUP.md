# Hướng dẫn cài đặt

## Yêu cầu hệ thống

- n8n >= 1.40 (self-hosted hoặc cloud)
- Tài khoản Google Cloud Platform (đã bật Sheets API, Drive API)
- Tài khoản Pinecone (free tier đủ dùng)
- Tài khoản Google AI Studio (Gemini API)
- Tài khoản Groq (free tier)

## Bước 1: Chuẩn bị Google Cloud

### 1.1 Tạo OAuth2 Credentials

1. Truy cập [Google Cloud Console](https://console.cloud.google.com/)
2. Tạo project mới hoặc chọn project có sẵn
3. Bật các API:
   - Google Sheets API
   - Google Drive API
4. Vào **APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client ID**
5. Chọn Application type: **Web application**
6. Thêm Authorized redirect URI: `https://<your-n8n-domain>/rest/oauth2-credential/callback`
7. Download JSON credentials

### 1.2 Chuẩn bị Google Sheets

Tạo 4 Google Sheets với cấu trúc:

**Sheet Saleorder:**
| Cột | Mô tả |
|---|---|
| Mã khách hàng | ID duy nhất |
| Khách hàng | Tên khách hàng |
| Email | Email liên hệ |
| Ngày đặt hàng | Định dạng dd/mm/yyyy |
| Số đơn hàng | Mã đơn hàng |
| Giá trị đơn hàng | Số tiền (VND) |
| Mã hàng hóa | SKU sản phẩm |
| Loại hàng hóa | Phân loại |
| Tỉnh/Thành phố (Giao hàng) | Địa chỉ giao |

**Sheet Account:** Danh sách khách hàng và thông tin liên hệ

**Sheet Product:** Danh mục sản phẩm

**Sheet Merged (output):** Workflow sẽ tự ghi kết quả vào đây, gồm các tab:
- `merge_sheet`: Dữ liệu tổng hợp
- `rfm_sheet`: Kết quả RFM
- `dsgoi_homnay`: Danh sách gọi hôm nay
- `kichbangoi`: Kịch bản gọi

### 1.3 Google Drive

Tạo folder trên Drive để chứa tài liệu nội bộ (policy, sản phẩm, chương trình KM...). Workflow sẽ tự động ingest khi có file mới.

## Bước 2: Cấu hình Pinecone

1. Đăng ký tại [pinecone.io](https://www.pinecone.io/)
2. Tạo index mới:
   - **Index name:** tuỳ chọn (ví dụ: `n8nself`)
   - **Dimensions:** 768 (phù hợp với Gemini embedding)
   - **Metric:** cosine
3. Copy API key từ Dashboard

## Bước 3: API Keys

### Google Gemini
1. Truy cập [Google AI Studio](https://aistudio.google.com/)
2. Tạo API key mới
3. Lưu lại key

### Groq
1. Đăng ký tại [console.groq.com](https://console.groq.com/)
2. Tạo API key
3. Free tier cho phép ~30 req/min với Llama 3.1 8B

## Bước 4: Import Workflow vào n8n

1. Mở n8n instance
2. Vào **Settings → Import from File**
3. Chọn file `data_ana_final.json`
4. Workflow sẽ được import với tất cả nodes

## Bước 5: Cấu hình Credentials trong n8n

Sau khi import, cần tạo lại các credentials:

| Credential | Loại | Ghi chú |
|---|---|---|
| Google Sheets account | Google Sheets OAuth2 | Dùng cho đọc/ghi sheets |
| Google Drive account | Google Drive OAuth2 | Dùng cho document ingestion |
| PineconeApi account | Pinecone API | API key từ bước 2 |
| Google Gemini(PaLM) Api account | Google PaLM API | API key từ bước 3 |
| Google Sheets Trigger account | Google Sheets Trigger OAuth2 | Cho trigger tự động |

Sau khi tạo credentials, gán lại vào từng node tương ứng trong workflow.

## Bước 6: Cập nhật Document/Sheet IDs

Mở từng node Google Sheets trong workflow và chọn lại đúng document + sheet tab từ tài khoản Google đã liên kết.

## Bước 7: Test

1. Upload 1-2 file tài liệu vào Google Drive folder → kiểm tra Pinecone đã index
2. Chạy manual trigger để test flow RFM
3. Kiểm tra sheet output có dữ liệu RFM và kịch bản gọi

## Xử lý lỗi thường gặp

| Lỗi | Nguyên nhân | Giải pháp |
|---|---|---|
| `401 Unauthorized` | Token hết hạn | Reconnect OAuth2 trong n8n |
| `429 Too Many Requests` | Rate limit Groq | Tăng wait time giữa các batch |
| Pinecone timeout | Index chưa ready | Đợi 1-2 phút sau khi tạo index |
| Dữ liệu RFM sai | Định dạng ngày không đúng | Kiểm tra cột "Ngày đặt hàng" đúng format |
