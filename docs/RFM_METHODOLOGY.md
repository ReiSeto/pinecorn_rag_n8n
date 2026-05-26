# Phương pháp phân tích RFM

## RFM là gì?

RFM (Recency - Frequency - Monetary) là phương pháp phân khúc khách hàng dựa trên hành vi mua hàng thực tế. Phương pháp này hiệu quả trong B2B distribution vì chu kỳ mua hàng có tính lặp lại cao.

## Các chỉ số

### Recency (R)
- **Định nghĩa:** Số ngày kể từ đơn hàng gần nhất
- **Xử lý:** `recency = (today - max(order_date)).days`, clip >= 0

### Frequency (F)  
- **Định nghĩa:** Số lần mua trong 90 ngày gần nhất
- **Cửa sổ:** 90 ngày (phù hợp chu kỳ mua B2B)

### Monetary (M)
- **Định nghĩa:** Tổng giá trị đơn hàng trong 90 ngày

## Tính điểm

### Percentile Ranking
```python
freq_rank = frequency.rank(pct=True)      # 0.0 - 1.0
monetary_rank = monetary.rank(pct=True)    # 0.0 - 1.0
```

### Recency Score
```python
recency_score = pd.qcut(recency, 5, labels=[5, 4, 3, 2, 1])
```

### RFM Score tổng hợp
```python
RFM_score = 5 * freq_rank + 3 * monetary_rank + 2 * recency_score
```
Trọng số: Frequency (5) > Monetary (3) > Recency (2).

## Phân khúc

| Segment | Tiêu chí | Ưu tiên |
|---|---|---|
| **A** | R <= 30 hoặc M_rank >= 0.8 hoặc F >= 3 | Cao nhất |
| **B** | 31 <= R <= 60 hoặc 0.2 <= M_rank < 0.5 | Trung bình |
| **C** | Còn lại | Thấp |

## Purchase Cycle Warning

Phát hiện sớm khi khách "trễ hẹn" mua lại sản phẩm cụ thể:

1. Nhóm theo `(customer_id, sku_key)`
2. Tính `deltas = order_date.diff().days`
3. Chu kỳ mua = `median(deltas)`
4. Ngưỡng: `warn_due = last_purchase + 0.2 * purchase_cycle`
5. Nếu `today > warn_due` -> Warning

Hệ số 0.2 = cảnh báo sớm khi mới quá 20% chu kỳ.

## Lập lịch gọi

- 30 khách/ngày/nhân viên
- Bỏ Chủ nhật, timezone UTC+7
- Sắp xếp: số SP quá hạn > Segment > RFM score
