# nessus-credentialed-readiness check

Script PowerShell này được thiết kế để kiểm tra các vấn đề phổ biến nhất có thể khiến quá trình credentialed scan của Nessus thất bại.

## Lưu ý
* Phải chạy với quyền quản trị (Administrator) trong PowerShell phiên bản x64.
* Script sẽ từ chối khởi chạy nếu không có quyền Admin hoặc không phải 64-bit.
* Script **không thay đổi** cấu hình hệ thống. Hãy xem kết quả và tự tay chỉnh sửa khi cần.
* Bạn phải truyền tên (username) của (các) tài khoản được phép thực hiện quét Nessus để chạy đánh giá.
    * Nếu user/group được lồng nhau, hãy truyền group cấp cao nhất dự kiến có trên hệ thống đích.
* Script này có thể không phát hiện hết tất cả các vấn đề ngăn chặn việc quét có xác thực, nhưng tập trung vào những lỗi phổ biến nhất. Nếu bạn có đề xuất kiểm tra thêm, vui lòng tạo issue.
* Script có thể đánh dấu một số cấu hình "có vấn đề" mà thực ra không cần chỉnh sửa do cấu hình hệ thống khác bù đắp. Khi biết được những trường hợp này, chúng đã được ghi chú trong phần `info` của từng kiểm tra; hãy đọc kỹ.

## Các Kiểm Tra Có Trong Script
* Yêu cầu User/Group có trong nhóm Local Administrators
* Đảm bảo Remote Shares được bật (Client hoặc Server)
* Các share quản trị (`ADMIN$`, `C$`, `IPC$`) thực sự được publish
* Cổng TCP 135 (RPC endpoint mapper) đang lắng nghe
* Service `Remote Registry` của Windows phải được đặt là `Automatic` hoặc `Manual`
* Service `Server` (LanmanServer) phải được bật
* Service WMI phải được bật
* Người dùng phải xác thực bằng chính tài khoản của mình, không phải Guest
* Kiểm tra Tối Thiểu Cho Windows Firewall
    * Windows Management Instrumentation (DCOM-In)
    * Windows Management Instrumentation (WMI-In)
    * Windows Management Instrumentation (ASync-In)
    * File and Printer Sharing (SMB-In)
* Vấn đề xác thực trên Windows 10 > 1709 / Server 2016 (SPN Validation)
* Symantec Endpoint Protection có thể chặn việc quét
* Kiểm tra UAC Remote Auth Token

## Thông tin thêm
* **Local Admin User/Group** — Xác nhận (các) tài khoản bạn truyền vào tham số `-ScanningAccounts` là thành viên của nhóm `Administrators` cục bộ. Chỉ kiểm tra thành viên trực tiếp; không xử lý các domain group lồng nhau.
* **Remote Shares (registry)** — Đọc giá trị `AutoShareServer` / `AutoShareWks` tại `HKLM\…\LanmanServer\Parameters`. Hai khóa này yêu cầu Windows publish các admin share.
* **Các share quản trị thực sự được publish** — Gọi `Get-SmbShare` để xác nhận `ADMIN$`, `C$`, và `IPC$` thật sự đang tồn tại. Bắt được trường hợp GPO hoặc lệnh `net share /delete` đã xóa chúng dù registry vẫn nói là phải có.
* **Cổng TCP 135** — Gọi `Get-NetTCPConnection` để xác nhận có process đang lắng nghe ở cổng 135. Nessus dùng cổng này cho RPC/WMI; nếu không có gì trả lời, các kiểm tra WMI sẽ thất bại.
* **Các service Remote Registry / Server / WMI** — Kiểm tra start mode của `RemoteRegistry`, `LanmanServer`, và `Winmgmt`. Phải là `Auto` (riêng `Remote Registry` cũng có thể đặt là `Manual`).
* **ForceGuest** — Kiểm tra `HKLM\System\CurrentControlSet\Control\Lsa\ForceGuest = 0`. Nếu đặt thành 1, mọi đăng nhập cục bộ từ xa sẽ bị map thành Guest và Nessus mất quyền admin.
* **Windows Firewall** — Với mỗi profile firewall đang bật, kiểm tra rằng bốn rule inbound nói trên đang được bật và đặt thành Allow.
* **SPN Validation (Windows 10 1709+ / Server 2016)** — Đọc `SMBServerNameHardeningLevel`. Nếu giá trị là 1 hoặc 2 trên OS bị ảnh hưởng, xác thực của Nessus có thể thất bại.
* **Symantec Endpoint Protection** — Tìm trong khóa uninstall 32-bit `Wow6432Node` xem SEP có cài hay không. Nếu có, sẽ cảnh báo vì cấu hình mặc định của SEP có thể chặn quét có xác thực.
* **UAC token (`LocalAccountTokenFilterPolicy`)** — Nếu UAC đang bật và `LocalAccountTokenFilterPolicy` không bằng `1`, các phiên đăng nhập local-admin từ xa sẽ bị giảm xuống token người dùng thường và scan sẽ mất quyền cao.

## Cách Sử Dụng
* Xem trợ giúp (tương tự tài liệu này)
`get-help .\credential_check.ps1 -full`

* Mở PowerShell với quyền quản trị và chạy script `credential_check.ps1` trực tiếp. Kết quả sẽ in ra ngay tại cửa sổ PowerShell.
`.\credential_check.ps1 -ScanningAccounts "vuln_scan"`

* Chạy đánh giá với các tài khoản sau được phép quét lỗ hổng:
    * Local User: "vuln_scan"
    * Domain User: "DOMAIN\vuln_scan"
    * Domain User Group: "DOMAIN\Vuln Scanning Group"

    `.\credential_check.ps1 -ScanningAccounts "vuln_scan","DOMAIN\vuln_scan","DOMAIN\Vuln Scanning Group"`

* Đẩy kết quả ra file để có một báo cáo độc lập.
`.\credential_check.ps1 -ScanningAccounts "vuln_scan" *> Nessus_Credential_Check_Status.txt`

* Mở PowerShell với quyền quản trị và chạy script trên một máy từ xa. Phương án này yêu cầu [Remote PowerShell Management](https://docs.microsoft.com/en-us/windows/win32/winrm/portal) đã được cấu hình và hoạt động.
`Invoke-Command -ComputerName 203.0.113.5 -FilePath .\credential_check.ps1 -ScanningAccounts "vuln_scan"`
