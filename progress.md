# Progress: fix-fb-name-avatar

## Checkpoints

| Date | Status | Note |
|---|---|---|
| 2026-08-14 | ✅ Spec | Spec valid (openspec validate), user confirmed (Gate G1) |
| 2026-08-14 | ✅ Planning | task_plan.md + findings.md tạo xong (Gate G2) |
| 2026-08-14 | ✅ T1-T8 | Implementation xong, tsc 0 lỗi |
| 2026-08-14 | ✅ Verify backend | Script electron PASS (chi tiết dưới) |
| 2026-08-14 | 🔄 UI test | App restarted, chờ user xác nhận hiển thị |

## T1. Expose participants trong parseThreadNodes
- **Trạng thái**: ✅ done
- **Verification evidence**: group "Fan anh Độ Mixi" (833190669769543) → 9 participants: "Văn Công" (61552807417498), "Trần Minh Khôi" (100024455161025)... avatar scontent URL hợp lệ

## T2. Persist participants → contacts + group members
- **Trạng thái**: ✅ done
- **Verification evidence**: contacts owner=100056964890740 = 232 rows; page_group_member group 833190669769543 = 9 rows

## T3. UI: FB group members vào cache
- **Trạng thái**: ✅ code done, 🔄 chờ user test UI
- **Verification evidence**: ChatWindow load bằng fbOwnerId; log db:getMessages zaloId=100056964890740 (numeric) — owner đúng

## T4. IPC owner key = facebook_id
- **Trạng thái**: ✅ done
- **Verification evidence**: facebookIpc.ts dùng fbAcc.facebook_id

## T5. Owner filter chống contamination
- **Trạng thái**: ✅ done
- **Verification evidence**: grep — 4 query contacts trong FacebookService + 1 trong DatabaseService đều có owner_zalo_id

## T6. Scraper og:title + avatar order + guard rác
- **Trạng thái**: ✅ done
- **Verification evidence**: scraper 1052024084656604 → name="HTool Việt Nam" (khớp thread name); lần test đầu trả "Xem thêm" (rác) → guard chặn tên rác đã thêm

## T7. Enrichment UI: finally clear + đúng owner
- **Trạng thái**: ✅ done
- **Verification evidence**: finally delete requestedMemberInfoRef; updateContactProfile zaloId=acc.facebook_id

## T8. E2EE senderId=0 guard
- **Trạng thái**: ✅ done
- **Verification evidence**: userID='' khi senderId=0; 2 caller checkAndFetchUserInfo thêm !== '0'

## T9. Verify toàn diện + archive
- **Trạng thái**: 🔄 chờ UI test + archive
- **Verification evidence**: tsc 0 lỗi; debug-name-avatar.js PASS toàn bộ (thấy trên)