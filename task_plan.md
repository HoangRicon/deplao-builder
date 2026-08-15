# Task Plan: fix-fb-name-avatar

## Tổng quan
Fix tên/avatar Facebook chat: 7 thay đổi. Spec: openspec/changes/fix-fb-name-avatar/
Thứ tự: phụ thuộc từ backend → frontend → verify.

## Checklist

### T1. Expose participants trong parseThreadNodes
- File: `src/services/facebook/FacebookThreadManager.ts:104-171`
- Thêm `participants: {id, name, avatarUrl}[]` vào mỗi FBThread output
- Test: script electron in participants group "Box NRO"

### T2. Persist participants → contacts + group members
- File: `src/services/facebook/FacebookService.ts` (sau saveFBThreads) + `DatabaseService.ts`
- Phụ thuộc: T1
- Test: fb:getThreads → contacts mới owner=fbId

### T3. UI: FB group members vào cache
- File: `src/ui/components/chat/ChatWindow.tsx:1126-1195`
- Phụ thuộc: T2 (dữ liệu page_group_member)
- Test: mở "Box NRO" thấy tên thành viên

### T4. IPC owner key = facebook_id
- File: `electron/ipc/facebookIpc.ts:383-414`
- Test: gọi IPC → row contacts đúng owner

### T5. Owner filter chống contamination
- File: `FacebookService.ts:1623`, `DatabaseService.ts:7903/7956`, handleIncomingMessage lookups
- Test: grep không còn contact lookup thiếu owner

### T6. Scraper og:title + avatar order
- File: `src/services/facebook/FacebookSession.ts:225-293`
- Test: script electron fetch profile → name đúng

### T7. Enrichment UI: finally clear + đúng owner
- File: `src/ui/components/chat/ChatWindow.tsx:1361-1402`
- Test: fetch fail → retry được

### T8. E2EE senderId=0 guard
- File: `src/services/facebook/FacebookService.ts:1373` + rebuild bridge
- Test: không fetch user "0"

### T9. Verify toàn diện + archive
- tsc 0 lỗi, script electron, test UI, openspec archive

## File structure mapping
| File | Task |
|---|---|
| FacebookThreadManager.ts | T1 |
| FacebookService.ts | T2, T5, T8 |
| DatabaseService.ts | T2, T5 |
| ChatWindow.tsx | T3, T7 |
| facebookIpc.ts | T4 |
| FacebookSession.ts | T6 |
| bridge/events.go | T8 (rebuild) |

## Test strategy
- Backend: script electron debug (pattern chuẩn, require dist-electron)
- Bridge: build lại exe + log
- UI: test thủ công qua app dev (nhờ user)