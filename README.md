# Xora contribution controller

**Migration manual:** deploy controller này trước notifier, giữ bridge tắt trong lúc thay đổi. App Sync phải có Actions read và installation được admin A chấp nhận. Notifier mẫu đã render cho từng cặp trong `notifiers/`; main/default phải có entrypoint, riêng xora-resolve-be dùng dev để contribute và sync. Các hướng dẫn bootstrap bên dưới chỉ dùng cho repo mới; không chạy lại với repo Xora hiện có.

Developer vào Actions của A, chọn Notify contribution bridge trên default main, nhập PR number và head SHA 40 ký tự. Thêm commit không tự cập nhật B; gửi lại bằng Run workflow mới. Controller kiểm run manual thành công, actor có Write và SHA vẫn đúng; không tin input giả. Xem [hướng dẫn A/B](https://github.com/pho-veteran/github-org-contribution-lab/blob/main/github-app-workflows-ab.md).

B quyết định review/merge. Developer góp code tại A; developer bấm Run workflow với PR + SHA để controller chuyển đúng revision, đồng bộ review/comment và đưa B/main về A/main sau khi B merge. Nhiều cặp repo dùng chung **3 GitHub App và 1 controller**.

**Org A: `XBrainVN`. Org B: `TechX-Corp`.** Registry đã điền 7 cặp repo cùng tên giữa hai org.

Đây là **source public để bàn giao**, chưa kết nối với repo private và chưa bật bridge. Controller vận hành phải đặt trong **repo private do B quản lý** vì log/trạng thái có thể chứa thông tin contribution.

## 1. Lấy code về repo controller private

Tạo repo private rỗng `TechX-Corp/xora-contribution-controller`, không initial README. Chạy trong Bash:

```bash
git clone https://github.com/pho-veteran/xora-contribution-controller.git
cd xora-contribution-controller
git remote rename origin upstream
git remote add origin git@github.com:TechX-Corp/xora-contribution-controller.git
```

Dùng SSH alias bên B thay `github.com` nếu máy đã cấu hình alias. Không cần fork.

## 2. Tạo ba App

Tại org B: **Settings → Developer settings → GitHub Apps → New GitHub App**.

| Tên gợi ý | Only select repositories | Repository permissions |
|---|---|---|
| **Xora Contribution Sync** — App A | 7 repo tại `XBrainVN` | Contents: **Read and write**; Pull requests: **Read and write**; Workflows: **Read and write**; Actions: **Read-only** |
| **Xora Contribution Transport** — App B | 7 repo tại `TechX-Corp` | Contents: **Read and write**; Pull requests: **Read and write** |
| **Xora Contribution Dispatch** | Chỉ `TechX-Corp/xora-contribution-controller` | Actions: **Read and write** |

Cả ba: Metadata **Read-only**, quyền khác **No access**, tắt **Active webhook**. App A chọn **Any account** để admin A cài được; hai App còn lại chọn **Only on this account** nếu tạo tại org B. Tải PEM và ghi Client ID từng App.

Tên App là gợi ý; nếu GitHub báo trùng tên, thêm hậu tố org. Lấy **slug thực tế** từ URL `github.com/apps/<slug>` để điền registry.

## 3. Kiểm tra và push controller

[bridge.config.json](bridge.config.json) đã có đủ 7 repo B:

- `TechX-Corp/xora-evaluation-framework`
- `TechX-Corp/xora-eval-framework-fe`
- `TechX-Corp/xora-learning-loop`
- `TechX-Corp/xora-platform`
- `TechX-Corp/xora-platform-fe`
- `TechX-Corp/xora-resolve-be`
- `TechX-Corp/xora-resolve-agent`

Cấu hình đã điền sẵn, không cần sửa org, mapping hoặc điều kiện workflow:

| File | Giá trị đã cấu hình |
|---|---|
| `bridge.config.json` | Controller `TechX-Corp/xora-contribution-controller`; 7 mapping `XBrainVN/*` → `TechX-Corp/*`; `bot`: `xora-contribution-transport[bot]`; `sourceBot`: `xora-contribution-sync[bot]` |
| `.github/workflows/controller.yml` | Chỉ reconcile khi chạy tại `TechX-Corp/xora-contribution-controller` trên `main` và `BRIDGE_ENABLED=true` |

Chỉ sửa `bot`/`sourceBot` nếu slug App thực tế khác tên đã chuẩn bị. Giữ nguyên scripts, tests, pair IDs, tên secrets và biểu thức `${{ ... }}`. Node.js 24; không cần `npm install`.

```bash
node --check scripts/pr-bridge.mjs
node --test tests/*.test.mjs
git push -u origin main
```

Trong controller private, đặt **Settings → Secrets and variables → Actions**:

| Loại | Tên | Giá trị |
|---|---|---|
| Variable | `SOURCE_APP_CLIENT_ID` | Client ID App A |
| Secret | `SOURCE_APP_PRIVATE_KEY` | PEM App A |
| Variable | `BRIDGE_APP_CLIENT_ID` | Client ID App B |
| Secret | `BRIDGE_APP_PRIVATE_KEY` | PEM App B |
| Variable | `BRIDGE_ENABLED` | `false` trong lúc setup |

Không đưa PEM/token vào Git hoặc repo public này. Push controller chạy job `policy`; bridge chưa reconcile khi chưa bật.

## 4. Bootstrap A và cài notifier

`XBrainVN` đã có 7 repo private **cùng tên repo tại `TechX-Corp`**. Với từng cặp:

1. Review workflow B trước khi sao lịch sử: deploy/CD chỉ chạy tại B; build/test contribution không được nhận secret hay token ghi.
2. Dùng credential admin đã được cấp quyền, fetch/push toàn bộ lịch sử B/main vào A/main. Giữ SHA gốc; không dùng `--mirror`. [Lệnh bootstrap](https://github.com/pho-veteran/github-org-contribution-lab/blob/main/github-app-setup.md#41-khởi-tạo-a-từ-b-có-sẵn).
3. Đặt variable `CONTROLLER_DISPATCH_APP_CLIENT_ID` và secret `CONTROLLER_DISPATCH_APP_PRIVATE_KEY` tại **cả A và B**, lấy từ App Dispatch.
4. Copy [notifier mẫu](https://github.com/pho-veteran/github-org-contribution-lab/blob/main/templates/notifier/.github/workflows/notify-controller.yml) vào `.github/workflows/notify-controller.yml` tại B. Thay `A_OWNER` → `XBrainVN`, `B_OWNER` → `TechX-Corp`, `CONTROLLER_REPO` → `xora-contribution-controller`; `A_REPO` và `B_REPO` là cùng tên repo của cặp đang cấu hình.
5. Mở PR notifier, maintainer B review/merge. Khi kích hoạt, controller đưa cùng workflow từ B/main sang A/main.

App Dispatch chỉ được cài trên controller; private key Dispatch được lưu ở A/B để notifier gọi workflow controller. Key App A/B chỉ ở controller.

### Nếu muốn tự động các bước setup

Dùng [5 script Bash và hướng dẫn](https://github.com/pho-veteran/github-org-contribution-lab/blob/main/github-app-quick-setup.md) từ repo docs. Trong lệnh `01-init.sh`, đặt `--org-a XBrainVN`, `--controller xora-contribution-controller` và `--repos /duong-dan/xora-contribution-controller/repos-b.txt` (thay đường dẫn theo máy). File [repos-b.txt](repos-b.txt) đã chứa đúng 7 URL `TechX-Corp`; phải truyền file này để thay danh sách mặc định cũ của bộ setup.

Chọn cách này **từ đầu với controller rỗng**, để `03-apply.sh` tự sinh/deploy controller, bootstrap và đặt secrets. Không push bản cấu hình thủ công ở mục 3 rồi chạy apply chồng lên: script dừng nếu nội dung controller khác bộ sinh.

## 5. Bảo vệ quyền và kích hoạt

- **A/main:** chỉ App A cập nhật; cho App A direct/force push để reconcile, developer không push/merge.
- **B/main:** maintainer B quyết định; App B không được push/merge main, không bypass protection.
- **Controller/main:** chỉ B quản lý code/workflow/secrets.
- App B có Contents write trên repo; quyền App tự nó không giới hạn branch, cần protection/ruleset phù hợp gói GitHub.
- PR contribution dùng **Create a merge commit** để giữ nguyên SHA/author/committer. Bot ghi yêu cầu trong PR body; không cần khóa phương thức merge tại settings.

Sau khi mọi cặp có main, installation, secrets và notifier đúng; policy pass và protection được kiểm tra: đặt `BRIDGE_ENABLED=true`. Chạy **Actions → Contribution bridge controller → Run workflow → main**, để trống inputs.

Kiểm tra đủ 7 job `Reconcile <id>` thành công, A/main trùng B/main, notifier có ở cả A/B. Sau đó thử PR A → Run workflow PR + SHA → B request changes → A sửa → Run workflow SHA mới → B approve/merge → base A cập nhật và PR A đóng khi revision được chấp nhận.

A chỉ gửi code qua workflow_dispatch. B notifier gửi event khi review/comment/merge hoặc base được push; schedule không gửi code mới. Schedule 6 giờ là dự phòng; thời gian thực tế phụ thuộc hàng đợi Actions. Dispatch Success chưa chứng minh reconcile đã hoàn tất.

## Cấu trúc

| Đường dẫn | Vai trò |
|---|---|
| `.github/workflows/controller.yml` | Policy, chọn cặp repo và chạy reconcile |
| `bridge.config.json` | 7 mapping A/B và danh tính bot |
| `scripts/` | Logic controller dùng chung |
| `tests/` | Policy, comment/review, Git recovery/lease và orchestration |

[Guide đầy đủ](https://github.com/pho-veteran/github-org-contribution-lab/blob/main/github-app-setup.md) mô tả quyền, xử lý lỗi và nghiệm thu. Tests local dùng Git thật/API giả lập; người nhận vẫn cần nghiệm thu quyền và lifecycle trên các repo thực tế.
