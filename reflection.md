# Individual Reflection — Nguyễn Tiến Đạt (2A202600217)

**Project:** VinmecPrep AI — Chatbot hỗ trợ bệnh nhân chuẩn bị trước khi khám tại Vinmec  
**Track:** Vinmec  
**Prototype level:** Working prototype

---

## 1. Role trong nhóm

Backend AI Engineer — phụ trách toàn bộ phần backend, AI pipeline và infrastructure.  
Cụ thể: thiết kế LangGraph agent, xây dựng hệ thống guardrails, security hardening, tích hợp Kafka/Redis, và viết system prompt.

---

## 2. Đóng góp cụ thể (có output rõ ràng)

**a. LangGraph ReAct Agent + LiteLLM routing (`src/agent/vinmec_agent.py`)**  
Xây dựng agent với 3 nhóm tools: RAG (Weaviate), web search (Serper + SearXNG fallback), và hospital finder. Chuyển từ `langchain-groq` trực tiếp sang `ChatLiteLLM` để agent có thể switch giữa Groq API và vLLM Qwen3-4B local chỉ bằng đổi env var — không cần sửa code.

**b. Guardrails 5 lớp (`src/guardrails.py`)**  
Tự thiết kế và implement pipeline kiểm tra input trước khi đến agent: giới hạn độ dài (>2000 ký tự), block nội dung bạo lực/jailbreak, detect cấp cứu → redirect 115, lọc off-topic dựa trên medical keyword list, và cảnh báo PII (CCCD/SĐT/email). False positive thấp vì lớp off-topic chỉ block khi không có từ khoá y tế — "đau dạ dày sau khi ăn canh rau" qua được, "cách nấu canh rau" bị chặn.

**c. Hospital Finder tool + Static DB 10 cơ sở Vinmec (`src/tools/hospital_finder.py`)**  
Tạo mới hoàn toàn. Tool dùng Serper Places API để tìm real-time có rating, fallback về static database khi Serper không có kết quả, tính khoảng cách bằng Haversine. Database có đầy đủ địa chỉ, hotline, giờ làm việc, thông tin đỗ xe, tuyến xe buýt cho 10 cơ sở từ Hà Nội đến Phú Quốc.

---

## 3. SPEC — phần mạnh nhất và yếu nhất

**Mạnh nhất: Failure modes**  
Nhóm nghĩ ra được 3 failure mode thực sự nguy hiểm thay vì chỉ liệt kê lỗi kỹ thuật. Đặc biệt là case "triệu chứng mơ hồ + AI tự tin cao" — bệnh nhân nhận checklist sai chuyên khoa nhưng không biết, đi khám xong mới biết bị chuyển viện. Mitigation cụ thể: thêm câu follow-up hỏi thêm triệu chứng khi query ngắn hơn 10 từ.

**Yếu nhất: ROI**  
3 kịch bản ROI thực ra chỉ thay đổi số user, assumption nền gần như giống nhau. Conservative và realistic dùng cùng cost-per-query, không phân biệt được scenario "chỉ deploy 1 chi nhánh" với "rollout toàn hệ thống". Nếu làm lại sẽ tách assumption rõ hơn về scope triển khai và mô hình LLM (Groq free vs paid vs vLLM self-host) vì cost thay đổi rất khác nhau giữa 3 option đó.

---

## 4. Đóng góp khác

- **Security hardening cho nginx và API server:** phát hiện và fix lỗ hổng rate limit bypass qua `X-Forwarded-For` giả — nginx cũ dùng `$proxy_add_x_forwarded_for` nên client có thể gửi header giả để bypass giới hạn. Fix bằng cách buộc nginx set `X-Real-IP = $remote_addr`, không tin bất kỳ header nào từ client.
- **CORS fix:** `allow_origins=["*"]` là lỗi bảo mật nghiêm trọng trong production — đổi sang đọc từ env var `ALLOWED_ORIGINS`, mặc định chỉ cho phép localhost.
- **SearXNG config:** fix lỗi `FileNotFoundError: google images.py / youtube.py` do cách disable engine sai — đổi từ `disabled: true` sang `keep_only` theo đúng cú pháp SearXNG settings.

---

## 5. Điều học được trong hackathon mà trước đó chưa biết

Trước hackathon, mình thiết kế guardrails theo kiểu "block càng nhiều càng an toàn". Sau khi test thực tế mới nhận ra false positive mới là vấn đề lớn hơn false negative — nếu guardrail quá chặt, bệnh nhân hỏi hợp lệ bị từ chối và bỏ dùng app, còn tệ hơn là để lọt một vài câu off-topic.

Từ đó mới hiểu cách thiết kế đúng: **không block dựa trên absence of medical keywords mà pass dựa trên presence** — tức là chỉ block khi chắc chắn không có từ khoá y tế, thay vì block khi không chắc có. Logic tưởng đơn giản nhưng thay đổi hoàn toàn cách viết regex và thứ tự các lớp guardrail.

---

## 6. Nếu làm lại, đổi gì

Sẽ viết và chạy smoke test cho toàn bộ API endpoints từ **trước khi build feature mới**, không phải sau. Thực tế là mình phát hiện ra `web_search_tool.py` rỗng hoàn toàn (0 bytes) và `server.py` không tồn tại **khi đang debug** giữa hackathon — nếu có `scripts/_run_smoke.py` chạy từ đầu thì đã phát hiện sớm hơn 2-3 tiếng, dành thời gian đó để polish RAG data và viết thêm test cases cho guardrails thay vì chạy theo bug.

---

## 7. AI giúp gì — AI sai/mislead ở đâu

**AI giúp được:**
- Dùng Claude để brainstorm failure modes — nó gợi ý được case "bệnh nhân nhập triệu chứng bằng tiếng Anh nhưng hệ thống ưu tiên Vietnamese RAG" mà nhóm không nghĩ ra, dẫn đến việc thêm bilingual handling vào system prompt.
- Dùng AI để sinh nhanh static database 10 cơ sở Vinmec với schema đồng nhất (địa chỉ, hotline, tọa độ GPS, giờ mở cửa, đỗ xe, xe buýt) — việc này làm tay mất khoảng 1-2 tiếng.
- Claude giúp debug lỗi `asyncio.get_event_loop()` deprecated trong FastAPI lifespan — chỉ ra đúng vấn đề là phải dùng `asyncio.get_running_loop()` trong async context.

**AI sai/mislead:**
- Khi hỏi về cách fix SearXNG engine config, Claude đề xuất comment out các engine trong `settings.yml` — đó là cách viết sai cú pháp, chạy không báo lỗi nhưng không có tác dụng gì. Phải tự đọc docs SearXNG mới biết phải dùng `keep_only`. Bài học: AI rất tự tin với config syntax mà nó không chắc chắn.
- Claude gợi ý thêm Playwright MCP để agent có thể tự điền form đặt lịch trên website Vinmec — nghe hay nhưng mỗi browser instance tốn ~300MB RAM, không thể dùng per-request cho production. Suýt scope creep nếu không dừng lại. AI brainstorm tốt nhưng không tự tính toán resource constraint.
