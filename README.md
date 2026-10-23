# Đề số 3 — Website quảng bá CPU AMD Ryzen

## 1. Giới thiệu

Đây là Đề số 3: website WordPress quảng bá CPU AMD Ryzen, kèm cơ sở dữ liệu, reverse proxy HTTPS và hệ thống quan sát chạy bằng Docker Compose.

| Service | Vai trò |
|---|---|
| `wordpress` | Website và trang quản trị WordPress. |
| `db` | MySQL 8.0 lưu dữ liệu WordPress. |
| `phpmyadmin` | Giao diện quản trị MySQL. |
| `nginx` | Reverse proxy HTTPS và endpoint `stub_status`. |
| `nginx-exporter` | Chuyển metrics Nginx thành metrics Prometheus. |
| `mysql-exporter` | Chuyển metrics MySQL thành metrics Prometheus. |
| `prometheus` | Thu thập và lưu metrics trong 7 ngày/1 GB. |
| `grafana` | Hiển thị dashboard metrics và log. |
| `loki` | Lưu log tập trung trong mạng Docker. |
| `promtail` | Đọc log container Nginx qua Docker socket và gửi vào Loki. |
| `cadvisor` | Thu thập metrics CPU/RAM của container. |

## 2. Điều kiện chạy

Cần có trên Windows:

- Docker Desktop đang chạy **Linux containers**.

- Git, Docker Compose (`docker compose`) và PowerShell.

- Các cổng host phải trống: `80`, `443`, `8082`, `9091`, `3001`.

Không cần mở terminal Ubuntu/WSL và không cần cài MySQL, Nginx, WordPress, Prometheus hay Grafana trực tiếp lên Windows. Compose chỉ bind các cổng publish vào `127.0.0.1`.

Có hai tình huống khác nhau:

- **Máy mới:** phải tạo `.env`, chứng chỉ HTTPS, file `.my.cnf`, database và tài khoản exporter theo thứ tự ở mục 4.

- **Máy đã có dữ liệu:** chỉ cần Docker Desktop chạy, các file runtime còn nguyên và dùng `docker compose up -d`; không tạo lại tài khoản hay xóa named volumes.

## 3. Clone repository

Chọn bất kỳ thư mục lưu nào trên Windows rồi mở PowerShell:

```powershell
git clone https://github.com/dtc245220044/de3.TK-QTHT.git
Set-Location .\de3.TK-QTHT
```

Nếu muốn tên thư mục khác, chỉ định tên đó trong lệnh clone, ví dụ `git clone https://github.com/dtc245220044/de3.TK-QTHT.git ten-thu-muc`, rồi chạy `Set-Location .\ten-thu-muc`. Mọi lệnh còn lại phải chạy tại thư mục chứa `docker-compose.yml`.

## 4. Triển khai lần đầu trên máy mới

### 4.1. Tạo `.env` mà không ghi đè file hiện có

```powershell
if (-not (Test-Path -LiteralPath '.env')) {
    Copy-Item -LiteralPath '.env.example' -Destination '.env'
    Write-Host 'Đã tạo .env. Hãy điền giá trị trước khi tiếp tục.'
} else {
    Write-Host '.env đã tồn tại; không ghi đè.'
}
code .env
```

Điền các biến có trong `.env.example`:

- `MYSQL_DATABASE`: tên database WordPress.

- `MYSQL_USER`, `MYSQL_PASSWORD`: user và mật khẩu ứng dụng WordPress.

- `MYSQL_ROOT_PASSWORD`: mật khẩu quản trị MySQL.

- `GRAFANA_ADMIN_USER`, `GRAFANA_ADMIN_PASSWORD`: tài khoản khởi tạo Grafana.

Dùng mật khẩu ngẫu nhiên mạnh, không dùng placeholder hoặc mật khẩu mẫu yếu. Để đơn giản hóa việc nhập trong `.env`, SQL và `.my.cnf`, có thể dùng mật khẩu riêng gồm ít nhất 24 ký tự chữ hoa, chữ thường và số. Giữ `MYSQL_DATABASE=wordpress` và `MYSQL_USER=wp_user` nếu không chủ động điều chỉnh các bước hướng dẫn liên quan. Không đưa secret vào README, Git, tham số dòng lệnh hay log. Nếu VS Code chưa có lệnh `code`, mở `.env` bằng VS Code từ giao diện; không cần cài thêm công cụ cho Compose.

### 4.2. Tạo cặp chứng chỉ localhost

Nginx mount `./nginx/certs:/etc/nginx/certs:ro` và cần đúng hai file `nginx/certs/localhost.crt`, `nginx/certs/localhost.key`. Chứng chỉ phải có SAN `DNS:localhost,IP:127.0.0.1`. Đoạn sau dùng container tạm có OpenSSL, không giả định Windows đã cài OpenSSL và ghi file trực tiếp qua bind mount:

```powershell
New-Item -ItemType Directory -Force -Path .\nginx\certs | Out-Null
$crt = Test-Path -LiteralPath .\nginx\certs\localhost.crt
$key = Test-Path -LiteralPath .\nginx\certs\localhost.key
if ($crt -and $key) {
    Write-Host 'Đã có đủ chứng chỉ và private key; giữ nguyên.'
} else {
    if ($crt -or $key) {
        Write-Warning 'Chỉ có một phần cặp chứng chỉ. Cặp không đầy đủ sẽ được tạo lại.'
        Remove-Item -Force .\nginx\certs\localhost.crt, .\nginx\certs\localhost.key -ErrorAction SilentlyContinue
    }
    $certs = (Resolve-Path .\nginx\certs).Path
    docker run --rm --mount "type=bind,source=$certs,target=/certs" alpine:3.20 sh -c 'apk add --no-cache openssl >/dev/null && openssl req -x509 -nodes -newkey rsa:2048 -days 825 -keyout /certs/localhost.key -out /certs/localhost.crt -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"'
    if ($LASTEXITCODE -ne 0) { throw 'Tạo chứng chỉ thất bại; kiểm tra lỗi Docker/OpenSSL trước khi tiếp tục.' }
}
```

Không truyền dữ liệu nhị phân qua PowerShell. Chứng chỉ này tự ký, vì vậy trình duyệt sẽ cảnh báo khi mở HTTPS.

### 4.3. Tạo file cấu hình MySQL exporter trước khi khởi động exporter

Compose mount `./mysqld-exporter/.my.cnf:/cfg/.my.cnf:ro` và chạy exporter với user container `1000:1000`. Tạo file bằng VS Code để tránh lỗi encoding hoặc vô tình tạo thư mục:

```powershell
New-Item -ItemType Directory -Force -Path .\mysqld-exporter | Out-Null
code .\mysqld-exporter\.my.cnf
```

Dán đúng nội dung sau, thay `<MAT_KHAU_EXPORTER>` bằng mật khẩu riêng sẽ dùng khi tạo MySQL user; không ghi mật khẩu thật vào README:

```ini
[client]
user=exporter
password=<MAT_KHAU_EXPORTER>
host=db
port=3306
```

Trong VS Code, lưu file dạng UTF-8 (không BOM nếu có lựa chọn), dùng LF hoặc CRLF đều được, và kiểm tra đường dẫn là file `mysqld-exporter/.my.cnf`. Không dùng root trong file này và không commit/chia sẻ file.

### 4.4. Khởi động riêng MySQL và chờ healthy

```powershell
docker compose up -d db
if ($LASTEXITCODE -ne 0) { throw 'Không khởi động được MySQL.' }
docker compose ps db
```

Chờ trạng thái `db` thành `healthy`; chạy lại `docker compose ps db` sau vài giây để kiểm tra. Nếu vẫn chưa healthy sau khoảng 2 phút, xem `docker compose logs --tail 100 db` và xử lý lỗi trước khi tiếp tục. Không chạy vòng chờ vô hạn.

### 4.5. Tạo user exporter bằng prompt mật khẩu tương tác

Sau khi `db` healthy, vào MySQL bằng root với prompt nhập mật khẩu; không đặt mật khẩu root trên command line:

```powershell
docker compose exec -e MYSQL_HISTFILE=/dev/null db mysql -uroot -p
```

Khi MySQL hỏi `Enter password:`, nhập `MYSQL_ROOT_PASSWORD` trong `.env`. Tại dấu nhắc `mysql>`, chạy SQL sau và thay `<MAT_KHAU_EXPORTER>` bằng **đúng mật khẩu đã ghi trong `.my.cnf`**. Dấu `<...>` chỉ là placeholder, không gõ nguyên văn:

```sql
CREATE USER 'exporter'@'%' IDENTIFIED BY '<MAT_KHAU_EXPORTER>' WITH MAX_USER_CONNECTIONS 3;
GRANT SELECT, PROCESS, REPLICATION CLIENT ON *.* TO 'exporter'@'%';
EXIT;
```

Bước SQL chỉ dùng khi triển khai mới. Nếu báo user đã tồn tại, không tự đổi mật khẩu; kiểm tra cấu hình đang dùng. `MYSQL_HISTFILE=/dev/null` tránh lưu lịch sử MySQL client; mật khẩu trong câu SQL vẫn có thể hiện trên terminal, vì vậy không chụp hoặc chia sẻ màn hình lúc nhập. Các lệnh `CREATE USER`/`GRANT` có hiệu lực ngay, không cần `FLUSH PRIVILEGES`.

Tài khoản exporter không dùng root; quyền và `MAX_USER_CONNECTIONS=3` khớp yêu cầu của exporter. Khi chạy lại hệ thống đã có database/user, **không** chạy lại bước tạo user, reset mật khẩu hay cấp lại quyền nếu không có lý do vận hành riêng.

### 4.6. Validate rồi khởi động toàn bộ hệ thống

```powershell
docker compose config --quiet
if ($LASTEXITCODE -ne 0) { throw 'Compose không hợp lệ; chưa khởi động hệ thống.' }
if (-not (Test-Path -LiteralPath .\nginx\certs\localhost.crt -PathType Leaf)) { throw 'Thiếu nginx/certs/localhost.crt' }
if (-not (Test-Path -LiteralPath .\nginx\certs\localhost.key -PathType Leaf)) { throw 'Thiếu nginx/certs/localhost.key' }
if (-not (Test-Path -LiteralPath .\mysqld-exporter\.my.cnf -PathType Leaf)) { throw 'Thiếu mysqld-exporter/.my.cnf' }
docker compose up -d
if ($LASTEXITCODE -ne 0) { throw 'Khởi động chưa thành công; xem lỗi trước khi tiếp tục.' }
docker compose ps
```

Không dùng `docker compose down -v` hoặc `docker system prune` trong quy trình này. Không tạo service/file runtime nào ngoài `.env`, cặp chứng chỉ và `.my.cnf` mà Compose đã yêu cầu.

## 5. Website WordPress

### Cài mới

Mở `https://localhost`, hoàn tất trình cài WordPress và tạo tài khoản quản trị. Cách này khởi tạo dữ liệu site trong MySQL (`db_data`), đồng thời sử dụng file WordPress trong `wordpress_data`, không tự dùng nội dung trong thư mục `website/`.

### Dùng bộ `website/` hiện có

Bộ xuất gồm:

- `content.xml`: WXR 1.2 với 9 trang publish và dữ liệu block/navigation/global styles.

- `block-content.json`: dữ liệu `wp_navigation` và `wp_global_styles`, gồm CSS/global styles và ID nguồn.

- `metadata.json`: tên, mô tả, permalink, theme, kiểu trang chủ và ID nguồn.

- `uploads-manifest.json` cùng `uploads/2026/10/`: 49 file ảnh/media.

Kiểm kê cho biết theme là **Twenty Twenty-Four 1.3** và không có plugin active được xuất trong bộ này. XML hiện **không có attachment records/metadata ảnh**. Chép `uploads/` vào thư mục upload không tự đăng ký ảnh trong Media Library. Không thể hứa import XML sẽ phục hồi đủ ảnh, menu, global styles, CSS hoặc các liên kết ID.

Quy trình thủ công khả dụng:

1. Vào **Appearance → Themes**, chọn Twenty Twenty-Four và kiểm tra phiên bản. Website gốc dùng 1.3. Nếu image có phiên bản khác, không mặc định giao diện sẽ giống hoàn toàn; dùng gói theme đúng phiên bản từ nguồn WordPress chính thức khi cần đối chiếu.

2. Cài và dùng WordPress Importer để nhập `website/content.xml` cho các trang; kiểm tra từng trang sau nhập. XML không chứa nguyên database, user hoặc toàn bộ options.

3. Tải các ảnh gốc cần dùng từ `website/uploads/` lên Media Library bằng giao diện WordPress, sau đó chọn lại ảnh trong các block/trang. 49 file có thể bao gồm các ảnh kích thước nhỏ WordPress tạo tự động; không mặc định tải tất cả thành 49 attachment riêng. Attachment ID mới có thể khác ID nguồn.

4. Đặt trang chủ tĩnh trong **Settings → Reading** theo `metadata.json`; không giả định ID trang nguồn còn đúng.

5. Dựng lại menu/navigation trong Site Editor. Dùng `block-content.json` để tham khảo, nhưng thay `page_id` và URL bằng trang mới.

6. CSS nằm trong dữ liệu global styles đã xuất. Mở `website/block-content.json`, tìm trường/nội dung `styles.css` trong phần `wp_global_styles`, lấy giá trị CSS đã giải mã JSON (không sao chép cả JSON hoặc các ký tự escape như `\n`, `\"`), rồi dán vào **Additional CSS** của Site Editor nếu giao diện WordPress cung cấp mục đó.

7. Sửa liên kết nội bộ và URL ảnh sau khi ID/URL thay đổi; kiểm tra trang chủ, trang sản phẩm, menu, ảnh và CSS.

Chưa kiểm thử khôi phục đầy đủ trên máy mới. Đây là quy trình dựng lại thủ công, không phải cam kết phục hồi nguyên trạng tự động.

## 6. Địa chỉ truy cập

| Thành phần | Địa chỉ |
|---|---|
| Website | <https://localhost> |
| WordPress admin | <https://localhost/wp-admin> |
| phpMyAdmin | <http://localhost:8082> |
| Prometheus | <http://localhost:9091> |
| Grafana | <http://localhost:3001> |
| Loki | Chỉ nội bộ: `http://loki:3100` |
| Exporter | Chỉ nội bộ trong mạng Docker |

Loki không publish `3100` ra Windows; không dùng `http://localhost:3100` để kiểm tra.

## 7. Grafana thủ công

Đăng nhập tại `http://localhost:3001` bằng `GRAFANA_ADMIN_USER` và `GRAFANA_ADMIN_PASSWORD` trong `.env` lúc volume Grafana được khởi tạo lần đầu.

1. Vào **Connections → Data sources → Add data source → Prometheus**, đặt URL `http://prometheus:9090`, chọn làm **Default** và bấm **Save & test**.

2. Thêm datasource **Loki**, đặt URL `http://loki:3100`, bấm **Save & test**.

3. Kiểm tra và điều chỉnh UID như đoạn dưới **trước khi import**. Sau đó vào **Dashboards → New → Import → Upload dashboard JSON file**, chọn `grafana/dashboards/tong-quan-he-thong.json` và bấm **Import**.

Dashboard JSON hiện tham chiếu UID Prometheus `cg2322d0mz668f`. UID là định danh nội bộ, không coi giá trị này là mặc định trên máy khác. Nếu datasource mới có UID khác, vào trang datasource Prometheus để xem UID (trong URL/trang cấu hình), rồi trước khi import tìm đúng chuỗi UID cũ trong JSON và thay các tham chiếu `datasource.uid` thuộc Prometheus, gồm cả datasource của biến `container_id`. Không thay các giá trị không phải tham chiếu datasource và không gán UID Prometheus cho Loki. Có thể tạo datasource mới với UID cũ nếu phiên bản Grafana cho phép, nhưng vẫn phải kiểm tra Save & test.

Biến dashboard cAdvisor chọn **container ID** từ metric `id=~"/docker/.+"`, không phải tên service. Vì cấu hình cAdvisor không lọc Compose labels, lựa chọn `All` có thể gồm container ngoài project; nên chọn ID cần xem. Khi recreate container, ID có thể đổi và phải chọn lại. cAdvisor chạy `privileged`, dùng các mount host và Docker socket theo Compose; đây là giới hạn/quyền mở rộng có chủ ý của cấu hình hiện tại.

### LogQL

Các log Promtail hiện gắn selector `project="quang-ba-san-pham"`, `service="nginx"`:

```logql
{project="quang-ba-san-pham", service="nginx"}
{project="quang-ba-san-pham", service="nginx"} |= "GET"
sum(count_over_time({project="quang-ba-san-pham", service="nginx"}[5m]))
```

Truy vấn thứ nhất xem log Nginx, thứ hai lọc các dòng có request GET, thứ ba đếm tổng **dòng log** của các stream khớp selector trong 5 phút. Số dòng log không luôn bằng số request: một request có thể sinh nhiều dòng, log có thể bị gộp hoặc một dòng có thể đại diện cho hoạt động khác.

## 8. Chạy lại và dừng

Từ thư mục chứa `docker-compose.yml`:

```powershell
# Khởi động hoặc chạy lại, giữ named volumes
docker compose up -d
# Trạng thái
docker compose ps
# Log giới hạn số dòng
docker compose logs --tail 100
docker compose logs --tail 100 nginx
# Dừng container nhưng giữ dữ liệu
docker compose stop
```

Không cần tạo lại `.env`, chứng chỉ hoặc user exporter mỗi lần chạy lại. Dữ liệu nằm trong named volumes `db_data`, `wordpress_data`, `prometheus_data`, `grafana_data`, `loki_data`, `promtail_positions` (Docker có thể hiển thị tên kèm tiền tố project). Đổi mật khẩu trong `.env` không tự đổi mật khẩu MySQL/Grafana đã lưu khi volume đã khởi tạo; phải đổi theo quy trình riêng của từng dịch vụ. Không dùng `down -v` để dừng thông thường vì có thể xóa dữ liệu.

## 9. Hardening và giới hạn

- Compose tách các mạng `app`, `database`, `monitoring`, `logging`; các service chỉ nối vào mạng cần thiết. Các cổng publish đều loopback, không mở trực tiếp ra LAN.

- WordPress chỉ truy cập MySQL qua mạng `database`; exporter dùng user riêng với `SELECT`, `PROCESS`, `REPLICATION CLIENT` và giới hạn 3 kết nối.

- `mysql-exporter` chạy với user container `1000:1000`; không khẳng định toàn bộ container đều non-root vì cấu hình không đặt cùng chính sách cho mọi service.

- `cadvisor` dùng `privileged: true`, `cgroup: host`, các mount hệ thống và Docker socket để lấy metrics; đây là ngoại lệ quyền cao.

- Promtail mount Docker socket với `:ro` để discovery và đọc log. `:ro` không biến Docker API thành API chỉ đọc; quyền truy cập socket vẫn nhạy cảm.

- Chứng chỉ HTTPS tự ký chỉ phù hợp phát triển/nội bộ. Promtail đã hết hỗ trợ upstream/deprecated; cấu hình hiện tại vẫn dùng nó.

- cAdvisor không giới hạn danh sách theo project labels trong dashboard, nên metrics có thể gồm container ngoài project.

## 10. Xử lý lỗi ngắn gọn

- **Port bị chiếm:** kiểm tra cổng `80`, `443`, `8082`, `9091`, `3001` bằng Docker Desktop hoặc `Get-NetTCPConnection`; xác định ứng dụng đang chiếm cổng. Chỉ dừng khi đó là ứng dụng bạn có thể dừng; nếu cần đổi cổng host thì cập nhật Compose và URL truy cập tương ứng. Không tự dừng project khác.

- **Bind mount thiếu/sai loại:** kiểm tra `Test-Path -PathType Leaf` cho hai file chứng chỉ và `.my.cnf`. Nếu có thư mục mang tên file, dừng và sửa thủ công; không khởi động exporter trước khi `.my.cnf` là file.

- **DB authentication/exporter down:** xác nhận db `healthy`, `host=db`, user là `exporter`, mật khẩu trong `.my.cnf` khớp user và không dùng root. Xem `docker compose logs --tail 100 db mysql-exporter`; khi chạy lại dữ liệu hiện có không tự cấp lại quyền/reset user.

- **Datasource/dashboard lỗi:** kiểm tra URL nội bộ, **Save & test**, rồi đối chiếu UID Prometheus trong JSON với UID datasource mới. UID trên máy phát triển không phải mặc định.

- **Loki chưa sẵn sàng:** xem `docker compose logs --tail 100 loki promtail`. Kiểm tra từ Grafana hoặc trong Docker network, ví dụ `docker compose exec grafana wget -qO- http://loki:3100/ready` nếu image Grafana có `wget` (nếu không có, dùng **Save & test** của datasource Loki; phép thử đó kiểm tra kết nối datasource, không thay thế chính xác endpoint `/ready`); không dùng `localhost:3100` trên Windows.

- **LogQL không có dữ liệu:** kiểm tra Promtail đang chạy và dùng đúng selector `project="quang-ba-san-pham", service="nginx"`; log mới có thể cần thời gian được thu thập.

- **HTTPS tự ký:** dùng `curl.exe -k` khi kiểm thử, để bỏ kiểm tra chứng chỉ riêng cho request thử; không dùng `-k` như cách xác minh chứng chỉ hợp lệ:

```powershell
curl.exe -k -I https://localhost
curl.exe -k -I https://localhost/wp-admin
```

Không xóa named volumes như cách sửa lỗi mặc định; việc đó có thể làm mất dữ liệu WordPress, MySQL, Grafana, Prometheus hoặc Loki.

Quy trình triển khai và khôi phục trong README chưa được chạy thử đầy đủ trên một máy Windows mới. Bộ xuất website hỗ trợ dựng lại thủ công; chưa xác nhận khôi phục nguyên trạng tự động.
