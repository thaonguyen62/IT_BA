# Tài liệu Phân tích Nghiệp vụ (BA Specification)
## Xuất & Điều chỉnh Hóa đơn VAT — Hotel Booking (OMH)

| |                                                                                   |
|---|-----------------------------------------------------------------------------------|
| **Phiên bản** | 1.0                                                                               |
| **Ngày** | 16/07/2026                                                                        |
| **Người soạn** | Thảo Nguyên                                                                             |
| **Trạng thái** | Draft                                                                             |
| **Nguồn tham chiếu** | `index.html` (Invoice Lifecycle — Hotel Booking), BRD OMH, Confluence VAT ISSUING |

---

## 1. Mục đích & Phạm vi

### 1.1. Mục đích
Mô tả yêu cầu nghiệp vụ và luồng xử lý cho việc **tạo hóa đơn VAT** và **điều chỉnh/hủy hóa đơn VAT** đối với booking khách sạn qua supplier OMH, tích hợp giữa Platform Service, ERP, VACOM và các hệ thống liên quan.

### 1.2. Bối cảnh nghiệp vụ (Context)
- **Expedia:** MoMo đóng vai trò **forwarder** (trung gian) — không trực tiếp quyết định giá bán → **không phải bên xuất hóa đơn**.
- **OMH:** MoMo nhận **giá B2B** rồi cộng **markup**, trực tiếp quyết định giá bán → MoMo là **bên bán (principal)** → **bắt buộc xuất hóa đơn VAT**.
- Luồng xuất hóa đơn phía **Platform (B2C)** đã và đang triển khai. Phạm vi phần Hotel chỉ implement việc **truyền thông tin hóa đơn** sang Platform.

### 1.3. Trong phạm vi (In-scope)
- Thu thập thông tin hóa đơn khi đặt phòng (in-app) hoặc qua CS (CRM).
- Truyền thông tin hóa đơn từ Hotel/CRM sang Platform Service.
- Luồng xuất hóa đơn: Platform → ERP → VACOM → AF (email) → thông báo user.
- Luồng hủy booking → xử lý tracking hóa đơn theo trạng thái → hóa đơn điều chỉnh.
- Đồng bộ trạng thái hóa đơn về CRM (Mobio).

### 1.4. Ngoài phạm vi (Out-of-scope)
- Luồng xuất hóa đơn cho supplier Expedia (không xuất HĐ).
- Nghiệp vụ nội bộ VACOM (đơn vị phát hành hóa đơn điện tử).
- Chính sách hoàn tiền/charge của Partner (chỉ tiêu thụ kết quả).

---

## 2. Actor & Hệ thống liên quan

| Actor / Hệ thống | Vai trò |
|---|---|
| **Khách hàng (User)** | Đặt phòng, yêu cầu xuất HĐ (tick + điền info), nhận email HĐ, nhận push notification, hủy booking. |
| **Nhân viên CS** | Tạo yêu cầu xuất HĐ hộ khách qua CRM (có `csTicketId`). |
| **CRM (Mobio)** | Nhận request HĐ từ CS, gửi sang Platform; nhận sync trạng thái HĐ. |
| **Hotel Service** | Màn Pax Info, validate FE, lưu info HĐ theo booking; xử lý hủy (gọi Partner, tính tiền hoàn). |
| **Platform Service** | Đầu mối xử lý HĐ: validate BE, tạo tracking, gửi request sang ERP, quản lý trạng thái, build HĐ điều chỉnh, sync CRM, thông báo user. |
| **ERP Service** | Nhận request xuất HĐ (async, batch theo cron), gọi VACOM, callback kết quả về Platform, trigger AF gửi mail. |
| **VACOM** | Đơn vị phát hành hóa đơn VAT điện tử. |
| **AF Service** | Gửi email hóa đơn (HĐ gốc / HĐ điều chỉnh) cho user. |
| **Partner** | Xử lý hủy phòng, trả về số tiền bị charge để tính tiền hoàn. |

---

## 3. Use Case tổng quan

| ID | Use Case | Actor | Mô tả |
|---|---|---|---|
| UC-01 | Đặt phòng (booking) | Khách hàng | Đặt phòng; «extend»: tick "Xuất HĐ" + điền thông tin HĐ tại màn Pax Info. |
| UC-02 | Yêu cầu xuất HĐ khi booking | Khách hàng | Nhập thông tin HĐ in-app khi thanh toán. «include» Phát hành hóa đơn. |
| UC-03 | Tạo yêu cầu HĐ qua CS | Nhân viên CS, CRM | Khách không tick lúc booking → liên hệ CS → CS nhập info trên CRM → CRM tạo request (kèm `csTicketId`). «include» Phát hành hóa đơn. |
| UC-04 | Xem trạng thái HĐ | Khách hàng, CRM | Theo dõi trạng thái tracking hóa đơn. |
| UC-05 | Hủy booking & điều chỉnh HĐ | Khách hàng | Hủy 1 phòng hoặc toàn bộ → hệ thống xử lý HĐ theo trạng thái tracking. «include» Phát hành hóa đơn (điều chỉnh). |
| UC-06 | Phát hành hóa đơn | ERP, VACOM, AF | Use case lõi: ERP batch → VACOM xuất HĐ → AF gửi mail → callback Platform. |

---

## 4. Luồng nghiệp vụ chi tiết

### 4.1. Luồng tạo hóa đơn (UC-02, UC-03, UC-06)

**Trigger:**
- **Cách A (In-app):** User tick "Xuất HĐ" + điền info tại màn Pax Info khi thanh toán. Hotel Service validate FE → lưu info HĐ theo booking → đẩy sang Platform.
- **Cách B (CS/CRM):** User liên hệ CS → CS tạo ticket → CRM tạo request HĐ (có `csTicketId`) → gửi Platform.

**Các bước xử lý:**

1. Platform Service **validate BE**: mandatory fields, format, TID (lấy transaction theo Tid), chống trùng.
2. **Không hợp lệ** → status = `Error`, sync CRM (Error), báo lỗi user. **Kết thúc.**
3. **Hợp lệ** → tạo `trackingInvoiceStatus = Init` → gửi request xuất HĐ sang ERP.
4. ERP trả **202 Accepted** → Platform set status = `Pending`, sync CRM (Pending).
   - ERP **timeout/không nhận** → status = `Error`, sync CRM (Error).
5. ERP xử lý **batch theo cron** (không realtime): tạo danh sách HĐ cần xuất → call API VACOM.
6. **VACOM lỗi/timeout** → ERP callback fail → Platform status = `Error`, sync CRM (Error), thông báo user thất bại.
7. **VACOM xuất thành công** → chạy song song (fork):
   - ERP → AF Service: gửi email HĐ VAT cho user.
   - ERP callback success → Platform status = `Success`, lưu DB, sync CRM (Success), push notification thành công cho user.

### 4.2. Luồng hủy booking / điều chỉnh hóa đơn (UC-05)

**Tiền điều kiện hiển thị:**
- Platform kiểm tra booking có hủy được không:
  - **Không hủy được** → **ẩn nút "Hủy"** tại màn Pocket. Kết thúc.
  - **Hủy được** → hiển thị nút "Hủy".

**Các bước xử lý:**

1. User bấm Hủy booking (**1 phòng** hoặc **toàn bộ**) tại Pocket.
2. Platform call Hotel Service → Hotel Service request hủy sang Partner → Partner trả **số tiền bị charge** → Hotel Service **tính tiền hoàn** → lưu thông tin hủy về Platform.
3. Platform **query `trackingInvoice` theo Tid**, rẽ nhánh theo trạng thái:

| Trạng thái tracking | Xử lý |
|---|---|
| **Not found / Error / Cancel** | Không làm gì thêm (no-op). |
| **Init** | Cancel request HĐ; sync CRM = `Cancel`. |
| **Pending** | Gọi ERP `CancelERPRecordTask` → ERP update = `CANCEL` → chặn xuất hóa đơn; sync CRM = `Cancel`; push noti "đã hủy, không xuất HĐ". |
| **Success** | Đi tiếp luồng **hóa đơn điều chỉnh** (bước 4). |

4. **Hóa đơn điều chỉnh** (chỉ khi tracking = Success):
   - **Chờ kết quả tiền hoàn từ Partner** rồi mới build HĐ điều chỉnh (bắt buộc).
   - **Partial (hủy 1 phần):** build HĐ điều chỉnh cho phần hủy + **giữ HĐ gốc**.
   - **Full (hủy toàn bộ):** build HĐ điều chỉnh = 0 + kèm HĐ gốc.
5. Platform gửi HĐ điều chỉnh sang ERP (vào hàng chờ) → ERP trả **202 Accepted** → status = `Pending`, sync CRM.
6. ERP batch theo cron → tạo danh sách HĐ điều chỉnh → call VACOM:
   - **Fail/Timeout** → callback fail → status = `Error`, sync CRM (Error), thông báo user thất bại.
   - **Thành công** → ERP request AF gửi mail HĐ điều chỉnh → AF email user; callback success → status = `Success (đã điều chỉnh)`, sync CRM (Success).
7. Platform push notification cho user.

---

## 5. Business Rules

| ID | Rule |
|---|---|
| BR-01 | Chỉ booking qua **OMH** mới xuất hóa đơn VAT (MoMo là principal). Expedia không xuất HĐ (MoMo là forwarder). |
| BR-02 | HĐ điều chỉnh **chỉ được build sau khi có kết quả tiền hoàn từ Partner**. |
| BR-03 | **Mỗi lần hủy = 1 HĐ điều chỉnh riêng** (delta độc lập), 1 tracking record riêng: `parentInvoiceId` luôn trỏ về HĐ gốc + `adjustmentSeq`. |
| BR-04 | Các tờ HĐ điều chỉnh **chạy độc lập** — không cancel / không gộp / không chờ nhau. 1 tờ fail → retry riêng tờ đó, không ảnh hưởng tờ khác. |
| BR-05 | Record `Error` **không xóa / không ghi đè**. Error = chưa xong → phải retry/re-trigger đến khi `Success`. |
| BR-06 | **Chống trùng** theo cặp (booking, phòng/lần hủy). |
| BR-07 | Hủy toàn bộ (Full) → HĐ điều chỉnh = 0 kèm HĐ gốc. Hủy 1 phần (Partial) → HĐ điều chỉnh phần hủy + giữ HĐ gốc. |
| BR-08 | Booking không hủy được → **ẩn nút Hủy** tại Pocket (không cho user thao tác). |
| BR-09 | ERP xử lý xuất HĐ theo **batch/cron**, không realtime. Platform nhận kết quả qua **callback**. |
| BR-10 | Request từ CS phải có `csTicketId` để truy vết. |
| BR-11 | Mọi thay đổi trạng thái tracking đều phải **sync về CRM (Mobio)**. |

---

## 6. Trạng thái tracking hóa đơn (`trackingInvoiceStatus`)

| Trạng thái | Ý nghĩa | Chuyển tiếp |
|---|---|---|
| `Init` | Platform validate xong, đã tạo tracking, chưa gửi/chưa được ERP nhận. | → `Pending` (ERP 202) / → `Error` (ERP timeout) / → `Cancel` (user hủy booking) |
| `Pending` | ERP đã nhận (202), chờ batch xuất HĐ. | → `Success` (callback success) / → `Error` (callback fail) / → `Cancel` (CancelERPRecordTask) |
| `Success` | VACOM xuất HĐ thành công. | → phát sinh HĐ điều chỉnh khi hủy (record mới, không đổi record gốc) |
| `Error` | Validate fail / ERP timeout / VACOM fail. Giữ record, retry đến khi Success. | → `Pending`/`Success` sau retry |
| `Cancel` | Request bị hủy trước khi xuất. Trạng thái cuối. | — |

---

## 7. Yêu cầu chức năng (Functional Requirements)

| ID | Yêu cầu | Hệ thống |
|---|---|---|
| FR-01 | Màn Pax Info có checkbox "Xuất hóa đơn" + form nhập thông tin HĐ; validate FE (mandatory, format). | Hotel FE |
| FR-02 | Lưu thông tin HĐ theo booking, truyền sang Platform Service. | Hotel Service |
| FR-03 | CRM cho phép CS tạo request xuất HĐ kèm `csTicketId`. | CRM |
| FR-04 | Validate BE: mandatory, format, TID hợp lệ (query transaction theo Tid), chống trùng. | Platform |
| FR-05 | Tạo & quản lý tracking record theo vòng đời Init → Pending → Success/Error/Cancel. | Platform |
| FR-06 | API gửi request xuất HĐ sang ERP; xử lý response 202 / timeout. | Platform ↔ ERP |
| FR-07 | ERP batch (cron) tạo danh sách HĐ, call API VACOM, callback kết quả về Platform. | ERP |
| FR-08 | Gửi email HĐ VAT / HĐ điều chỉnh cho user qua AF Service. | ERP → AF |
| FR-09 | Push notification kết quả (thành công/thất bại/đã hủy) cho user. | Platform |
| FR-10 | Sync trạng thái HĐ về CRM tại mọi bước chuyển trạng thái. | Platform → CRM |
| FR-11 | Kiểm tra điều kiện hủy booking; ẩn/hiện nút Hủy tại Pocket. | Platform |
| FR-12 | Luồng hủy: call Hotel Service → Partner, nhận số tiền charge, tính tiền hoàn. | Hotel Service |
| FR-13 | API `CancelERPRecordTask` để hủy record HĐ đang Pending tại ERP, chặn xuất. | Platform ↔ ERP |
| FR-14 | Build HĐ điều chỉnh (Partial/Full) sau khi có kết quả tiền hoàn; gửi ERP xuất qua VACOM. | Platform |
| FR-15 | Cơ chế retry/re-trigger cho record Error đến khi Success; retry độc lập từng tờ. | Platform/ERP |

---

## 8. Yêu cầu phi chức năng (gợi ý — cần confirm)

- **Idempotency:** chống trùng request xuất HĐ theo (booking, phòng/lần hủy); callback ERP phải idempotent.
- **Async/Batch:** ERP xử lý theo cron — cần SLA thời gian xuất HĐ tối đa (theo quy định HĐ điện tử).
- **Auditability:** giữ toàn bộ tracking record (không xóa Error), truy vết theo `csTicketId`, `parentInvoiceId`, `adjustmentSeq`.
- **Thông báo:** email (AF) + push notification cho mọi kết quả cuối (Success/Error/Cancel).

---

## 9. Điểm mở / cần chốt (Open Issues)

| # | Vấn đề | Ghi chú |
|---|---|---|
| 1 | SLA thời gian xuất HĐ sau thanh toán (quy định thuế về thời điểm xuất HĐ điện tử)? | Cần confirm với Kế toán/Legal. |
| 2 | Cơ chế retry cho record Error: tự động (job) hay thủ công (CS/ops re-trigger)? Max retry? | BR-05 mới nói "retry đến khi Success". |
| 3 | User có được sửa thông tin HĐ sau khi đã submit (trước khi Success)? | Chưa thể hiện trên diagram. |
| 4 | Deadline cho phép yêu cầu xuất HĐ qua CS sau khi thanh toán (bao nhiêu ngày)? | |
| 5 | Trường hợp hủy khi tracking = Pending nhưng ERP đã lỡ xuất (race condition giữa cron và CancelERPRecordTask)? | Cần xử lý: nếu đã xuất → chuyển sang luồng điều chỉnh. |
| 6 | Nội dung/format email HĐ và push notification. | Cần content từ Marketing/CS. |

---

## 10. Tài liệu tham chiếu

| Tài liệu | Link |
|---|---|
| Xuất HĐ hiện có (Confluence) | https://atlassiantool.mservice.com.vn:9443/display/BOOK/VAT+ISSUING |
| BRD OMH | https://docs.google.com/document/d/1XvvE8t_yCM-wFIn7qprT_jSyYQINvARGoakfUnvBnVI/edit?tab=t.0 |
| Diagram source | `index.html` — Mindmap, Use case, Sequence (tạo / hủy-điều chỉnh), Activity, BPMN |
