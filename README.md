# clean-transcript-to-report — Use Case

🌐 **Tiếng Việt** · [English](README.en.md)

Một web app (một container) biến **transcript họp / phỏng vấn thô** thành **report HTML có cấu trúc**.
Hai bước nằm trong cùng một pipeline, dùng chung 1 endpoint LLM:

1. **Clean** — làm sạch transcript thô (`.vtt` Teams / `.docx` / `.txt`).
2. **Analyze** — biến transcript đã sạch thành report theo loại `cs` / `ux` / `meeting`.

> Tài liệu này tập trung mô tả **dùng để làm gì** và **dùng như thế nào**.
> Kiến trúc, pipeline kỹ thuật và cách deploy xem `docs/01-agent-README.md` và `DEPLOY.md`.

---

## 1. Vấn đề cần giải quyết

Sau mỗi buổi họp, phỏng vấn người dùng (UX research) hay buổi hỗ trợ khách hàng (CS), ta thường
có một file transcript thô — đầy timestamp, tên người nói lặp lại, từ đệm ("ờ", "à"), câu nói dở
dang, nhiễu của công cụ ghi âm. Đọc tay để rút ra insight rất tốn thời gian và không nhất quán.

Agent này tự động hoá hai việc tốn công nhất:

- **Làm sạch** transcript về dạng đọc được, giữ đúng ý người nói (ưu tiên *Precision > Recall*).
- **Phân tích** transcript đã sạch thành report có cấu trúc, bám sát **mục tiêu nghiên cứu
  (objectives)** mà người dùng đặt ra.

---

## 2. Ai dùng & dùng khi nào

| Vai trò | Tình huống | Loại report |
|---|---|---|
| **UX researcher** | Sau phỏng vấn người dùng, cần tổng hợp pain point / insight bám theo research question | `ux` |
| **CS / vận hành** | Sau cuộc hỗ trợ khách hàng, cần tóm tắt vấn đề + hướng xử lý | `cs` |
| **PM / lead / thư ký** | Sau cuộc họp, cần biên bản: quyết định, action item, người phụ trách | `meeting` |

Điểm chung: có một file ghi âm/transcript và cần một report sạch, có cấu trúc, **nhanh**.

---

## 3. Luồng sử dụng (wizard 2 bước)

Mở endpoint → giao diện wizard một trang, hai bước:

**Bước 1 — Clean**
1. Upload `.vtt` / `.docx` / `.txt` (parse ngay trong trình duyệt).
2. Bật/tắt 9 rule làm sạch (+ thêm custom rule nếu cần).
3. Bấm làm sạch → LLM xử lý qua proxy server (`POST /api/chat`).
4. **Xem lại / chỉnh tay** transcript đã sạch trước khi sang bước 2.

**Bước 2 — Analyze**
1. Chọn loại report: `cs` / `ux` / `meeting`.
2. Nhập **objectives** (mục tiêu / câu hỏi nghiên cứu) — bắt buộc.
3. Bấm phân tích → nhận **report HTML** dựng từ JSON đã validate.

> Mỗi lần phân tích nhận **đúng 1 transcript đã sạch**. Có điều kiện tối thiểu về độ dài
> (ux ≥ 500 từ · cs ≥ 300 từ · meeting ≥ 500 từ) — kiểm tra **trước** khi gọi LLM để tránh tốn quota.

---

## 4. Đầu vào / đầu ra

| | Nội dung |
|---|---|
| **Input** | 1 file transcript (`.vtt` / `.docx` / `.txt`) + loại report + objectives |
| **Trung gian** | Transcript đã làm sạch (người dùng review/sửa được) |
| **Output** | Report HTML có cấu trúc theo loại (`cs` / `ux` / `meeting`) |

---

## 5. Hai cách gọi

- **UI (khuyến nghị):** mở `https://<ENDPOINT>/` → dùng wizard Bước 1 → Bước 2.
- **API trực tiếp (bỏ qua Clean):** nếu đã có transcript sạch, gọi thẳng:

```bash
curl -s -X POST https://<ENDPOINT>/invocations \
  -H 'Content-Type: application/json' \
  -d '{"type":"meeting","mode":"single","transcripts":[{"participant":"x","text":"..."}],"objectives":"RQ1 ..."}'
```

---

## 6. Giới hạn (phạm vi hiện tại)

- Chỉ **single-file** (Route A). Toàn bộ Route B (multi-file ladder) đã được gỡ khỏi agent này.
- **Objectives là bắt buộc** cho bước Analyze — không có thì gate sẽ chặn.

---

## 7. Chạy nhanh local

```bash
cd src
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # điền LLM_API_KEY / LLM_BASE_URL / LLM_MODEL thật
python -m app.server            # http://localhost:8080
```

---

## 8. Tài liệu liên quan

| File | Nội dung |
|---|---|
| `docs/01-agent-README.md` | Kiến trúc, pipeline phân tích, run/deploy chi tiết |
| `docs/02-clean-pipeline-9-rules.md` | Spec 9 rule làm sạch (đứng sau Bước 1) |
| `docs/prompts/{cs,ux,meeting}.md` | Brain prompt từng loại report |
| `HANDOVER.md` | Bàn giao + bảo mật |
| `DEPLOY.md` | Build/push image + tạo runtime GreenNode AgentBase |
