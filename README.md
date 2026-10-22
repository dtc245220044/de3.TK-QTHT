# Quảng bá sản phẩm CPU AMD Ryzen

Website WordPress giới thiệu sản phẩm CPU AMD Ryzen, kèm các dịch vụ dữ liệu, reverse proxy và giám sát hệ thống chạy bằng Docker Compose.

## Dịch vụ

- **WordPress**: nội dung website giới thiệu sản phẩm.
- **MySQL** (`db`): cơ sở dữ liệu WordPress.
- **phpMyAdmin**: quản trị MySQL.
- **Nginx**: reverse proxy HTTPS cho WordPress và endpoint `stub_status` cho exporter.
- **Prometheus**: thu thập và lưu metrics.
- **Grafana**: hiển thị dashboard và quản lý datasource thủ công.
- **Loki**: lưu trữ log.
- **Promtail**: đọc log container Nginx và gửi vào Loki qua Docker socket.
- **cAdvisor**: metrics CPU/RAM container.
- **nginx-exporter**, **mysql-exporter**: exporter metrics cho Nginx và MySQL.

## Điều kiện chạy

- Docker Desktop đang chạy **Linux containers**.
- PowerShell mở tại thư mục dự án:

```powershell
Set-Location C:\Users\MY_PC\source\quang-ba-san-pham
```

- Compose file sử dụng là `docker-compose.yml`.

## Chuẩn bị cấu hình

Sao chép file mẫu rồi điền các giá trị riêng trên máy chạy:

```powershell
Copy-Item .env.example .env
```

`.env` cần có các biến cho MySQL và tài khoản quản trị Grafana theo `.env.example`. Không ghi mật khẩu thật vào README, Git hoặc log.

Một số file bị Git ignore nhưng vẫn cần có khi chạy:

- `nginx/certs/localhost.key`: private key HTTPS. File `localhost.crt` cũng cần đi cùng private key để Nginx phục vụ HTTPS.
- `mysqld-exporter/.my.cnf`: credentials để `mysql-exporter` kết nối MySQL.
- `.env`: các mật khẩu và cấu hình runtime.

Khi chạy lại trên máy đã có dữ liệu, các named volume hiện có sẽ giữ dữ liệu WordPress, MySQL, Prometheus, Grafana, Loki và vị trí đọc log. Khi triển khai máy mới, clone mã nguồn là chưa đủ: phải chuẩn bị `.env`, chứng chỉ/private key và `.my.cnf`; dữ liệu WordPress hiện nằm trong volume, không được chứa đầy đủ trong Git.

## Khởi động và quản lý

Từ thư mục dự án:

```powershell
# Kiểm tra cấu hình
 docker compose config --quiet

# Khởi động
 docker compose up -d

# Xem trạng thái
 docker compose ps

# Xem log một dịch vụ
 docker compose logs -f nginx

# Xem log tất cả dịch vụ
 docker compose logs -f

# Dừng hệ thống nhưng giữ container/volume
 docker compose stop
```

Không dùng `down -v` để dừng thông thường vì lệnh đó có thể xóa dữ liệu trong named volume. Việc tạo lại container hoặc thay đổi image cũng không phải bước cần thiết khi chỉ muốn dừng hệ thống.

## URL truy cập

- Website HTTPS: <https://localhost>
- phpMyAdmin: <http://localhost:8082>
- Prometheus: <http://localhost:9091>
- Grafana: <http://localhost:3001>

Chứng chỉ HTTPS hiện là chứng chỉ tự ký, nên trình duyệt có thể hiển thị cảnh báo bảo mật.

## Grafana: quản lý thủ công

Grafana không còn provisioning datasource/dashboard từ file Compose. Sau khi đăng nhập Grafana:

1. Thêm datasource **Prometheus** với URL nội bộ:
   `http://prometheus:9090`
2. Thêm datasource **Loki** với URL nội bộ:
   `http://loki:3100`
3. Import file:
   `grafana/dashboards/tong-quan-he-thong.json`.

Dashboard hiện tham chiếu datasource Prometheus bằng UID:

```text
cg2322d0mz668f
```

UID datasource Prometheus phải khớp UID này trước khi import hoặc sử dụng dashboard. Datasource Loki không được dashboard này tham chiếu.

## Dữ liệu và lưu trữ

Compose sử dụng các named volume cho dữ liệu:

- `db_data`: dữ liệu MySQL.
- `wordpress_data`: toàn bộ mã nguồn/nội dung WordPress trong container.
- `prometheus_data`: dữ liệu metrics Prometheus.
- `grafana_data`: dữ liệu Grafana.
- `loki_data`: dữ liệu Loki.
- `promtail_positions`: vị trí đọc log của Promtail.

Tên thực tế trong project có tiền tố `quang-ba-san-pham_`. Nội dung WordPress không nằm đầy đủ trong Git; cần bảo toàn volume hoặc có quy trình backup/restore riêng khi chuyển máy.

## Giới hạn hiện có

- HTTPS dùng chứng chỉ tự ký, chỉ phù hợp cho môi trường nội bộ/phát triển.
- Dashboard cAdvisor chọn container theo **container ID**; khi recreate container, cần chọn lại ID trong biến dashboard nếu danh sách thay đổi.
- Promtail đã hết giai đoạn hỗ trợ upstream/deprecated; cấu hình hiện tại vẫn dùng Promtail để thu thập log Docker, nhưng nên có kế hoạch chuyển sang thành phần được hỗ trợ khi bảo trì hệ thống.


## Bàn giao phần riêng của WordPress

Thư mục `website/` chứa bản xuất hiện tại, không chứa tài khoản, password hash, session, token, secret, `wp-config.php` hoặc nguyên database:

- `website/content.xml`: WXR 1.2 cho 9 trang publish và metadata block của navigation/global styles. Có thể dùng WordPress Importer cho phần post type `page`; các post type block không được giả định là Importer mặc định xử lý đầy đủ.
- `website/block-content.json`: nội dung block của `wp_navigation` và `wp_global_styles`, gồm CSS/global styles và tham chiếu ID nguồn. Đây là file trung gian có cấu trúc mô tả, **không phải** file có thể import trực tiếp bằng công cụ Import mặc định của WordPress.
- `website/metadata.json`: tên/mô tả website, kiểu trang chủ, permalink, theme và ID nguồn của trang chủ/navigation/global styles.
- `website/uploads/`: 49 ảnh/media từ Media Library, khoảng 508 KB.
- `website/uploads-manifest.json`: đường dẫn tương đối và kích thước từng file media.

### Khôi phục thủ công trên máy mới

1. Cài/khởi chạy WordPress với theme **Twenty Twenty-Four 1.3** hoặc phiên bản tương thích đã kiểm tra.
2. Dùng WordPress Importer để nhập `website/content.xml` cho các trang nếu môi trường đã có Importer. Kiểm tra kết quả từng trang, vì WXR này không bao gồm database, user hoặc toàn bộ options.
3. Đưa media trong `website/uploads/` vào Media Library bằng quy trình media/import phù hợp, sau đó cập nhật các tham chiếu media trong nội dung nếu URL hoặc attachment ID thay đổi. `source_id` và đường dẫn năm/tháng chỉ là thông tin đối chiếu, không bảo đảm ID mới giống máy nguồn.
4. Tạo/điều chỉnh trang chủ tĩnh theo `metadata.json`. ID trang mới không nhất thiết là `6`.
5. Tạo menu/navigation mới rồi nhập lại nội dung navigation từ `block-content.json`, thay các liên kết `page_id` và URL bằng ID/URL của các trang mới. ID navigation nguồn `5` không được giả định giữ nguyên.
6. Khôi phục global styles bằng giao diện Site Editor hoặc một importer riêng có hỗ trợ `wp_global_styles`; đối chiếu nội dung `styles.css`, màu nền và palette từ `block-content.json`. Không dán JSON tùy ý vào công cụ Import mặc định.
7. Đặt permalink theo `metadata.json`, kiểm tra lại URL nội bộ và URL media sau khi đổi domain/host.
8. Kiểm tra trực quan trang chủ, các trang sản phẩm, menu, CSS nền tím/hiệu ứng sao và media.

Bộ xuất này đã được kiểm tra cú pháp và số lượng file, nhưng chưa được nhập thử trên máy mới; chưa thể khẳng định khôi phục thành công. Việc nhập đầy đủ navigation/global styles có thể cần thao tác Site Editor hoặc một công cụ/importer chuyên dụng. Không tự tạo hoặc thêm công cụ đó trong repository.
