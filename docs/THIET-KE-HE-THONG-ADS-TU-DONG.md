# Thiết kế hệ thống Marketing tự động: Facebook Ads + TikTok Ads chạy trên Claude Code

> Ngày nghiên cứu: 13/11/2026. Mục "Nguồn & mức độ tin cậy" ở cuối tài liệu ghi rõ điều gì đã xác minh từ tài liệu chính thức và điều gì mới chỉ đến từ blog bên thứ ba. Cần đối chiếu lại tài liệu chính thức trước khi code.

---

## 1. Nguyên tắc thiết kế (quan trọng nhất)

"Hoàn toàn tự động" khả thi cho ~90% công việc lặp lại, nhưng một hệ thống không có phanh sẽ đốt tiền và làm mất tài khoản quảng cáo. Vì vậy thiết kế theo 3 vùng:

| Vùng | Ví dụ hành động | Cơ chế |
|---|---|---|
| **Xanh – tự chạy** | Đọc báo cáo, tạo báo cáo, tạm dừng ad lỗ vượt ngưỡng, tăng/giảm ngân sách ≤20%/ngày, tạo biến thể creative, gửi sự kiện CAPI | Agent tự làm, ghi nhật ký |
| **Vàng – tự chạy nhưng có giới hạn cứng** | Tạo chiến dịch mới trong hạn mức ngày, nhân bản adset thắng | Chạy trong "ngân sách ngân hàng" (budget cap) do con người đặt |
| **Đỏ – cần người duyệt** | Tăng hạn mức tổng, đổi tài khoản/thanh toán, ad thuộc ngành nhạy cảm, creative có người thật/giá cả/cam kết y tế, xóa chiến dịch | Gửi thông báo (Telegram/Zalo) chờ bấm duyệt |

Ba lớp an toàn bắt buộc: (1) **ngân sách cứng** ở cả tầng code lẫn tầng nền tảng (spending limit của ad account), (2) **mặc định tạo ở trạng thái PAUSED** rồi mới bật qua bước kiểm tra chính sách, (3) **nút dừng khẩn cấp** (kill switch).

## 2. Bức tranh nền tảng 2026 (đã nghiên cứu)

**Meta (Facebook/Instagram)**
- Cấu trúc chiến dịch đã đổi: chiến dịch Advantage+ Shopping/App (ASC/AAC) kiểu cũ **không còn tạo/sửa qua API**. Cách mới: tạo campaign `OUTCOME_SALES` với 3 thứ tự động bật cùng lúc — ngân sách Advantage+ ở cấp campaign, đối tượng Advantage+, vị trí hiển thị Advantage+ (trường `advantage_state`). Ví dụ của Meta dùng API v25.0.
- **Andromeda**: hệ thống xếp hạng ad của Meta xem *creative* là tín hiệu nhắm mục tiêu chính. Hệ quả cho thiết kế: cấu trúc gọn (ít campaign, broad targeting) và **nhiều concept creative thực sự khác nhau**, không phải đổi tiêu đề/màu nền.
- Quyền truy cập API: tự quản tài khoản của chính mình chỉ cần Standard Access + `ads_read`/`ads_management`; quản lý tài khoản khách cần Advanced Access (App Review + Business Verification). Standard yêu cầu app ở chế độ Live và có ≥1.500 lệnh gọi API trong 30 ngày.
- Có MCP chính thức của Meta (mcp.facebook.com/ads) nhưng theo một hướng dẫn, **Claude Code CLI đang lỗi `redirect_uris` khi đăng nhập OAuth** → không nên phụ thuộc. Phương án ổn định hơn: tự viết lớp gọi Marketing API.
- Chính sách: từ 03/2026 quảng cáo thương mại cũng phải khai báo nội dung AI (nguồn blog – cần kiểm tra lại).

**TikTok**
- Marketing API có Smart+ (gói tự động hóa theo module, tích hợp Symphony tạo creative bằng AI). Một nguồn nói endpoint campaign đã gộp vào "Upgraded Smart+ API" — chưa xác minh với tài liệu TikTok. Tính năng có thể chưa mở cho mọi tài khoản.
- Phải được duyệt app (sandbox → production), mỗi Business Center phải cấp quyền cho app.
- Events API: **bắt buộc dùng chung `event_id`** giữa Pixel và Events API để khử trùng; cửa sổ khử trùng 48 giờ. Token Events API (tạo trong Events Manager) khác với OAuth token của Marketing API.

**Việt Nam**
- Mô hình thực tế: TikTok/Facebook kéo nhận biết → **Zalo/Pancake/Inbox/COD chốt đơn**. Do đó chuyển đổi quan trọng nhất thường xảy ra *ngoài website* → phải đẩy dữ liệu đơn hàng thật (đã chốt, đã giao, hoàn) ngược về CAPI, nếu không thuật toán học sai tín hiệu.
- Rủi ro cần theo dõi: tin Meta hạn chế quảng cáo dẫn về TikTok (chưa xác minh). Nên tránh chạy Facebook Ads với link đích là TikTok/TikTok Shop.
- Kendo (repo này) đã có trang chuyển tiếp OAuth TikTok → có thể dùng làm `redirect_uri` cho bước cấp quyền TikTok Marketing API (xem mục 8).

## 3. Kiến trúc tổng thể

```
                        ┌───────────────────────────────┐
                        │   CON NGƯỜI (chủ doanh nghiệp) │
                        │ Telegram/Zalo: duyệt vùng đỏ   │
                        │ Báo cáo sáng/tuần, kill switch │
                        └──────────────┬────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────────┐
│                 ĐIỀU PHỐI (Claude Code – chạy theo lịch / sự kiện)            │
│  Orchestrator: đọc state → chọn agent → kiểm tra Guardrail → thực thi → log   │
├──────────┬───────────┬────────────┬────────────┬───────────┬─────────────────┤
│ Strategy │ Creative  │ Campaign   │ Optimizer  │ Analyst   │ Compliance      │
│ Agent    │ Agent     │ Builder    │ Agent      │ Agent     │ Agent           │
└────┬─────┴─────┬─────┴──────┬─────┴──────┬─────┴─────┬─────┴────────┬────────┘
     │           │            │            │           │              │
┌────▼───────────▼────────────▼────────────▼───────────▼──────────────▼───────┐
│  LỚP CÔNG CỤ (tools/MCP tự viết, có kiểm soát quyền)                          │
│  meta_api · tiktok_api · capi_sender · storage · notifier · guardrail         │
└────┬──────────────┬──────────────┬──────────────┬───────────────┬────────────┘
     │              │              │              │               │
  Meta Marketing  TikTok Marketing  CAPI/Events   Kho dữ liệu    CRM/đơn hàng
  API + Insights  API + Events API  (server-side) (SQLite/PG)    (Pancake/Sapo/
                                                                  Haravan/Zalo)
```

**Vì sao viết lớp công cụ riêng thay vì dùng MCP bên thứ ba (Pipeboard, Ryze, Adkit…)?** MCP bên thứ ba dùng nhanh để *thử nghiệm* nhưng bạn giao token quảng cáo và dữ liệu cho họ, không kiểm soát được guardrail, và khó audit. Khuyến nghị: giai đoạn đầu dùng MCP có sẵn ở chế độ **chỉ đọc**, sản xuất thì tự viết (Python/TypeScript, vài trăm dòng cho mỗi nền tảng).

## 4. Sáu agent và nhiệm vụ

1. **Strategy Agent (chiến lược)** — Đầu vào: sản phẩm, biên lợi nhuận, giá, đối thủ, mùa vụ. Đầu ra: *brief tuần* gồm mục tiêu ROAS/CPA hòa vốn, phân bổ ngân sách Facebook/TikTok, 5–8 concept creative. Tính **CPA hòa vốn = giá bán × biên LN gộp × tỷ lệ giao thành công (trừ hoàn COD)**.
2. **Creative Agent (nội dung)** — Sinh kịch bản video ngắn (hook 3 giây, vấn đề → giải pháp → bằng chứng → CTA), caption tiếng Việt, ảnh/video qua công cụ tạo (Symphony của TikTok, hoặc API tạo ảnh/video riêng). Ép **đa dạng concept thật sự**: mỗi đợt tối thiểu 5 góc tiếp cận khác nhau (nỗi đau, so sánh, UGC/đánh giá, ưu đãi, câu chuyện). Gắn nhãn AI theo chính sách.
3. **Campaign Builder (dựng chiến dịch)** — Tạo campaign/adset/ad qua API, luôn ở PAUSED, đặt tên theo chuẩn: `[FB|TT]_[mục tiêu]_[sản phẩm]_[concept]_[YYYYMMDD]`. Meta: `OUTCOME_SALES` + Advantage+ đủ 3 thành phần. TikTok: Smart+ nếu tài khoản được bật, ngược lại dùng campaign thường.
4. **Optimizer Agent (tối ưu, chạy mỗi 3–6 giờ)** — Áp dụng luật quyết định ở mục 5. Không sửa sau khi mới đổi < 48–72h (tránh reset giai đoạn học).
5. **Analyst Agent (phân tích)** — Mỗi sáng gửi báo cáo tiếng Việt: chi tiêu, đơn, CPA/ROAS *thật* (theo đơn đã chốt, không chỉ theo pixel), creative thắng/thua, cảnh báo bất thường. Hàng tuần: rút bài học → ghi vào kho tri thức để Strategy Agent dùng lại.
6. **Compliance Agent (tuân thủ)** — Quét creative/landing trước khi bật: từ cấm (cam kết chữa bệnh, "100%", so sánh tuyệt đối), nhãn AI, quyền hình ảnh, ngành hạn chế. Theo dõi trạng thái ad bị từ chối và sức khỏe tài khoản.

## 5. Luật quyết định cho Optimizer (khởi điểm, cần hiệu chỉnh theo ngành)

Gọi `CPA_hv` = CPA hòa vốn, `X` = ngân sách ngày của ad.

| Điều kiện (đủ dữ liệu) | Hành động |
|---|---|
| Đã tiêu ≥ 2×`CPA_hv`, 0 đơn | Tạm dừng ad |
| Tiêu ≥ 3×`CPA_hv`, CPA > 1,5×`CPA_hv` | Tạm dừng |
| CPA ≤ 0,8×`CPA_hv` liên tục 3 ngày, ≥ 15 đơn/7 ngày | Tăng ngân sách 15–20%/ngày (trần: hạn mức ngày) |
| CPA trong 0,8–1,2×`CPA_hv` | Giữ nguyên, quan sát |
| CTR/hook rate giảm >30% so với tuần trước, tần suất >3 | Gắn cờ "mệt creative" → yêu cầu Creative Agent làm biến thể mới |
| Concept thắng | Nhân bản sang tệp/vị trí mới + tạo 3 biến thể (hook khác) |
| Chi tiêu bất thường >150% dự kiến/ngày | Dừng tất cả + báo động ngay |

Quy tắc chung: không quyết định khi dữ liệu thấp (ví dụ <50 lượt nhấp), không đổi cùng lúc nhiều biến, và mọi quyết định ghi lại *lý do + số liệu* để kiểm toán.

## 6. Hạ tầng đo lường (nền móng – làm trước khi chạy tiền thật)

1. **Pixel + Conversions API (Meta)** và **Pixel + Events API (TikTok)** cùng lúc, chung `event_id` để khử trùng.
2. Gửi `Purchase` thật từ hệ thống đơn hàng (Pancake/Sapo/Haravan/webhook), băm SHA-256 email/SĐT (chuẩn hóa +84), kèm `fbc/fbp` và `ttclid/ttp`. Đẩy sự kiện **đơn đã xác nhận/đã giao**, trừ đơn hoàn → đây là điều tạo lợi thế lớn nhất với thị trường COD.
3. Với kênh Messenger/Zalo: lưu `click_id` lúc khách bấm quảng cáo, gắn vào hội thoại để quy chiếu đơn về ad.
4. Kiểm tra chất lượng khớp (Event Match Quality) và cảnh báo khi tụt.
5. Tuân thủ đồng thuận dữ liệu cá nhân (Nghị định 13/2023/NĐ-CP).

## 7. Kho dữ liệu & bộ nhớ

- DB đơn giản (SQLite → Postgres): `campaigns`, `adsets`, `ads`, `daily_metrics`, `orders`, `creatives`, `decisions` (log mọi hành động + lý do), `learnings` (bài học theo ngành/creative).
- Thư mục `knowledge/`: brand voice, USP, FAQ, các hook thắng, danh sách từ cấm. Claude đọc trước khi sinh nội dung.
- Mọi lần chạy ghi `decisions` bất biến để quay lại điều tra.

## 8. Cách chạy trên Claude Code

- **Lịch chạy**: Claude Code chạy headless theo cron/CI (ví dụ GitHub Actions hoặc scheduled task): `optimizer` mỗi 4 giờ, `analyst` 7:30 sáng, `strategy` thứ Hai, `creative` thứ Tư.
- **Cấu trúc thư mục đề xuất**:
```
ads-system/
├─ CLAUDE.md                  # luật toàn hệ thống, guardrail, tiếng Việt
├─ .claude/agents/            # strategy.md creative.md builder.md optimizer.md analyst.md compliance.md
├─ .claude/skills/            # skill: bao-cao-ngay, tao-chien-dich, kiem-tra-chinh-sach
├─ .claude/settings.json      # quyền: chỉ cho phép chạy tools/ đã duyệt; chặn ghi ngoài
├─ tools/
│  ├─ meta_api.py   tiktok_api.py   capi_sender.py
│  ├─ guardrail.py            # kiểm tra hạn mức, % thay đổi, trạng thái PAUSED
│  └─ notifier.py             # Telegram/Zalo
├─ config/limits.yaml         # budget_ngay, max_tang_%, CPA_hoa_von, ngành cấm
├─ data/                      # SQLite, báo cáo
└─ knowledge/                 # brand, hook thắng, từ cấm
```
- **Bí mật**: token để trong biến môi trường/secret manager, không commit. Dùng **System User token** (Meta) và token có thời hạn (TikTok), xoay vòng định kỳ.
- **Trang chuyển tiếp TikTok trong repo hiện tại** (`index.html`) có thể làm `redirect_uri` cho luồng OAuth của TikTok Marketing API; bước đổi `auth_code` lấy `access_token` nên làm ở server, không ở trang tĩnh.

## 9. Lộ trình triển khai

| Giai đoạn | Thời gian | Việc | Tiêu chí qua |
|---|---|---|---|
| 0. Nền móng | Tuần 1–2 | Business Manager/Business Center xác minh, Pixel + CAPI/Events API, đồng bộ đơn hàng, tạo app Meta & TikTok | Event Match Quality tốt, đơn thật về đúng ad |
| 1. Chỉ đọc | Tuần 3 | Analyst Agent + báo cáo sáng | Số liệu khớp trình quản lý quảng cáo |
| 2. Có người duyệt | Tuần 4–6 | Builder + Creative + Optimizer ở chế độ *đề xuất*, người bấm duyệt | ≥80% đề xuất được duyệt |
| 3. Tự động có hạn mức | Tuần 7–10 | Bật vùng xanh/vàng, ngân sách nhỏ | Không vượt hạn mức, CPA ≤ hòa vốn |
| 4. Mở rộng | Tuần 11+ | Tăng hạn mức theo ROAS, thêm kênh/sản phẩm | ROAS bền 4 tuần |

## 10. Rủi ro và cách giảm

| Rủi ro | Giảm thiểu |
|---|---|
| Agent đốt ngân sách do lỗi/hallucination | Hạn mức cứng ở code + spending limit ở nền tảng; kiểm tra bằng `guardrail.py` trước *mọi* lệnh ghi; kill switch |
| Tài khoản bị hạn chế | Tạo ad PAUSED → Compliance duyệt; tốc độ API vừa phải; tránh sửa hàng loạt; không chạy nhiều tài khoản mờ ám |
| Dữ liệu đo sai → tối ưu sai | CAPI + khử trùng `event_id`; đối chiếu hàng tuần với đơn thật |
| Creative lặp, mệt nhanh | Bắt buộc đa dạng concept, đo hook rate/tần suất |
| Vi phạm chính sách/AI label | Compliance Agent + nhãn AI tự động |
| Lệ thuộc nền tảng đổi API | Cố định phiên bản API, theo dõi changelog, kiểm thử hàng tuần |
| Prompt injection (nội dung bình luận, tên sản phẩm chứa lệnh) | Agent coi dữ liệu ngoài là dữ liệu, không là lệnh; công cụ ghi chỉ nhận tham số có kiểm tra |
| Hạn chế quảng cáo dẫn về TikTok trên Meta (chưa xác minh) | Không dẫn chéo Facebook → TikTok; dùng website/Zalo làm đích |

## 11. Chi phí tham khảo cần xác thực

CPM TikTok Việt Nam ước tính 25.000–50.000đ (nguồn blog, chưa kiểm chứng). Không có số liệu CPA theo ngành đáng tin cậy → hệ thống phải **tự học** từ tài khoản của bạn trong 4–6 tuần đầu thay vì tin con số trung bình.

---

## Nguồn & mức độ tin cậy

**Tài liệu chính thức / gần chính thức (đáng tin hơn):**
- [Meta – Ads Management Standard Access](https://developers.secure.facebook.com/docs/features-reference/ads-management-standard-access)
- [Meta – Access and Authentication](https://developers.facebook.com/docs/marketing-api/access/)
- [Meta – Marketing API v24.0 changelog](https://developers.facebook.com/documentation/ads-commerce/marketing-api/marketing-api-changelog/version24.0)
- [Meta – ASC/AAC deprecation & migration](https://developers.facebook.com/blog/post/2025/10/08/upcoming-asc-and-aac-mapi-deprecation-migration-options-to-advantage-plus/)
- [Meta – Advantage+ Shopping Campaigns (docs)](https://developers.secure.facebook.com/documentation/ads-commerce/marketing-api/advantage-shopping-campaigns)
- [TikTok – Event deduplication](https://ads.tiktok.com/help/article/event-deduplication)
- [TikTok – About upgrades to Smart+](https://ads.tiktok.com/resources/help/article/about-updates-to-smart-plus)
- [TikTok – Smart+ giới thiệu](https://ads.tiktok.com/business/blog/smart-plus-ai-performance-solution)

**Báo chí ngành / blog bên thứ ba (cần kiểm tra lại):**
- [PPC Land – Meta unified API structure for Advantage](https://ppc.land/meta-launches-unified-api-structure-for-advantage-campaigns/) · [Meta deprecates legacy campaign APIs](https://ppc.land/meta-deprecates-legacy-campaign-apis-for-advantage-structure/)
- [Meta Ads MCP: hướng dẫn (mcp.directory)](https://mcp.directory/blog/meta-ads-mcp-complete-guide-2026) · [Meta vs Google vs TikTok Ads MCP](https://mcp.directory/blog/meta-ads-vs-google-ads-vs-tiktok-ads-mcp-2026) · [Pipeboard Meta Ads MCP](https://mcpservers.org/servers/pipeboard-co/meta-ads-mcp)
- [Andromeda & creative diversity (Segwise)](https://segwise.ai/blog/meta-andromeda-update-creative-strategy-2026) · [Billo](https://billo.app/blog/meta-andromeda-update/)
- [Meta AI disclosure 2026 (cinerads)](https://www.cinerads.com/blog/ai-ad-disclosure-requirements)
- [Ecomobi – TikTok Ads có hiệu quả không 2026](https://ecomobi.com/vi/tiktok-ads-co-hieu-qua-khong-2026/)
- [VnBusiness – Meta chặn quảng cáo dẫn về TikTok](https://vnbusiness.vn/meta-chan-quang-cao-dan-ve-tiktok-nha-ban-hang-viet-bi-anh-huong-ra-sao.html)

**Chưa xác minh được (cần kiểm tra trực tiếp trước khi xây):** endpoint/tên tham số TikTok Smart+ API và rate limit; hạn mức rate limit chính xác của Meta; ngày cuối ngưng ASC/AAC (nguồn mâu thuẫn); chính sách Meta/TikTok về quảng cáo do bot/tự động quản lý; quy định quảng cáo riêng của Việt Nam; hướng tích hợp tracking cho Haravan/Sapo.
