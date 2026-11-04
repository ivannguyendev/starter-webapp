# starter-webapp

Repo khởi tạo cho dự án web app, có sẵn bộ cấu hình agent (Claude Code và các AI agent khác).
Dev chỉ cần đưa yêu cầu. Agents sẽ brainstorm, lập kế hoạch theo từng phase, code, test và báo cáo tiến độ theo một workflow cố định. Mỗi bước đều chờ dev duyệt.

Stack không cố định, được chốt ở bước brainstorm của từng dự án. Gợi ý theo skills có sẵn: **React + TypeScript**, **Node.js**, **Docker**, **Playwright** (E2E).

## Có sẵn gì

| Đường dẫn | Vai trò |
| --- | --- |
| `CLAUDE.md` | Sổ tay vận hành cho agent, gồm 2 phần. **Hồ sơ dự án** (mục đích, kiến trúc, lệnh, testing) để `TBD`, agent điền sau khi dev duyệt. **Rule chung** gồm workflow bắt buộc và Definition of Done |
| `.claude/skills/` | Skills cho Claude Code: `brainstorming`, `writing-plans`, `subagent-driven-development`, `react`, `javascript`, `docker`, `playwright`, `git-worktrees`, `commit` |
| `.agent/skills/` | Bộ skills tương tự cho các agent khác |
| `.memory/` | Bộ nhớ chung của agents (bối cảnh, quyết định và lý do), được track trong git |
| `INDEXES.csv` | Chỉ mục ngữ cảnh của codebase. Agent đọc file này trước khi tìm trong cây thư mục |
| `.local/` | Input tạm (PRD, mockup, API docs). Đã gitignore, xoá lúc nào cũng được |

## Bắt đầu nhanh

1. Clone repo và đổi tên theo dự án:

   ```bash
   git clone <repo-url> my-app && cd my-app
   ```

2. Bỏ tài liệu đầu vào (PRD, mockup, API docs…) vào `.local/`.
3. Mở Claude Code tại thư mục gốc, gõ `/brainstorming` kèm yêu cầu (xem prompt mẫu bên dưới).
4. Duyệt từng bước theo [Workflow](#workflow). Agent sẽ dừng lại hỏi khi thiếu thông tin, không tự đoán.

## Prompt mẫu

```text
/brainstorming Khởi tạo web app quản lý đơn hàng nội bộ.
- Tài liệu: .local/prd-order-management.pdf, .local/mockups/
- Người dùng: nhân viên kho (xem, cập nhật trạng thái đơn), admin (quản lý user, xem báo cáo)
- Chức năng MVP: đăng nhập, danh sách đơn có lọc/tìm kiếm, chi tiết đơn, đổi trạng thái
- Ràng buộc: React + TypeScript, API REST có sẵn tại https://api.example.com (docs: .local/api.yaml)
- Ưu tiên: MVP chạy được trong 2 tuần, deploy bằng Docker
- Ngoài phạm vi: thanh toán, app mobile
```

Một prompt tốt cần có: mục tiêu, nguồn tài liệu, vai trò người dùng, phạm vi MVP, ràng buộc kỹ thuật, ưu tiên, và những gì nằm ngoài phạm vi.

## Workflow

Mọi việc không nhỏ (dự án mới, feature, refactor) đều đi qua các bước sau. Chi tiết nằm trong `CLAUDE.md`.

| # | Bước | Output | Người duyệt |
| --- | --- | --- | --- |
| 1 | Brainstorm: đọc tài liệu, đề xuất hướng làm, hỏi những điểm còn mở | Đề xuất trong hội thoại | Dev |
| 2 | Lưu quyết định | `.claude/plans/YYMMDD-<feature>-design.md` | — |
| 3 | Chia phase: mỗi phase giao được và kiểm chứng được độc lập | Danh sách phase | Dev |
| 4 | Plan summary cho phase hiện tại | `docs/plans/YYMMDD-<feature>-p<N>-summary.md` | Dev |
| 5 | Detailed plan | `.claude/plans/YYMMDD-<feature>-p<N>.md` | — |
| 6 | Review plan, xử lý hết finding Critical/Important | Plan đã chỉnh | Reviewer agent |
| 7 | Code theo plan, báo cáo mọi chỗ lệch plan | Code | — |
| 8 | Test và chạy lệnh hẹp nhất; cập nhật summary và `INDEXES.csv` | Exit code, pass/fail | Dev |
| 9 | Close-out: cập nhật rule và memory, báo phase vừa xong và các phase còn lại | `CLAUDE.md`, `.memory/` | Dev |

Nguyên tắc cốt lõi: **evidence first**. Mọi yêu cầu phải truy được về tài liệu trong `docs/` hoặc quyết định của dev. Agent không tự bịa hành vi, nội dung UI, giới hạn hay endpoint.
