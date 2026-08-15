# Tasks: fix-fb-name-avatar

## 1. Expose participants trong parseThreadNodes

- [x] File: `src/services/facebook/FacebookThreadManager.ts` (104-171)
- [x] Acceptance: output thread kèm `participants: {id, name, avatarUrl}[]` từ all_participants.edges
- [x] Test: script electron — group "Fan anh Độ Mixi" 9 participants tên/avatar đúng

## 2. Persist participants → contacts + group members

- [x] File: `src/services/facebook/FacebookService.ts` (persistThreadParticipants sau getThreadList)
- [x] Acceptance:
  - [x] Upsert contacts owner=fbId numeric, ON CONFLICT không ghi đè tên bằng rỗng
  - [x] Group thread → saveGroupMembers(fbId, threadId, members)
- [x] Test: contacts owner=facebook_id=232 rows; page_group_member=9 rows cho group

## 3. UI: FB group members vào cache

- [x] File: `src/ui/components/chat/ChatWindow.tsx` (1126, 1195)
- [x] Acceptance: mở group FB → buildAndSetGroupInfo từ page_group_member (owner fbId); bubble hiển thị displayName thay UID
- [x] Test: chờ user test UI "Fan anh Độ Mixi"/"Box NRO"

## 4. IPC owner key = facebook_id

- [x] File: `electron/ipc/facebookIpc.ts` (383-414)
- [x] Acceptance: INSERT/UPDATE contacts dùng owner = acc.facebook_id (không phải UUID)
- [x] Test: grep + code review — dùng fbAcc.facebook_id

## 5. Owner filter chống cross-contamination

- [x] Files: `FacebookService.ts` (551, 657, 770, 1625), `DatabaseService.ts` (7956)
- [x] Acceptance: mọi contact lookup/update của FB có `AND owner_zalo_id = ?`
- [x] Test: grep — không còn query contacts thiếu owner trong FB paths

## 6. Scraper og:title + avatar order + guard rác

- [x] File: `src/services/facebook/FacebookSession.ts` (225-293)
- [x] Acceptance: name ưu tiên og:title (strip suffix); avatar thứ tự 168px → profile_pic_uri → CDN → mbasic → scontent cuối; guard tên rác ("Xem thêm"/"Show more"/"Error"/digits)
- [x] Test: script electron — name="HTool Việt Nam" khớp thread name (lần trước trả "Xem thêm")

## 7. Enrichment UI: finally clear + đúng owner

- [x] File: `src/ui/components/chat/ChatWindow.tsx` (1361-1402)
- [x] Acceptance: requestedMemberInfoRef delete trong finally; updateContactProfile dùng acc.facebook_id
- [x] Test: code review + chờ user test

## 8. E2EE senderId=0 guard

- [x] File: `src/services/facebook/FacebookService.ts` (1373, 547, 683)
- [x] Acceptance: senderID=0 → userID='' → không fetch profile "0"
- [x] Test: code review (không đổi Go bridge — chỉ TS guard)

## 9. Verify toàn diện + archive

- [x] `tsc -p tsconfig.electron.json` 0 lỗi
- [x] Chạy script electron tổng hợp (participants persist, scraper, contact owner) — PASS
- [ ] Test UI: group FB hiện tên đúng; 1-1 tên/avatar đúng; không contamination 2 account
- [ ] `openspec archive fix-fb-name-avatar` + update checkbox