# 2A202600217-NguyenTienDat-Day06

**Họ tên:** Nguyễn Tiến Đạt  
**MSSV:** 2A202600217  
**Track:** Vinmec  
**Project:** VinmecPrep AI — Chatbot hỗ trợ bệnh nhân chuẩn bị trước khi khám

---

## Về project

VinmecPrep AI là working prototype chatbot giúp bệnh nhân:
- Biết cần chuẩn bị gì trước buổi khám (nhịn ăn, giấy tờ, đặt lịch)
- Nhận checklist cá nhân hoá theo từng chuyên khoa
- Tìm cơ sở Vinmec gần nhất kèm địa chỉ, hotline, giờ làm việc

Stack: **LangGraph ReAct Agent** · **FastAPI** · **Kafka + Redis Streams** · **Weaviate RAG** · **LiteLLM** (Groq / vLLM) · **SearXNG** · **Docker Compose**

**→ Repo nhóm:** *(link group repo)*

---

## Architecture

![Architecture](extras/architecture.png)

Flow tổng quát:

```
User Input
    │
    ▼
Guardrails (5 lớp)          ← src/guardrails.py
    │ PASS
    ▼
FastAPI /chat
    │
    ▼
Kafka Producer ──► Topic: vinmec.chat.jobs
                        │
                        ▼
                  Kafka Consumer
                        │
                        ▼
              LangGraph ReAct Agent
              ┌─────────────────────┐
              │  RAG (Weaviate)     │
              │  Web Search         │
              │  Hospital Finder    │
              └─────────────────────┘
                        │
                        ▼
                Redis (result cache)
                        │
                        ▼
              FastAPI poll → Response
```

---

## Nội dung repo này

| File/Folder | Mô tả |
|---|---|
| `reflection.md` | Individual reflection — role, đóng góp, bài học |
| `extras/prompt-test-log.md` | Log test system prompt qua 15+ test cases |
| `extras/guardrails-design.md` | Quá trình thiết kế 5 lớp guardrails — iterations và lý do chọn |
| `extras/security-fixes-notes.md` | Ghi chú khi tìm và fix 3 lỗ hổng bảo mật |

---

*VinUni A20 — AI Thực Chiến · Day 06 · 09/04/2026*
