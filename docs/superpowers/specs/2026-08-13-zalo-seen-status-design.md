# Thiết kế: Trạng thái tin nhắn Đã gửi / Đã xem (+ danh sách người đã xem)

- Ngày: 2026-08-13
- Dự án: Deplao (deplao-crm)
- Trạng thái: đã duyệt bởi user

## Mục tiêu

Hiển thị trạng thái tin nhắn Zalo đã gửi từ tài khoản của mình:

1. **Tick trên từng tin nhắn**: 1 tick xám = đã gửi lên Zalo; 2 tick xanh = người nhận đã xem. Áp dụng cho chat 1-1 và nhóm.
2. **Nhóm — danh sách người đã xem trên từng tin**: hover vào vùng tick hiển thị tooltip danh sách tên + avatar những người đã xem tin đó.
3. **Giữ nguyên** phần tổng hợp "Đã xem + avatars" cuối hội thoại (đã có sẵn từ trước).
4. **Bền vững**: trạng thái seen lưu vào DB — mở lại app vẫn hiển thị đúng, kể cả chế độ nhân viên (employee qua boss).

Ngoài phạm vi:

- Không hiển thị trạng thái "đã nhận" (Zalo không có sự kiện này).
- Không thay đổi luồng gửi tin nhắn, message queue, hay trạng thái gửi cục bộ hiện có.
- Không áp dụng cho Facebook/Telegram (chỉ Zalo).

## Kiến trúc hiện tại (tái sử dụng)

- **Main process** `src/utils/ZaloLoginHelper.ts:900` — listener `seen` của zca-js nhận sự kiện:
  - `UserSeenMessage` (1-1): `{ idTo, msgId, realMsgId }`
  - `GroupSeenMessage` (nhóm): `{ msgId, groupId, seenUids[] }`
- `src/services/event/EventBroadcaster.ts:1328` — `broadcastSeen(zaloId, data)` → gửi `event:seen` tới renderer.
- **Renderer** `src/ui/hooks/useZaloEvents.ts:1646` — nhận `event:seen` → `chatStore.setSeen(zaloId, threadId, seenUids, msgId, isGroup)`.
- `src/ui/store/chatStore.ts:902` — `setSeen` lưu in-memory theo thread (`seenInfo[zaloId_threadId] = { msgId, seenUids, isGroup }`).
- `src/ui/components/chat/ChatWindow.tsx:3520` — render "Đã xem + avatars" cuối hội thoại nhóm từ `seenInfo`.
- **Employee mode**: `event:seen` đã được relay qua `HttpRelayService` / `HttpClientService`.
- **DB migration pattern**: `DatabaseService` dùng `ALTER TABLE ... ADD COLUMN` trong try/catch (vd dòng 1466-1555).

## Thiết kế chi tiết

### 1. DB — thêm cột vào bảng `messages` (migration an toàn)

Trong `DatabaseService` (khối migration khởi động), thêm:

```sql
ALTER TABLE messages ADD COLUMN is_seen INTEGER DEFAULT 0;
ALTER TABLE messages ADD COLUMN seen_uids TEXT DEFAULT NULL;   -- JSON array of uid
ALTER TABLE messages ADD COLUMN seen_at INTEGER DEFAULT NULL;   -- epoch ms
```

- Migration bọc `try/catch` như pattern có sẵn (idempotent).
- `getMessages*` queries trả kèm 3 cột mới (SELECT \* hiện có sẽ tự bao gồm — cần kiểm tra các câu SELECT liệt kê cột tường minh).

### 2. Main process — lưu seen vào DB + broadcast

Mở rộng `EventBroadcaster.broadcastSeen(zaloId, data)`:

- Parse event như hiện tại (user vs group).
- Gọi `DatabaseService.markMessageSeen(...)`:
  - Xác định `threadId` (từ `idTo`/`groupId`), `msgId` (ưu tiên `realMsgId` nếu có).
  - Update `is_seen=1`, `seen_at=now` cho tin có `msg_id = msgId` (hoặc `cli_msg_id = msgId`).
  - **Đánh dấu cả các tin của mình gửi trước đó trong cùng thread** (`sender_id = zaloId`, `is_sent=1`, `channel='zalo'`) chưa seen → seen (Zalo seen = đã mở thread, đọc toàn bộ). Giới hạn phạm vi bằng timestamp: lấy `timestamp` của tin seen từ DB (lookup theo msgId/cli_msg_id); nếu không tìm được tin seen (vd đã xoá), đánh dấu toàn bộ tin của mình trong thread.
  - Group: merge `seenUids` vào cột `seen_uids` (JSON, không trùng lặp) cho các tin đó.
- Giữ nguyên broadcast `event:seen` tới renderer (không đổi format) — kênh employee đã hoạt động.

Thêm hàm `DatabaseService.markMessageSeen(zaloId, threadId, msgId, seenUids, isGroup)`.

### 3. Renderer — tick per-message + tooltip danh sách người xem

**Tính trạng thái hiển thị của một tin** (trong `ChatWindow.tsx`/`MessageBubbles.tsx`, chỉ với tin `sender_id === activeAccountId` và channel zalo):

- `is_seen === 1` → 2 tick xanh
- ngược lại → 1 tick xám
- Tin đang gửi cục bộ (send_status = pending/sending) → spinner như hiện tại (ưu tiên hơn tick)
- Tin failed → icon lỗi như hiện tại (ưu tiên hơn)

**Tooltip nhóm**: khi hover vùng tick của tin có `seen_uids` → hiển thị danh sách (tên + avatar) của từng uid. Lấy tên/avatar từ `groupInfoCache` (members) hoặc `contacts` — đúng logic đã có ở `ChatWindow.tsx:3527-3539`. Tối đa hiển thị ~10 + "+N", hiện trên 1 popover nhỏ.

**Dữ liệu**: `Message` model thêm optional `is_seen?: number; seen_uids?: string; seen_at?: number`. `useZaloEvents` nhận `event:seen` → ngoài `setSeen` hiện có, **update trực tiếp message trong chatStore** (patch `is_seen/seen_uids/seen_at` cho tin khớp `msgId` + các tin trước đó trong thread) — dùng hàm cập nhật message tương tự `updateMessageStatus` đã có, hoặc `setMessages` patch theo `threadKey`.

## Xử lý lỗi

- Zalo event thiếu msgId/threadId → log + bỏ qua (không crash).
- Migration chạy lỗi → try/catch bỏ qua (app vẫn chạy, chỉ mất tính năng cho DB đó).
- `seen_uids` JSON hỏng khi parse → coi như rỗng.

## Kiểm thử

- Thủ công: đăng nhập Zalo, gửi tin 1-1 → 1 tick; mở Zalo bên kia đọc → 2 tick xanh realtime; tắt/mở app → tick giữ nguyên.
- Nhóm: gửi tin trong nhóm 2+ người → hover thấy danh sách người xem cập nhật khi từng người đọc; sau khi mọi người đọc → tick xanh.
- Employee mode: máy nhân viên nhận seen từ boss → tick đúng.
- Regression: gửi tin thất bại/đang gửi vẫn hiển thị trạng thái cục bộ đúng; Facebook/Telegram không đổi.