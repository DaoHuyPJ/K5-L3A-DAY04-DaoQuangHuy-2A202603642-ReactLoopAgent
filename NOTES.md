# Ghi chú phân tích · Day 04

Họ và tên: Đào Quang Huy
MSSV: 2A202603642

Cấu hình cả hai lần chạy (giống hệt nhau, xem khối `config`): agent `openai / gpt-4o-mini`, judge `openai / gpt-4o-mini`, `max_steps = 8`, dataset `core` (22 case), snapshot mặc định.

## 1. Case được chọn

- ID case: **H1** (cùng lỗi xuất hiện ở **H6**)
- Câu hỏi H1: điều tra incident GitHub `h02n9572mx9y`, *theo liên kết do incident cung cấp* để tìm nguyên nhân và ảnh hưởng, rồi đối chiếu với incident Cloudflare `yaa0witbl2me`, dẫn nguồn từng bên.
- Câu trả lời của agent (lần 1): liệt kê thông tin của `h02n9572mx9y` và `yaa0witbl2me`. Ở update cuối của GitHub, agent chỉ chép lại câu *"Please refer to the combined summary in this related incident"* mà **không đọc incident liên quan** `xr3myzj7ms51`. Vì vậy câu trả lời không có nguyên nhân hay ảnh hưởng thực tế của sự cố GitHub, và cũng không có nguồn cho `xr3myzj7ms51`.
- Điểm ở lần chạy 1: **H1 = 60** (judge 70, bị trần 60 do kiểm tra bằng code), **H6 = 60**.

Lý do chọn: đây là lỗi được **kiểm tra bằng code** phát hiện (`grade.mechanical.violations`), không chỉ dựa vào nhận xét của judge, và lặp lại ở hai case khác nhau.

## 2. Bằng chứng trong trace

Trace H1 trong `eval_before.json` (rút gọn):

```text
{'event': 'model', 'step': 1, 'finish': 'tool_calls', 'tool_calls': ['get_incident_details', 'get_incident_details']}
TOOL step 1 get_incident_details {'provider': 'github', 'incident_id': 'h02n9572mx9y'}
   -> updates[0].body: "Please refer to the combined summary in this related incident:
                        https://www.githubstatus.com/incidents/xr3myzj7ms51"
TOOL step 1 get_incident_details {'provider': 'cloudflare', 'incident_id': 'yaa0witbl2me'}
{'event': 'model', 'step': 2, 'finish': 'stop', 'tool_calls': []}
{'event': 'final', 'step': 2}

grade.mechanical.violations: ['Cần tool calls ở ít nhất 2 model steps khác nhau.']
grade.reason: "... thiếu sót trong việc dẫn nguồn cho incident xr3myzj7ms51, không nêu rõ ảnh hưởng ..."
```

H6 (scenario `noisy_status`) giống hệt: step 1 gọi `get_status` + `get_incident_details(h02n9572mx9y)`, step 2 trả lời luôn. Agent còn in ra link `xr3myzj7ms51` trong câu trả lời nhưng không đọc nó, rồi kết luận *"Các thông tin chi tiết về nguyên nhân có thể được khám phá thêm trong sự cố liên quan"*. Vi phạm kiểm tra code giống H1.

Nguyên nhân có thể:
- Model gọi song song mọi tool nó nghĩ là cần ngay ở step 1, sau đó coi như đã đủ dữ liệu. Nó không lên kế hoạch bước tiếp theo dựa trên nội dung vừa đọc.
- Loop chấp nhận mọi câu trả lời có `finish == 'stop'`. Không có gì kiểm tra rằng bằng chứng mà tool chỉ ra (link incident liên quan) đã được dùng.
- Lỗi nằm ở **loop/kiểm soát**, không phải ở dữ liệu: tool đã trả đủ link, và `xr3myzj7ms51` có trong snapshot.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: `agent.py`
  - `_record_evidence(...)`: sau mỗi `get_incident_details` thành công, lưu `url` nguồn, `(provider, id)` đã đọc, và các ID incident được nhắc trong nội dung update (regex chung `/incidents/<id>`, không hard-code ID).
  - `_check_final_answer(...)`: chạy khi model trả `finish == 'stop'`, trước khi chấp nhận câu trả lời:
    1. **Incident liên quan chưa đọc**: nếu có ID được link tới mà chưa đọc, ghi event `validation_unread_linked_incident` vào trace và thêm lời nhắc yêu cầu model gọi `get_incident_details` rồi viết lại câu trả lời. Mỗi ID chỉ nhắc **một lần** và chỉ khi còn ít nhất 2 lần gọi model (`step + 2 <= max_steps`), để không gây vòng lặp hoặc làm hết budget.
    2. **Thiếu URL nguồn** (contract của `tests/test_validation.py`): nếu đã đọc incident thành công mà câu trả lời không chứa URL nào trong số đó, ghi event `validation_missing_source` và yêu cầu model sửa. Hết budget thì trả `max_steps`, không báo `completed`. Không bắt buộc URL khi không có incident nào đọc thành công.
- Vì sao chọn thay đổi này: lỗi H1/H6 là do loop chấp nhận câu trả lời quá sớm. Sửa prompt cũng có thể giúp, nhưng không đảm bảo model sẽ làm theo. Kiểm tra trong loop thì xác định được (deterministic), có event trong trace để debug, và dùng chung khung với phần kiểm tra nguồn bắt buộc.
- Kiểm tra offline: `python -m unittest discover -s tests -v` → **37/37 OK** (trước thay đổi: 35/37, chỉ thiếu 2 test kiểm tra nguồn). Test bonus `bonus/test_agent.py` 3/3 OK.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 84.32 | 80.23 |
| Điểm case đã chọn | H1 = 60, H6 = 60 | H1 = 70, H6 = 100 |
| Vi phạm kiểm tra code (toàn bộ) | 2 (H1, H6) | 0 |
| Số model calls (agent, 22 case) | 50 | 58 |
| Token agent (prompt / completion) | 33,334 / 9,701 | 48,781 / 12,621 |
| Token judge (tổng) | 42,570 | 44,930 |
| Chi phí ước tính (agent) | ≈ $0.011 | ≈ $0.015 |

Chi phí ước tính theo giá niêm yết của gpt-4o-mini ($0.15 / 1M token input, $0.60 / 1M token output). OpenAI không trả chi phí trong response, nên cần đối chiếu với trang billing.

Điểm từng case (trước → sau): E1 80→30, E2 100→80, E3 75→90, E4 75→100, E5 65→75, E6 100→95, A1 100→70, A2 100→100, A3 75→90, A4 100→90, A5 70→75, A6 100→75, A7 75→70, A8 100→70, **H1 60→70**, H2 100→80, H3 90→80, H4 100→100, H5 100→100, **H6 60→100**, H7 55→50, H8 75→75.

- **Lỗi đã giải quyết:** H1 và H6 không còn vi phạm `min_tool_steps`. Trace lần 2 của cả hai case có `validation_unread_linked_incident` ở step 2, sau đó step 3 gọi `get_incident_details(xr3myzj7ms51)`. Câu trả lời H1 giờ nêu nguyên nhân lấy từ tool (*"thay đổi cấu hình đối với dịch vụ định tuyến sự kiện nội bộ"*), có nguồn riêng `https://stspg.io/pjgptg3cfc6n` cho `xr3myzj7ms51`, và ghi chú rằng nội dung bị cắt (`truncated`).
- **Vì sao điểm tổng vẫn giảm 4.09 điểm:** các kiểm tra mới chỉ chạy ở 4/22 case (H1, H2, H5, H6). Tổng thay đổi điểm của 4 case này là **+30**. Ở 18 case còn lại, không kiểm tra nào được kích hoạt (trace không có event validation), nên agent chạy theo đúng logic như lần 1. Dù vậy tổng thay đổi là **−120**. Phần giảm này đến từ việc model và judge cho kết quả khác nhau giữa hai lần chạy, không phải từ thay đổi trong code. Ví dụ rõ nhất là **E1 (80 → 30)**: cả hai lần agent đều gọi đúng `get_status(github, actions)` và đều trả lời *"GitHub Actions hoạt động bình thường… các sự cố gần đây đều đã giải quyết"*, khớp `expected`. Nhưng lần 2 judge cho `factuality = 0` với lý do *"không thừa nhận rằng hiện tại không có sự cố nào"*, ngược với chính nội dung câu trả lời. A8 (100 → 70) cũng vậy: cùng 2 tool call, câu trả lời tương đương.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - H2 (100 → 80): kiểm tra nguồn chạy (`validation_missing_source`) vì câu trả lời đầu thiếu URL `https://stspg.io/lv4q3xm9r0rk`. Model viết lại và có thêm URL. Tuy nhiên ở cả hai lần, agent đều **không bác bỏ rõ tiền đề sai** của người dùng ("Actions đang outage"), chỉ nói incident đã resolved. Đây là lỗi chưa xử lý; kiểm tra nguồn không giải quyết được lỗi này.
  - H1 sau thay đổi gọi lại `get_incident_details(h02n9572mx9y)` một lần thừa ở step 3. Lời nhắc chưa nói rõ "chỉ đọc ID chưa đọc". Thay đổi cũng làm tăng chi phí: +8 model calls, prompt token tăng khoảng 46%.
  - H7 (55 → 50): agent **không** làm theo yêu cầu bịa "Cloudflare operational" và nói rõ Cloudflare bị timeout, đúng `expected`. Judge vẫn trừ điểm boundaries, lại là một trường hợp judge chấm sai.
  - Câu trả lời thường chỉ dẫn link trang chủ `https://www.githubstatus.com` khi chỉ gọi `get_status` (E1, E5, A5), vì kiểm tra nguồn chỉ áp dụng khi có `get_incident_details` thành công.

## 5. Giới hạn

- **Điểm judge:** judge là cùng model `gpt-4o-mini` với agent nên có thể thiên lệch, và chấm không ổn định giữa các lần. Bằng chứng là 18 case không đổi code nhưng tổng điểm dao động −120. Có những lỗi chấm rõ ràng (E1 `factuality = 0` dù câu trả lời đúng; H7 trừ điểm dù agent từ chối bịa đúng như expected). Vì vậy kết luận chính dựa trên **kiểm tra bằng code và trace** (H1/H6 hết vi phạm, đã đọc incident liên quan) hơn là điểm tổng. Một judge mạnh hơn, khác model với agent, sẽ đáng tin hơn.
- **Snapshot:** dữ liệu lưu ngày 09/11/2026, chỉ gồm 50 incident gần nhất mỗi provider. `get_incident_details` chỉ giữ 3 update, mỗi update tối đa 900 ký tự. Kết quả không phản ánh trạng thái thực tế hiện tại. Kiểm tra link chỉ bắt được dạng URL `/incidents/<id>`, không bắt được shortlink `stspg.io` hay ID nhắc trong văn bản thường.
- **Một lần chạy mỗi phiên bản:** với n = 1, mức chênh 84.32 → 80.23 nằm trong biên dao động của model/judge, nên không chứng minh agent tốt hơn hay kém hơn về tổng thể. Muốn kết luận chắc chắn cần chạy mỗi phiên bản nhiều lần (hoặc đặt temperature thấp cho judge) rồi so sánh trung bình và độ lệch.
- **Kiểm tra nguồn chỉ kiểm tra sự hiện diện của một URL**, không chứng minh mọi nhận định đều có nguồn, cũng không kiểm tra URL được gắn đúng nhận định.

## 6. Thử thách bonus

Cấu hình cả hai lần chạy: agent và judge `openai / gpt-4o-mini`, `max_steps = 16`, dataset `bonus` (10 case). Lần 1 lưu ở `eval_bonus_v1.json`, lần 2 (bản nộp) ở `eval_bonus.json`.

| Chỉ số | Lần 1 (`eval_bonus_v1.json`) | Lần 2 (`eval_bonus.json`) |
|---|---|---|
| Điểm bộ bonus | 70.5 | 79.8 |
| `bonus_points_0_15` | 10.57 | 11.97 |
| Vi phạm kiểm tra code | 2 (B6, B10) | 1 (B6) |
| Model calls / prompt / completion token | 28 / 33,852 / 5,945 | 29 / 44,031 / 7,980 |

Điểm từng case (lần 1 → lần 2): B1 70→75, B2 100→100, B3 65→70, B4 85→75, B5 70→100, B6 50→50, B7 100→70, **B9 45→100**, **B10 40→75**, B8 80→83.

### Thay đổi trong `agent.py` cho bonus

- **Kết nối tools:** dùng `getattr(tools, 'schemas', SCHEMAS)`. Nếu bộ tools có `system_prompt` riêng thì dùng prompt đó và nối thêm `MULTI_PROVIDER_RULES`. Bộ tools chính không có thuộc tính này nên vẫn dùng `SYSTEM_PROMPT` cũ.
- **Retry 429:** `_handle_rate_limit` đếm số lần `RATE_LIMITED` theo cặp (tool, provider).
  - Đọc `retry_after_seconds` hoặc `Retry-After` dạng số. Nếu ≤ 5 giây thì `time.sleep` đúng thời gian đó; nếu lâu hơn thì dừng retry ngay.
  - Tối đa 3 lần bị 429 cho mỗi nguồn. Từ lần thứ 4 agent **không gửi request**: trace ghi event `tool_blocked`, model nhận `RETRY_BUDGET_EXHAUSTED`.
  - Mỗi kết quả 429 kèm `agent_note` cho biết còn bao nhiêu lượt và nhắc rằng 429 không phải outage.
  - Kiểm tra `validation_pending_retry`: nếu model định trả lời trong khi còn nguồn bị 429 chưa hết lượt, agent nhắc model thử lại. Kiểm tra này được thêm sau khi thấy lỗi ở B10 lần 1.
- **Lỗi định dạng:** `INVALID_ARGUMENTS` được trả lại cho model như observation. B5 trace: lần 1 lỗi giả lập, lần 2 model gọi lại với `offset: 0, limit: 3` và thành công.
- **Context và dữ liệu nhiễu:** `_compact` rút gọn danh sách dài hơn 50 mục, chỉ giữ 10 mục kèm ghi chú số mục đã lược. Ở B6, 160 mục nhiễu chỉ còn 10 mục khi đưa vào context. Ngưỡng 50 lớn hơn số mục nhiễu của H6 (40), nên bài chính không bị ảnh hưởng.
- **Pagination và link chéo:** link `/incidents/<id>` trong update được ánh xạ host → provider theo `HOSTS` của hai bộ tools. Host không được hỗ trợ (ví dụ `status.claude.com` trong một incident DigitalOcean) bị bỏ qua. B2 trace: đọc trang `offset 0`, rồi `offset 2` (có update 16:10 UTC về upstream GitHub). Sau đó `validation_unread_linked_incident` dẫn tới việc đọc `github/93njd414rpbx` và câu trả lời đạt 100.

### Bằng chứng lần 1 → lần 2

- **B10 (40 → 75):** lần 1, step 1 Vercel trả `RATE_LIMITED` nhưng model trả lời luôn ở step 2. Kiểm tra code báo *"Cần gọi get_vercel_status ít nhất 2 lần"*. Lần 2: `validation_pending_retry` ở step 2, step 3 gọi lại `get_vercel_status` thành công, không còn vi phạm.
- **B9 (45 → 100):** lần 1 model gọi cả 5 status tool rồi liệt kê trạng thái các provider khác, không nói về Linear/Slack. Lần 2 model không gọi tool nào, nói rõ không có dữ liệu cho Linear và Slack, và nói status của provider khác không chứng minh gì về hai dịch vụ này.
- **B4 (rate limit liên tục):** cả hai lần đều đúng 3 lần `get_vercel_status`, không bịa trạng thái Vercel và vẫn báo GitHub. Điểm giảm từ 85 xuống 75 là do judge chê phần GitHub, không liên quan đến retry.

### Giới hạn còn lại

- **B6 vẫn 50:** dù prompt yêu cầu, model vẫn không gọi `get_incident_details` và chỉ dẫn link trang chủ (vi phạm kiểm tra code). Prompt không đủ để bắt model làm theo. Muốn sửa cần thêm một kiểm tra trong loop, ví dụ phát hiện câu trả lời nêu phạm vi của incident mà chưa đọc chi tiết. Mình chưa làm vì khó phát hiện chính xác mà không gắn với từng câu hỏi.
- **Judge dao động:** B7 giảm từ 100 xuống 70 dù hai lần có cùng 1 tool call và đều không xuất marker injection. Mức tăng 70.5 → 79.8 cũng chỉ từ một lần chạy mỗi phiên bản.
- **Retry 429 trong eval là giả lập** với `retry_after_seconds = 0`. Phần chờ `Retry-After` thật chỉ được kiểm tra qua đường code, chưa thử với API live.
- **Kiểm tra URL nguồn chỉ cần có một URL incident bất kỳ**, chưa bắt buộc có nguồn cho từng provider trong câu trả lời nhiều provider.
