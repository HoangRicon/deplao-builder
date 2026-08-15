# Findings: fix-fb-name-avatar

## Nguyên nhân gốc (đã xác nhận bằng đọc code)

1. **Group chat hiện UID**: `ChatWindow.tsx:1195` — `if (isNonZalo(acc.channel)) return;` chặn FB khỏi group-member load; `saveGroupMembers`/`upsertGroupMember` (DatabaseService 4701/4721) không có caller FB nào.
2. **Owner key không nhất quán**: IPC `fb:getUserInfoFacebookHtml` (facebookIpc.ts:387-405) dùng `internalId` (UUID) làm owner; backend `checkAndFetchUserInfo` (FacebookService.ts:1641) + `saveFBMessage` (DatabaseService:7903) dùng `facebook_id` (số). UI query theo numeric → row IPC vô hình sau restart.
3. **Cross-contamination**: `checkAndFetchUserInfo` lookup (1623-1626) `WHERE contact_id=? AND channel='facebook' LIMIT 1` không filter owner → account B lấy contact account A.
4. **Scraper yếu** (FacebookSession.ts:264-275): tên fallback `"NAME":"(.*?)"` match đầu tiên trong HTML = có thể tên user đang login (page config); avatar "last scontent" (249-254) có thể là ảnh quảng cáo. Profile page không có `<h1><div role=button>` (SPA) → fallback hay chạy.
5. **Enrichment failure-sticky** (ChatWindow.tsx:1363-1399): `requestedMemberInfoRef` add trước, không delete trong finally → 1 lần fail = UID dính cả session; `updateContactProfile` dùng `activeAccountId` (UUID).
6. **E2EE senderId=0** (bridge/events.go:733-736): `ParseInt("4:123")` fail → senderID=0 → handleE2EEMessage fetch profile user "0" (garbage).

## Nguồn dữ liệu sạch sẵn có

- `parseThreadNodes` (FacebookThreadManager.ts:111) đã parse `all_participants.edges[].node.messaging_actor` (id, name, big_image_src.uri, profile_picture.uri) — dùng cho thread name/avatar hiện tại, CHƯA persist vào contacts.
- CDN `/picture?type=large` HEAD redirect (tryFetchCdnRedirect, FacebookSession.ts:300-324) — reliable theo nhận định ban đầu.
- `page_group_member` table dùng chung Zalo/Telegram — FB có thể dùng (owner_zalo_id=fbId, group_id=threadId).
- UI `getContact(msg.sender_id)` + `getGroupMember(msg.sender_id)` (ChatWindow:2725-2732) — match theo member_id = senderId numeric nếu member lưu đúng.

## Ràng buộc

- Không đổi schema DB (contacts + page_group_member đủ).
- Bridge Go cần rebuild exe (`npm run build:bridge-e2ee`) nếu đổi events.go — tránh đổi nếu không cần (T8 chỉ sửa TS guard, giữ Go nguyên).
- `fb_threads.is_e2ee` chỉ set khi gửi tin — ngoài phạm vi change này (thuộc change E2EE UX sau).