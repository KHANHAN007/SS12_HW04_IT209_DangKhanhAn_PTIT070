# SS12 EX04 - Triển khai Spring Boot lên VPS qua SSH

- Sinh viên: Đặng Khánh An - PTIT070.
- Project và repository riêng cho SS12 EX04.
- Luồng: `build-image` → `deploy-vps`.
- API kiểm tra: `/health` và `/` (không sử dụng Actuator).

## 1. Chuẩn bị VPS

VPS phải có Docker Engine và curl. Kiểm tra:

```bash
docker version
curl --version
```

Tài khoản SSH phải có quyền chạy Docker. Nếu dùng `devops`, quản trị viên thêm vào nhóm Docker:

```bash
sudo usermod -aG docker devops
```

Thoát SSH và đăng nhập lại, rồi kiểm tra `docker ps` không cần sudo. Nhóm Docker có quyền quản trị máy chủ; chỉ cấp cho tài khoản triển khai tin cậy.

Kiểm tra cổng trước khi triển khai:

```bash
sudo ss -tlnp | grep ':8080' || true
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Nếu cổng 8080 đã có ứng dụng, xác định và dừng đúng ứng dụng của bạn trước. Workflow thay thế container tên `app`; không sử dụng tên này nếu nó đang thuộc ứng dụng khác cần giữ lại.

Nếu VPS dùng UFW:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp
sudo ufw status verbose
```

Mở thêm cổng 8080 ở Cloud Firewall nếu có. Không reset firewall hoặc xóa cấu hình Nginx của bài cũ.

## 2. Tạo SSH key riêng trên Mac

```bash
ssh-keygen -t ed25519 -f ~/.ssh/it209_ss12_ex04 -C "it209-ss12-ex04"
```

Với key dùng riêng cho bài tập, để trống passphrase vì workflow này chưa khai báo secret passphrase. Giữ file private key trên Mac, tuyệt đối không commit vào repository.

Cài public key vào đúng tài khoản VPS, ví dụ:

```bash
cat ~/.ssh/it209_ss12_ex04.pub | ssh devops@160.187.229.99 'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys'
```

Kiểm tra đăng nhập bằng key:

```bash
ssh -i ~/.ssh/it209_ss12_ex04 -o IdentitiesOnly=yes devops@160.187.229.99 'whoami; docker ps'
```

## 3. Tạo repository riêng

Chạy trên Mac trong thư mục project này:

```bash
git init
git branch -M main
git config user.name "KHANHAN007"
git config user.email "khanhanke@gmail.com"
git add .
git commit -m "init: SS12 EX04 build and SSH deployment"
gh auth switch -u KHANHAN007
gh repo create KHANHAN007/SS12_HW04_IT209_DangKhanhAn_PTIT070 --public --source . --remote origin
```

Chưa push cho đến khi Repository Secrets đã được cấu hình. Image tự lấy tên repository và chuyển thành chữ thường, ví dụ:

```text
ghcr.io/khanhan007/ss12_hw04_it209_dangkhanhan_ptit070:latest
```

## 4. Repository Secrets

Tại repository mới, mở Settings → Secrets and variables → Actions.

| Secret | Giá trị |
|---|---|
| `SSH_HOST` | IP VPS, ví dụ `160.187.229.99` |
| `SSH_USER` | Tài khoản SSH, ví dụ `devops` |
| `SSH_KEY` | Toàn bộ nội dung private key `~/.ssh/it209_ss12_ex04` |
| `REGISTRY_TOKEN` | GitHub PAT classic có quyền `read:packages` |

Copy private key vào clipboard trên Mac để dán trực tiếp vào Secret:

```bash
pbcopy < ~/.ssh/it209_ss12_ex04
```

Không dùng file `.pub` làm `SSH_KEY`. Token phải thuộc tài khoản có quyền đọc package. Có thể dùng lại PAT của bài trước nếu còn hạn và có quyền đọc package, nhưng phải thêm Secret vào repository mới.

Job build dùng `GITHUB_TOKEN` do GitHub tự cấp với `packages: write`; job deploy dùng PAT để VPS kéo package private.

## 5. Push và theo dõi deploy

```bash
git push -u origin main
```

Vào Actions → Build Image and Deploy to VPS. Kiểm tra `build-image` xanh rồi `deploy-vps` xanh. Trong bước Deploy over SSH, cần thấy đăng nhập GHCR thành công, pull image, container chạy và response health.

Workflow tạo image cho `linux/amd64`, đúng kiến trúc VPS đang triển khai. Không build thêm arm64 vì base image Alpine Java 17 dùng ở bài này không hỗ trợ kiến trúc đó. Nếu test/build lỗi thì deploy tự bị bỏ qua.

## 6. Kiểm tra ứng dụng

Trên Mac:

```bash
curl -i http://160.187.229.99:8080/health
```

Hoặc mở trình duyệt tại địa chỉ đó. Cần HTTP 200 và JSON `status: UP`.

Trên VPS nếu cần chẩn đoán:

```bash
docker ps --filter name=app
docker logs --tail 100 app
curl -i http://127.0.0.1:8080/health
```

## 7. Minh chứng nộp bài

Đặt ảnh log SSH triển khai thành công tại `homework/session_12/ex4/ssh-deploy.png`. Có thể thêm `api-health.png` chụp HTTP 200 trên trình duyệt.

```bash
git add homework/session_12/ex4
git commit -m "docs: add VPS deployment evidence"
git push
```

Workflow phải ở `.github/workflows/deploy-vps.yml` tại gốc repository. Chưa có kết quả triển khai VPS cho project này cho đến khi bạn cấu hình secrets và push.
