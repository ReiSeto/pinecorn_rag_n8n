# Contributing

## Quy trình phát triển

1. Fork repo
2. Tạo branch mới: `git checkout -b feature/ten-tinh-nang`
3. Commit changes: `git commit -m "Mô tả ngắn gọn"`
4. Push branch: `git push origin feature/ten-tinh-nang`
5. Tạo Pull Request

## Quy ước commit message

```
feat: thêm tính năng mới
fix: sửa lỗi
docs: cập nhật tài liệu
refactor: tái cấu trúc code
chore: công việc maintenance
```

## Lưu ý khi chỉnh sửa workflow

- Export workflow từ n8n dưới dạng JSON
- Thay thế tất cả credentials thật bằng placeholder `<...>` trước khi commit
- Không commit file `.env`
- Test workflow trên môi trường local trước khi push

## Cấu trúc code trong n8n nodes

### Python nodes (RFM)
- Viết code rõ ràng, có comment tiếng Việt
- Handle edge cases (NaN, empty data, wrong date format)
- Sử dụng pandas cho data processing

### JavaScript nodes
- Dùng ES6+ syntax
- Handle null/undefined gracefully
- Log errors thay vì crash silent
