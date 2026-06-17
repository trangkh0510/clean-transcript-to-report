# clean-transcript-to-report

🌐 **Tiếng Việt** · [English](README.en.md)

### Từ transcript thô → report dùng được ngay. Không phải "dán vào ChatGPT rồi cầu nguyện".

Bạn vừa xong một buổi phỏng vấn người dùng, một cuộc họp, hay một ca hỗ trợ khách hàng. Trên tay là
file ghi âm đầy timestamp, tên người nói lặp lại, từ đệm "ờ à", lỗi nhận dạng giọng nói, câu nói dở
dang. Cách thông thường: dán nguyên khối vào một con chatbot, nhận về một bản tóm tắt chung chung —
trôi tuột, bịa thêm chi tiết, và **không truy được câu nào người dùng thực sự đã nói**.

**Tool này làm khác.** Hai bước, một pipeline, chạy trong một container:

> **1. Clean** — một *rule engine* làm sạch transcript ở hậu trường, có kỷ luật.
> **2. Analyze** — biến transcript sạch thành report HTML có cấu trúc, **mọi kết luận đều có trích dẫn gốc**.

---

## Vì sao đáng dùng (3 điểm khác biệt)

### 🧹 1. Một rule engine 9 tầng làm sạch — không phải "nhờ AI sửa giùm"

Bước Clean không quăng nguyên transcript cho LLM tự xử. Nó chạy qua một **pipeline 9 stage được
thiết kế riêng cho transcript tiếng Việt** (có code-switching tiếng Anh và giọng miền Nam), theo
nguyên tắc xương sống **Precision > Recall — thà bỏ sót một lỗi còn hơn sửa sai một từ đúng**:

| Stage | Engine làm gì |
|---|---|
| **Token Locking** | Khoá cứng số điện thoại, OTP, mã giao dịch, số tiền, năm lịch — **tuyệt đối không bị sửa nhầm** |
| **ASR & Brand** | Sửa lỗi nhận dạng nặng, chuẩn brand ("Tóp Tóp" → TikTok), code-switch ("vau chờ" → voucher) |
| **Giọng miền Nam** | Nhận diện vùng giọng → bật ruleset riêng ("dzậy" → vậy), nhưng **giữ từ địa phương đúng** (tui, ổng, mắc) vì đó là dữ liệu demographic quý cho UX |
| **Số & Tiền tệ** | Phân biệt "năm 2023" với "5 triệu", chuẩn "1 5 triệu" → "một triệu rưỡi", slang "một củ" = 1.000.000đ |
| **Filler & Self-correction** | Bỏ "ờ, à, kiểu như là", gộp câu nói lại ("tôi tôi tôi" → "tôi"), giữ "ạ" lịch sự |
| **Câu, Tên người, QA** | Thêm dấu câu khi chắc chắn, capitalize tên qua whitelist, **flag vùng nghi ngờ để người review** thay vì đoán bừa |

Bạn **bật/tắt từng rule** và thêm rule riêng. Cái gì engine không chắc, nó **gắn cờ chứ không bịa** —
và bạn được xem lại, sửa tay trước khi sang bước phân tích.

### 🧠 2. Logic transcript → report: có cấu trúc, có bằng chứng, không trôi

Bước Analyze **không** trả về một đoạn văn tóm tắt. Nó chạy một pipeline có kiểm soát:

```
gate (chặn cửa) → analyze (LLM trả JSON) → validate (schema) → render (HTML)
```

- **Gate chặn trước khi tốn tiền:** bắt buộc có **objectives** (mục tiêu/câu hỏi nghiên cứu), đúng 1
  file, đủ độ dài tối thiểu — kiểm tra **trước mọi lần gọi LLM** để không đốt quota cho rác.
- **Mọi finding phải có bằng chứng:** mỗi insight / pain point kèm **trích dẫn gốc verbatim ≤15 từ**
  từ chính transcript. Không có chuyện "AI nói vậy" — bạn truy ngược được tới câu người dùng đã nói.
- **Schema ép kỷ luật:** LLM phải trả JSON đúng schema (Pydantic validate). Sai schema → tự thử lại.
  Report HTML do **Python dựng deterministic** từ JSON sạch — không để model tự "vẽ" layout lung tung.

### 🎯 3. May đo cho từng use case — không phải một template cho tất cả

Không phải report nào cũng giống nhau. Mỗi loại có **brain prompt riêng + schema riêng + cách render
riêng**, hỏi đúng câu hỏi mà vai trò đó cần:

| Loại | Dành cho | Report trả về |
|---|---|---|
| **`ux`** | UX researcher | Findings + **pain point có severity (high/med/low)** gắn theo flow, quotes, verdict pass/fail, scorecard, đề xuất |
| **`cs`** | CS / vận hành | Tóm tắt + **đường sentiment khách hàng theo từng phase**, danh sách issue theo mức độ, action item, verdict |
| **`meeting`** | PM / lead / thư ký | **Quyết định (ai chốt)** + action item (việc · người · deadline) + mục off-topic + verdict |

---

## Dùng trong 3 phút

Mở endpoint → wizard một trang, hai bước:

1. **Clean** — upload `.vtt` (Teams) / `.docx` / `.txt` → bật/tắt rule → làm sạch → **xem lại & sửa tay**.
2. **Analyze** — chọn `cs` / `ux` / `meeting` → nhập **objectives** → nhận **report HTML**.

Đã có transcript sạch sẵn? Gọi thẳng API, bỏ qua bước 1:

```bash
curl -s -X POST https://<ENDPOINT>/invocations \
  -H 'Content-Type: application/json' \
  -d '{"type":"meeting","mode":"single","transcripts":[{"participant":"x","text":"..."}],"objectives":"RQ1 ..."}'
```

Chạy local:

```bash
cd src
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # điền LLM_API_KEY / LLM_BASE_URL / LLM_MODEL
python -m app.server            # http://localhost:8080
```

---

## Giới hạn (nói thẳng)

- **Single-file** mỗi lần phân tích (Route A). Multi-file ladder (Route B) đã được gỡ khỏi agent này.
- **Objectives là bắt buộc** cho bước Analyze — không mục tiêu thì gate sẽ chặn (cố ý: report không có
  mục tiêu là report vô dụng).
- Rule engine làm sạch được tinh chỉnh cho **transcript tiếng Việt** (có code-switch + giọng miền Nam).

---

## Tài liệu liên quan

| File | Nội dung |
|---|---|
| `docs/01-agent-README.md` | Kiến trúc, pipeline phân tích, run/deploy chi tiết |
| `docs/02-clean-pipeline-9-rules.md` | **Spec đầy đủ 9 stage của rule engine làm sạch** |
| `docs/prompts/{cs,ux,meeting}.md` | Brain prompt từng loại report |
| `HANDOVER.md` · `DEPLOY.md` | Bàn giao + bảo mật · build/push image + tạo runtime |
