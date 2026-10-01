```markdown
# Write-up: Triển khai và Xử lý sự cố Hệ thống Wazuh SIEM (Home Lab)

**Mục tiêu:** Cấu hình hệ thống giám sát an toàn thông tin tập trung sử dụng Wazuh (Manager & Agent) trên môi trường ảo hóa VMware Workstation, thiết lập Giám sát tính toàn vẹn tệp tin (FIM) và phát triển các bộ quy tắc cảnh báo tùy chỉnh (Custom Rules).

**Môi trường:**
- **Nền tảng ảo hóa:** VMware Workstation
- **Hệ điều hành:** Ubuntu Linux, Windows
- **Thành phần cốt lõi:** Wazuh Manager, Wazuh Agent, Sysmon (Windows EventChannel)

---

## Sự cố 1: Wazuh Agent không khởi động được trên Linux (Lỗi cú pháp XML)

**Hiện tượng:**
Sau khi tiến hành cấu hình file để thiết lập tính năng giám sát, tiến trình `wazuh-agent.service` trên máy ảo Ubuntu báo trạng thái "Failed", từ chối khởi động lại. Agent mất kết nối hoàn toàn với Wazuh Manager.

**Quá trình điều tra (Troubleshooting):**
1. Truy xuất log hệ thống của tiến trình bằng lệnh:
   ```bash
   journalctl -xeu wazuh-agent.service

```

2. Kiểm tra thêm log nội bộ của Wazuh Agent tại `/var/ossec/logs/ossec.log` để khoanh vùng nguyên nhân. Log chỉ ra lỗi liên quan đến việc đọc tệp tin cấu hình.
3. Tiến hành rà soát file cấu hình chính tại `/var/ossec/etc/ossec.conf`.

**Nguyên nhân gốc rễ (Root Cause):**
Lỗi cú pháp định dạng XML (Syntax error). Cụ thể, trong quá trình cấu hình khối thẻ `<directories>` để thiết lập File Integrity Monitoring (FIM), một thẻ (tag) đã bị thiếu dấu gạch chéo đóng thẻ (`/`). Việc cấu trúc XML bị gãy khiến trình phân tích cú pháp của Wazuh Agent không thể đọc được file cấu hình và tiến trình tự động "crash" để bảo vệ hệ thống.

**Giải pháp khắc phục:**

1. Mở file `/var/ossec/etc/ossec.conf` bằng trình soạn thảo (`nano` hoặc `vim`), bổ sung dấu gạch chéo đóng thẻ còn thiếu.
2. Khởi động lại dịch vụ Agent thông qua `systemd`:
```bash
sudo systemctl restart wazuh-agent

```


3. Xác nhận lại trạng thái hoạt động:
```bash
sudo systemctl status wazuh-agent

```


*(Trạng thái sẽ chuyển sang `active (running)`).*

---

## Sự cố 2: Nhầm lẫn cơ chế quản lý dịch vụ chéo nền tảng (Cross-Platform)

**Hiện tượng:**
Cố gắng quản lý và kiểm tra trạng thái của Wazuh Agent trên máy ảo Linux bằng các công cụ quản lý dịch vụ quen thuộc trên Windows.

**Nguyên nhân gốc rễ:**
Sự khác biệt về kiến trúc quản lý tiến trình giữa hai hệ điều hành. Trình quản lý dịch vụ của Windows (Services Manager - `services.msc`) không thể giao tiếp trực tiếp hay điều khiển các daemon chạy trên nền tảng Linux (Systemd/SysVinit), kể cả khi chúng nằm chung trong cùng một hệ thống mạng nội bộ ảo hóa.

**Giải pháp khắc phục:**
Định hình lại quy trình quản lý theo từng nền tảng:

* **Đối với Windows Agent:** Sử dụng PowerShell, Command Prompt hoặc `services.msc`.
* **Đối với Ubuntu Agent:** Bắt buộc sử dụng giao diện dòng lệnh (Terminal) với bộ lệnh `systemctl` hoặc tương tác trực tiếp qua giao diện dòng lệnh của Wazuh Manager.

---

## Sự cố 3: Đảm bảo tính toàn vẹn khi phát triển Custom Rule qua Web UI

**Ngữ cảnh:**
Thay vì dùng dòng lệnh, tiến hành viết luật tùy chỉnh trực tiếp trên giao diện **Wazuh Dashboard** (Mục *Management > Rules*, chỉnh sửa file `local_rules.xml`). Mục tiêu là nhận diện hành vi bật tài khoản Guest trái phép trên Windows. Cấu trúc luật sử dụng thẻ `if_sid` trỏ vào Rule gốc `60109` (Windows Security) và lọc các trường dữ liệu bằng biểu thức chính quy PCRE2 regex.

**Rủi ro tiềm ẩn:**
Dù thao tác trên giao diện Web rất tiện lợi, nhưng một lỗi sai nhỏ trong regex hoặc cú pháp XML khi lưu file cũng có thể khiến bộ máy phân tích (`wazuh-analysisd`) trên Wazuh Manager gặp lỗi trong quá trình nạp lại luật (reload), ảnh hưởng đến khả năng xử lý log của toàn bộ hệ thống SIEM.

**Quy trình an toàn (Best Practice) đã áp dụng:**

1. Sử dụng công cụ **Ruleset Test** (được tích hợp sẵn ngay trên giao diện Wazuh Dashboard) để dán chuỗi raw log sự kiện Windows vào. Xác nhận bộ máy phân tích đọc đúng các trường và kích hoạt thành công (trigger) bộ luật tùy chỉnh ở **Phase 3**.
2. Sau khi xác nhận rule hoạt động đúng, tiến hành **Save** file `local_rules.xml` trên Web UI. Giao diện sẽ tự động kiểm tra nhanh cú pháp.
3. Giao diện Wazuh sẽ hiển thị một thông báo màu vàng (Pending restart) ở góc màn hình. Click vào nút **Restart** ngay trên giao diện web để hệ thống tự động khởi động lại dịch vụ Manager một cách an toàn và áp dụng luật mới (thay vì phải SSH vào máy chủ Linux để gõ lệnh `systemctl restart wazuh-manager`).

