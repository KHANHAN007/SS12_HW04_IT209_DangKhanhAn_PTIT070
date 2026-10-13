# Minh chứng SS12 EX04

Workflow: `.github/workflows/deploy-vps.yml` tại gốc repository.

Luồng `build-image` kiểm thử và đóng gói JAR, đẩy image lên GHCR; `deploy-vps` khai báo `needs: build-image`, dùng `appleboy/ssh-action@v1.0.3` để đăng nhập registry, pull image và chạy lại container trên VPS.

Bổ sung ảnh `ssh-deploy.png` sau khi deploy thành công và ảnh `api-health.png` nếu có. Không chụp nội dung private key hoặc token.

Chưa thực hiện triển khai thật; không ghi nhận thành công cho đến khi có log GitHub Actions và kết quả HTTP thực tế.
