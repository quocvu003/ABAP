# Hướng dẫn Sửa lỗi SAP ABAP S/4HANA (Error Fix Reference Guide)

Tài liệu này được trích xuất và dịch từ file tài liệu hướng dẫn sửa lỗi (`error_handle.pdf`). Hệ thống sửa lỗi tự động sẽ dựa vào đây làm căn cứ sửa code.

---

## 0. Nguyên tắc chung khi sửa lỗi

* **Không tự đoán khi thiếu dữ liệu:** Quy tắc này áp dụng cho toàn bộ task và mọi mã lỗi. Nếu thiếu thông tin cần thiết để xác định đúng cách sửa, ví dụ thiếu type/length/decimal của field, structure/table definition, logic trước/sau đoạn code, ATC detail, SAP Note/message, mapping field, hoặc business rule, phải yêu cầu người dùng cung cấp thêm trước khi viết code sửa.
* **Tuyệt đối không tự suy đoán:** Không tự đoán field, table, mapping, length, nghiệp vụ hoặc hướng xử lý khi dữ liệu chưa đủ.

---

## 1. B999 - Lỗi thay đổi màn hình Giao dịch trong Batch Input

* **ID:** `B999`
* **Nội dung lỗi (エラー内容):** Giao diện hoặc luồng chuyển trang (Screen Transition) của giao dịch tiêu chuẩn SAP thay đổi trên S/4HANA dẫn đến lỗi Batch Input (ví dụ: Thay đổi tên chương trình/số hiệu màn hình, hoặc xuất hiện thêm màn hình/popup mới).
* **Cách khắc phục (対応方法):**
  1. Sử dụng Transaction Recorder (T-Cd: `SHDB`) chạy thử giao dịch trên hệ thống mới S/4HANA để ghi lại luồng màn hình mới.
  2. Xác định các thông tin chính xác về: Tên chương trình (Program Name), Số màn hình (Dynpro Number), và Tên trường (Field Name).
  3. Thay thế các dòng BDC tương ứng trong code cũ bằng các thông số mới ghi nhận được từ `SHDB`.

### Danh sách các Transaction phổ biến cần sửa đổi (修正が必要だったトランザクション)

| TR-CD | Tên giao dịch (名称) | Nội dung lỗi / Thay đổi (エラー内容 - 変更概要) |
| --- | --- | --- |
| **`SU01`** | Quản lý người dùng (ユーザ管理) | Tên chương trình xử lý màn hình thay đổi (`SAPLSUU5` $\rightarrow$ `SAPLSUID_MAINTENANCE`). |
| **`OB52`** | Thay đổi kỳ kế toán (会計期間の変更 - C FI テーブル変更 T001B) | Xuất hiện thêm màn hình popup mới không có trên ECC. |

---

### Ví dụ thực tế sửa đổi giao dịch `SU01` (Chương trình `ZHKTCM0004`)

#### 1. Kiểm tra đoạn code cũ (ECC)

Chương trình cũ sử dụng chương trình màn hình `SAPLSUU5`, màn hình `0050` và `0500`, trường tên user `USR02-BNAME`.

```abap
FORM SET_BDCDATA.
  REFRESH TD_BDCDATA.
  
  PERFORM BDC_INPUT USING:
    'X'  'SAPLSUU5'  '0050',
    ' '  'USR02-BNAME'  TH_USR02-BNAME,
    ' '  'BDC_OKCODE'  'LOCK',
    'X'  'SAPLSUU5'  '0500',
    ' '  'BDC_OKCODE'  'LOCK'.
ENDFORM.
```

#### 2. Kết quả ghi nhận từ `SHDB` (trên S/4HANA)

Khi ghi lại thao tác chỉnh sửa User trên S/4HANA qua `SHDB`, thông tin mới ghi nhận được là:

* **Màn hình đầu tiên:** Program: `SAPLSUID_MAINTENANCE`, Dynpro: `1050`, Trường nhập User: `SUID_ST_BNAME-BNAME`.
* **Màn hình tiếp theo:** Program: `SAPLSUID_MAINTENANCE`, Dynpro: `1500`.

#### 3. Cú pháp code sau khi sửa (S/4HANA)

Thay thế nội dung cũ bằng thông tin ghi nhận từ `SHDB` (áp dụng định dạng comment thay thế `REPLACE` của dự án):

```abap
FORM SET_BDCDATA.
  REFRESH TD_BDCDATA.
  
  PERFORM BDC_INPUT USING:
* { REPLACE WKTK907116
*   'X'  'SAPLSUU5'  '0050',
    'X'  'SAPLSUID_MAINTENANCE'  '1050',
* } REPLACE
* { REPLACE WKTK907116
*   ' '  'USR02-BNAME'  TH_USR02-BNAME,
    ' '  'SUID_ST_BNAME-BNAME'  TH_USR02-BNAME,
* } REPLACE
    ' '  'BDC_OKCODE'  'LOCK',
* { REPLACE WKTK907116
*   'X'  'SAPLSUU5'  '0500',
    'X'  'SAPLSUID_MAINTENANCE'  '1500',
* } REPLACE
    ' '  'BDC_OKCODE'  'LOCK'.
ENDFORM.
```

---

## 2. F999 - Danh sách Function Module tiêu chuẩn SAP không khả dụng (Unsupported Function Modules)

* **ID:** `F999`
* **Nội dung lỗi (エラー内容):** Việc sử dụng các Function Module (汎用M) cũ không còn được hỗ trợ trên SAP HANA DB / S4HANA. Do kết quả phân tích từ công cụ PANAYA không liệt kê chi tiết nội dung của các Function Module này trong danh sách tổng hợp, lập trình viên cần chủ động kiểm tra trực tiếp trong mã nguồn chương trình (PGM).
* **Cách khắc phục (対応方法):**
  Lập trình viên áp dụng các quy tắc phân loại xử lý sau đối với các Function Module thuộc danh sách F999:
  
  1. **Nếu trùng với Function Module đã có mã lỗi di trú riêng** (ví dụ: `WS_DOWNLOAD` đã có mã lỗi `F014` riêng) $\rightarrow$ Thực hiện chỉnh sửa theo đúng tài liệu hướng dẫn của lỗi đó.
  2. **Nếu là Function Module Add-on/Custom do dự án tự phát triển** (tên bắt đầu bằng chữ `Z` hoặc `Y`) $\rightarrow$ **Không cần sửa đổi (改修不要)**.
  3. **Đối với các Function Module tiêu chuẩn khác của SAP nằm trong danh sách bảng đối chiếu bên dưới:**
     * Lập trình viên đối chiếu với cột **"Cần sửa/Không cần sửa" (改修必要・不要)** trong sheet `汎用M` thuộc file "Danh sách đối tượng sửa đổi" (改修対象一覧):
       * Nếu giá trị ở cột này để **Trống (空白)** $\rightarrow$ **Không cần sửa đổi (改修不要)**.
       * Nếu giá trị ở cột này ghi là **"改修対象" (Cần sửa đổi)** (Ví dụ trường hợp: **`WS_EXECUTE`**) $\rightarrow$ Lập trình viên cần **Tạo ticket/đề xuất vấn đề (課題起票)** để báo cáo và liên hệ trực tiếp với Trưởng nhóm Phát triển (Development Lead - 開発L) để nhận chỉ thị xử lý, không tự ý sửa đổi code.

### Bảng tra cứu trạng thái sửa đổi của Function Module tiêu chuẩn SAP (汎用M 改修必要・不要 一覧)

| Tên Function Module (汎用M) | Trạng thái (改修必要・不要) | Hành động yêu cầu (対応内容) |
| --- | --- | --- |
| **`WS_EXECUTE`** | **`改修対象`** (Cần sửa) | Tạo ticket báo cáo và liên hệ Development Lead (L) |
| `POPUP_TO_CONFIRM_STEP` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `SWW_WI_AGENTS_CHANGE` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `SWW_WI_FORWARD` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `CONVERSION_EXIT_ALPHA_INPUT` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `HELP_VALUES_GET_WITH_TABLE` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `SUSR_USER_PROFS_PROFILES_GET` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `POPUP_TO_CONFIRM_WITH_MESSAGE` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `CONVERSION_EXIT_CUNIT_OUTPUT` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `WS_QUERY` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `WS_UPLOAD` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `WS_DOWNLOAD` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `HELP_OBJECT_SHOW_FOR_FIELD` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `DDIF_NAMETAB_GET` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `RS_CONV_EX_2_IN` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `DDIF_TABL_GET` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `DYNP_VALUES_READ` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `DYNP_GET_STEPL` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `F4IF_FIELD_VALUE_REQUEST` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `F4TOOL_CHECKTABLE_HELP` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `GET_FIELDTAB` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `VIEW_AUTHORITY_CHECK` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修không) |
| `POPUP_TO_CONFIRM_DATA_LOSS` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `GUI_UPLOAD` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `HR_URL_CALL_BROWSER` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `WS_FILENAME_GET` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `GUI_DOWNLOAD` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `HR_JP_CONV_UNI_STR_TO_CP_XSTR` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `STRING_LENGTH` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `CONVERSION_EXIT_CUNIT_INPUT` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `ME_CHECK_T160M` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `SD_BACKORDER_CHECK_AND_SAVE` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `SD_BACKORDER_LIST` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `OUTBOUND_CALL_01010000_P` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |
| `OUTBOUND_CALL_01000802_P` | *Để trống* (Không cần sửa) | Không cần sửa đổi (改修不要) |

---

## 6. Quy tắc đặt tên biến và đối tượng (Project Coding Convention)

Mọi biến khai báo mới hoặc sửa đổi trong dự án phải tuân thủ quy tắc đặt tên (Naming Convention) dưới đây.

> [!IMPORTANT]
> **Nguyên tắc áp dụng đối với mã nguồn hiện hữu (Existing Code):**
>
> 1. **Giữ nguyên định dạng gốc:** Không tự ý căn lề, định dạng lại khoảng trắng của mã nguồn cũ nếu không có yêu cầu thay đổi logic liên quan.
> 2. **Giữ nguyên tên biến cũ:** Đối với các biến, tham số subroutine (FORM), hoặc đối tượng đã tồn tại sẵn trong code cũ, **không thực hiện đổi tên** theo Naming Convention Mục 6. Chỉ bắt buộc áp dụng Naming Convention Mục 6 đối với các biến hoặc đối tượng khai báo mới hoàn toàn.

### 6.1. Bảng Tra cứu Naming Convention

| Loại đối tượng (Object Type) | Định nghĩa Kiểu (Types) | Toàn cục (Global) | Cục bộ (Local) | FORM Parameter - Using | FORM Parameter - Changing | FORM Parameter - Using & Changing | FM Parameter - Import | FM Parameter - Export | FM Parameter - Changing | FM Tables (Import/Export/Changing) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **定数 (Constants)** | | `CNS_*****` | | | | | | | | |
| **変数 (Variables)** | `TP_*****_VAL` | `GV_*****` | `LV_*****` | `IV_*****` | `OV_*****` | `PV_*****` | `IV_*****` | `OV_*****` | `PV_*****` | |
| **構造 (Structures)** | `TP_*****_STR` | `GS_*****` | `LS_*****` | `IS_*****` | `OS_*****` | `PS_*****` | `IS_*****` | `OS_*****` | `PS_*****` | |
| **内部T (Internal Tables)** | `TP_*****_TBL` | `GT_*****` | `LT_*****` | `IT_*****` | `OT_*****` | `PT_*****` | `IT_*****` | `OT_*****` | `PT_*****` | `IT_*****` (Import) / `OT_*****` (Export) / `PT_*****` (Changing) |
| **RANGES** | `TP_*****_RNG` | `GR_*****` | `LR_*****` | `IR_*****` | `OR_*****` | `PR_*****` | `IR_*****` | `OR_*****` | `PR_*****` | `IR_*****` (Import) / `OR_*****` (Export) / `PR_*****` (Changing) |
| **PARAMETERS** | | `P_*****` | | | | | | | | |
| **CHECKBOX** | | `C_*****` | | | | | | | | |
| **RADIOBUTTON** | | `RB_*****` | | | | | | | | |
| **SELECT-OPTIONS** | | `S_*****` | | | | | | | | |
| **CONTROLS (Table)** | | `TC_9***` | | | | | | | | |
| **CONTROLS (Tab)** | | `TS_9***` | | | | | | | | |
| **Số màn hình Dynpro** | | `9***` | | | | | | | | |
| **GUI Status** | | `S9***` (*1) | | | | | | | | |
| **GUI Title (GUI 表題)** | | `T9***` (*1) | | | | | | | | |

* **Ghi chú (*1):** Đối với chương trình Report, có thể sử dụng `S*` cho GUI Status và `T*` cho GUI Title.

---

## 7. Quy tắc ABAP Dictionary (ABAP Dictionary – Rule Summary)

Khi tạo mới hoặc chỉnh sửa các đối tượng trong ABAP Dictionary (SE11), phải tuân thủ các quy tắc sau:

| STT | Đối tượng | Quy tắc (Rule) |
| :--- | :--- | :--- |
| 1 | **Field (項目)** | Không dùng số thứ tự hoặc romaji cho tên field. |
| 2 | **Data Element** | Phải sử dụng Domain (ưu tiên tái sử dụng - reuse). |
| 3 | **Domain** | Nếu có giá trị cố định (fixed values) -> bắt buộc định nghĩa range/value. |
| 4 | **Search Help** | Field dùng Data Element mới -> phải gán Search Help. |
| 5 | **Table Maintenance Generator (TMG)** | Khi tạo/cập nhật table -> phải tạo/cập nhật TMG. |
| 6 | **Lock Object** | Khi cập nhật dữ liệu -> phải tạo/cập nhật Lock Object. |
| 7 | **Table Type** | Khi tạo/sửa table hoặc structure -> phải tạo/cập nhật Table Type. |
| 8 | **Index** | Tạo index phù hợp khi cần tối ưu truy vấn. |
| 9 | **Data Class** | Bắt buộc sử dụng User Data Class. |
| 10 | **Size Category** | Phải thiết lập phù hợp với số lượng record dự kiến. |
| 11 | **Translation** | Ngôn ngữ gốc là tiếng Nhật (JA), bắt buộc phải dịch sang tiếng Anh (EN). |

---

## 8. Quy tắc lập trình cho chương trình Report và Online (Program Coding Rules)

Khi viết chương trình Report hoặc Online, bắt buộc tuân thủ các quy tắc cấu trúc và lập trình dưới đây:

### 8.1. Cấu trúc chương trình (プログラム構成)

Chương trình phải được chia thành các Include theo cấu trúc sau:

* **Khai báo dữ liệu (データ宣言):** `INCLUDE: *****`
* **Tham số đầu vào (パラメータ):** `INCLUDE: *****`
* **Sự kiện (イベント):** `INCLUDE: *****`
* **Subroutine (サブルーチン):** `INCLUDE: *****`
* **Dynpro:**
  * **PBO:** `INCLUDE: *****`
  * **PAI:** `INCLUDE: *****`
*Lưu ý: Nếu SAP tự động đề xuất các Include chuẩn (SAP standard), ưu tiên sử dụng cấu trúc chuẩn của SAP.*

### 8.2. Khai báo dữ liệu (データ宣言について)

Tạo các file Include riêng biệt theo mục đích sử dụng (ví dụ: Include khai báo tham số, Include khai báo chung). Trong mỗi Include khai báo dữ liệu, sắp xếp các phần khai báo theo thứ tự sau:

1. **Include:** `INCLUDE:`
2. **Type Pools:** `TYPE-POOLS:`
3. **Data Types:** `TYPES:`
4. **Biến đơn:** `DATA:`
5. **Cấu trúc (Structure):** `DATA:`
6. **Bảng nội bộ (Internal Table):** `DATA:`
7. **Bảng chọn (Selection Table):** `DATA:`
8. **Field Symbol:** `FIELD-SYMBOLS:`
9. **Control:** `CONTROLS:`
10. **Table:** `TABLES:`
11. **Hằng số (Constants):** `CONSTANTS:`

**Quy tắc chung:**

* **Global Types & Variables:** Gom các kiểu dữ liệu và biến có thể dùng chung độc lập với chương trình vào một Include dùng chung. Đối với các cấu trúc (Structure), ưu tiên thiết kế trong ABAP Dictionary (SE11) thay vì định nghĩa trực tiếp trong code *(trừ trường hợp cấu trúc chỉ dùng tạm thời/không tái sử dụng)*. Tránh lạm dụng biến toàn cục (Global Variable) để giữ phạm vi biến rõ ràng.
* **Local Types & Variables:** Các kiểu dữ liệu và biến chỉ có hiệu lực trong phạm vi cục bộ thì khai báo trực tiếp trong Report program hoặc Subroutine.
* **Lệnh TABLES:** Hạn chế tối đa sử dụng lệnh `TABLES` (chỉ dùng cho các trường hiển thị trên màn hình Dynpro).

### 8.3. Tham số đầu vào (パラメータについて)

* Sắp xếp theo nhóm: `PARAMETERS:` và `SELECT-OPTIONS:`.

* Hỗ trợ tối đa cho cả xử lý tương tác (Online/Interactive) và xử lý nền (Batch/Background). Khi chạy ở chế độ Batch, thiết lập bỏ qua các màn hình Dynpro tiếp theo.
* Tránh hardcode các giá trị cố định (như Company Code, Plant, v.v.). Nên đưa các thông số này ra ngoài màn hình chọn làm giá trị mặc định (Default Value) thiết lập tại sự kiện khởi tạo.
* Xử lý tải file (Upload/Download) cần hỗ trợ cả đường dẫn trên máy local và trên File Server.

### 8.4. Sự kiện (イベントについて)

Các sự kiện phải được cấu trúc rõ ràng và mỗi sự kiện chỉ gọi một Subroutine duy nhất để xử lý logic:

* `INITIALIZATION.` -> `PERFORM ...`
* `AT SELECTION-SCREEN.` -> `PERFORM ...`
* `AT SELECTION-SCREEN ON (P_*****/S_*****).` -> `PERFORM ...`
* `AT SELECTION-SCREEN ON END OF (S_*****).` -> `PERFORM ...`
* `AT SELECTION-SCREEN ON BLOCK (BlockName).` -> `PERFORM ...`
* `AT SELECTION-SCREEN ON RADIOBUTTON GROUP (GroupName).` -> `PERFORM ...`
* `AT SELECTION-SCREEN ON VALUE-REQUEST FOR (P_*****/S_*****-LOWorHIGH).` -> `PERFORM ...`
* `AT SELECTION-SCREEN OUTPUT.` -> `PERFORM ...`
* `AT SELECTION-SCREEN ON EXIT-COMMAND.` -> `PERFORM ...`
* `START-OF-SELECTION.` -> `PERFORM ...`
* `END-OF-SELECTION.` -> `PERFORM ...`
* `TOP-OF-PAGE.` -> `PERFORM ...`
* `TOP-OF-PAGE DURING LINE-SELECTION.` -> `PERFORM ...`
* `AT LINE-SELECTION.` -> `PERFORM ...`
* `AT USER-COMMAND.` -> `PERFORM ...`

### 8.5. Subroutine (サブルーチンについて)

* Các biến/kiểu dữ liệu chỉ dùng trong Subroutine hoặc các Subroutine cấp dưới phải được khai báo cục bộ bên trong Subroutine đó.

* Ưu tiên phát triển các logic có thể dùng chung thành Function Module (FM) thay vì Subroutine cục bộ.
* **Rất quan trọng:** Các thao tác đọc/ghi cơ sở dữ liệu và cập nhật chứng từ (UPDATE/INSERT/DELETE) không được viết trực tiếp trong chương trình Report hoặc Online, mà phải được đóng gói vào Function Module.
* Không sử dụng tham số `TABLES` trong Subroutine để truyền nhận dữ liệu vì khó phân biệt tham số đầu vào (Input) và đầu ra (Output). Hãy sử dụng `USING` hoặc `CHANGING` với kiểu dữ liệu bảng.

### 8.6. Dynpro và PBO/PAI (Dynpro, PBO, PAI について)

* Cấu trúc Dynpro thành `PROCESS BEFORE OUTPUT` (PBO) và `PROCESS AFTER INPUT` (PAI). Khuyến khích mô-đun hóa các màn hình Dynpro có ý nghĩa thành Function Module.

* **PBO:** Thực hiện khởi tạo biến, kiểm soát ẩn/hiển thị của các nút bấm và đối tượng trên màn hình.
* **PAI:** Thực hiện kiểm tra tính hợp lệ dữ liệu (validation checks) và thu thập, xử lý dữ liệu cần thiết.

### 8.7. Các quy tắc khác

* **Dịch thuật (翻訳):** Ngôn ngữ gốc (Master Language) của chương trình phải là tiếng Nhật (`JA`), và bắt buộc phải dịch đầy đủ sang tiếng Anh (`EN`).

* **Unicode:** Bắt buộc phải tích chọn và kích hoạt kiểm tra Unicode (Unicode Check Active) cho chương trình.
* **Kết quả xử lý (処理終了後の結果):** Phải luôn xuất kết quả xử lý (thành công/thất bại/số lượng dòng xử lý...) rõ ràng sau khi kết thúc chương trình.

---

## 9. Quy tắc lập trình cho Function Module (汎用モジュールコーディングルール)

Khi viết Function Module (FM), bắt buộc tuân thủ các quy tắc cấu trúc và lập trình dưới đây:

### 9.1. Cấu trúc Function Module (汎用モジュール構成)

* **Xử lý chính (主処理):** Logic xử lý chính của FM phải được viết gọn gàng trong một Subroutine chính:
  `PERFORM MAIN_*************************.`

* Các phần khai báo dữ liệu chung phải được đưa vào Include khai báo đầu trang (ví dụ: `*****TOP`). Việc phân chia các file Include phụ thuộc tương tự như quy tắc đối với chương trình Report và Online (Mục 8).

### 9.2. Quy tắc cho xử lý chính (主処理について)

Thiết lập các tham số đầu ra (Export parameters) đầy đủ nhất có thể:

* **Tham số EXPORT biểu thị thành công/thất bại:** Khai báo biến `O_SUBRC` (với quy ước `0`: Thành công, `4`: Thất bại, các giá trị khác tùy ý nghĩa nghiệp vụ cụ thể).
* **Tham số dạng bảng chứa kết quả xử lý chi tiết:** Khai báo bảng `OT_RESULT` (trả về toàn bộ nội dung thành công, thông báo lỗi hoặc kết quả xử lý nếu có nhiều dòng kết quả cần xuất ra).
* **Tối ưu xử lý hàng loạt (Bulk/Mass Processing):** Cân nhắc thiết kế các tham số đầu vào và điều kiện chọn dưới dạng tham số Bảng (Table parameters) để nhận dữ liệu hàng loạt và trả về kết quả dưới dạng Bảng.
* **Tái sử dụng Function Module:** Đối với các luồng xử lý tương tự nhau nhưng có một vài khác biệt nhỏ về nghiệp vụ, **không tạo mới nhiều Function Module**. Thay vào đó, hãy dùng chung một Function Module và điều khiển các luồng logic khác nhau bằng cách sử dụng các cờ điều khiển (Control Flags / Import parameters).
* **Hạn chế dùng Changing và Tables:** Hãy ưu tiên thay thế các tham số dạng `CHANGING` và `TABLES` bằng cặp tham số rõ ràng `IMPORT` và `EXPORT` bất cứ khi nào có thể để làm rõ hướng truyền nhận dữ liệu.

---

## 10. Bảng tra cứu quy tắc đặt tên đối tượng SAP Custom và JOB (SAP Naming Convention – Cheat Sheet)

Khi tạo mới các đối tượng tùy chỉnh (Custom Objects) hoặc các công việc chạy nền (Background Jobs), bắt buộc tuân thủ cấu trúc đặt tên dưới đây:

### 10.1. Cấu trúc tên đối tượng Custom cốt lõi (Core Structure)

Định dạng chung: `Z + aa + b + c + d + nnn`
Trong đó:

* **`Z`:** Custom Object (Bắt buộc bắt đầu bằng chữ Z).
* **`aa` (Phân hệ / Module):**
  * `SD`: Sales (Bán hàng)
  * `FI`: Finance (Tài chính)
  * `PP`: Production (Sản xuất)
  * `HR`: Human Resources (Nhân sự)
  * `PU`: Purchasing (Mua hàng)
* **`b` (Loại quy trình / Process Type):**
  * `M`: Master Data (Dữ liệu chủ)
  * `T`: Transaction (Giao dịch)
  * `D`: Data Migration (Chuyển đổi dữ liệu)
  * `A`: Analysis (Phân tích)
  * `S`: System (Hệ thống)
* **`c` (Phương thức / Method):**
  * `U`: Update (Cập nhật)
  * `R`: Report (Báo cáo)
  * `F`: Form (Biểu mẫu)
  * `I`: Include (Chương trình con)
  * `Z`: Other (Khác)
* **`d` (Hệ thống / System):**
  * `0`: Common (Dùng chung cho toàn hệ thống)
  * `J`: Japan (Nhật Bản)
  * `K`: Korea (Hàn Quốc)
  * `U`: US (Mỹ)
  * `Z`: Personal (Cá nhân)
* **`nnn`:** Counter (Số thứ tự chạy từ `001` đến `999`).

*Ví dụ: `ZSDMU0001` = Phân hệ `SD` (Sales) + Quy trình `M` (Master) + Phương thức `U` (Update) + Hệ thống `0` (Common) + Số thứ tự `001`.*

### 10.2. Ví dụ về các đối tượng (Object Examples)

* **Chương trình Report (Report program):** `ZSDMU0001`

* **Bảng cơ sở dữ liệu (Database Table):** `ZSDM0001`
* **Góc nhìn (View):** `ZSDVW0001`
* **Hàm xử lý (Function Module):** `Z_SDMU0_GET_DATA`
* **Mã giao dịch (Transaction Code / T-code):** `Z001`

### 10.3. Quy tắc đặt tên JOB chạy nền (JOB Naming)

Định dạng chung: `J + e + f + g + d + nnn`
Trong đó:

* **`J`:** Đại diện cho JOB (Bắt buộc).
* **`e` (Loại Job / Job types):**
  * `D`: Daily (Hằng ngày)
  * `W`: Weekly (Hằng tuần)
  * `M`: Monthly (Hằng tháng)
  * `Y`: Yearly (Hằng năm)
* **`f` (Loại hình nghiệp vụ / Business types):**
  * `A`: Aviation (Hàng không)
  * `C`: Connector (Kết nối)
  * `U`: UIS (Hệ thống thông tin người dùng)
  * `Z`: Common (Dùng chung)
* **`g` (Connector / System):** Đại diện cho hệ thống hoặc kết nối cụ thể.
* **`d` (Hệ thống / System):** `0` đại diện cho Common (Dùng chung).
* **`nnn`:** Số thứ tự tăng dần.

*Ví dụ: `JDSC0001` = Job chạy hằng ngày (`D`) + Nghiệp vụ Sales (`S`) + Hệ thống kết nối (`C`) + Common (`0`) + Số thứ tự (`001`).*

---

## 11. Hướng dẫn di trú và Tối ưu hóa hiệu năng ABAP trên S/4HANA (Migration & Performance Best Practices)

Khi nâng cấp hệ thống và tối ưu hóa hiệu năng trên cơ sở dữ liệu SAP HANA (In-Memory Database), lập trình viên bắt buộc phải tuân thủ các quy tắc lập trình dưới đây để đảm bảo code hoạt động chính xác, ổn định và đạt tốc độ tối đa.

### 11.1. Quy tắc tối ưu hóa hiệu năng Cơ sở dữ liệu (Database Performance Guidelines)

#### 1. Tuyệt đối không viết lệnh `SELECT` bên trong vòng lặp (`LOOP ... ENDLOOP.`)

* **Lý do:** Việc gọi `SELECT` liên tục trong vòng lặp (N+1 Select) sẽ làm tăng số lượng kết nối mạng (Network round-trips) giữa Application Server và Database Server. Đây là nguyên nhân hàng đầu gây chậm/treo hệ thống.
* **Cách khắc phục:**
  * Sử dụng mệnh đề `INNER JOIN` hoặc `LEFT OUTER JOIN` để lấy dữ liệu liên kết từ nhiều bảng DB trong một câu lệnh duy nhất.
  * Nếu dữ liệu nguồn nằm ở một bảng nội bộ (Internal Table), sử dụng phương pháp `FOR ALL ENTRIES IN` hoặc Join trực tiếp với bảng nội bộ (từ phiên bản ABAP 7.52 trở đi).
* **Mẫu xử lý sai:**

  ```abap
  LOOP AT lt_vbak INTO ls_vbak.
    SELECT SINGLE * FROM vbap INTO ls_vbap
      WHERE vbeln = ls_vbak-vbeln.
  ENDLOOP.
  ```

* **Mẫu xử lý đúng (sử dụng JOIN):**

  ```abap
  SELECT a~vbeln, b~posnr, b~matnr
    FROM vbak AS a
    INNER JOIN vbap AS b ON a~vbeln = b~vbeln
    INTO TABLE @DATA(lt_sales_data).
  ```

#### 2. Luôn kiểm tra bảng nội bộ trước khi sử dụng `FOR ALL ENTRIES` (FAE)

* **Lý do:** Nếu bảng nội bộ dùng làm điều kiện so sánh trong `FOR ALL ENTRIES IN` bị rỗng (không chứa dữ liệu), SAP sẽ tự động bỏ qua điều kiện `WHERE` và thực hiện quét toàn bộ dữ liệu của bảng cơ sở dữ liệu (Full Table Scan). Điều này cực kỳ nguy hiểm nếu bảng DB chứa hàng triệu dòng dữ liệu.
* **Cách khắc phục:** Luôn đặt khối điều kiện `IF <internal_table> IS NOT INITIAL.` trước khi thực hiện truy vấn.
* **Mẫu xử lý đúng:**

  ```abap
  IF lt_sales_data IS NOT INITIAL.
    SELECT matnr, maktx
      FROM makt
      FOR ALL ENTRIES IN @lt_sales_data
      WHERE matnr = @lt_sales_data-matnr
        AND spras = @sy-langu
      INTO TABLE @DATA(lt_material_txt).
  ENDIF.
  ```

#### 3. Hạn chế sử dụng `SELECT *` (Chỉ lấy các trường cần thiết)

* **Lý do:** Cơ sở dữ liệu SAP HANA lưu trữ dữ liệu theo dạng cột (Column-based storage). Việc truy vấn `SELECT *` sẽ buộc hệ thống phải đọc và chuyển toàn bộ các cột dữ liệu qua mạng lên Application Server, gây lãng phí bộ nhớ đệm và băng thông.
* **Cách khắc phục:** Liệt kê tường minh danh sách các trường cần lấy trong lệnh `SELECT`.
* **Mẫu xử lý đúng:**

  ```abap
  SELECT vbeln, erdat, ernam, netwr
    FROM vbak
    INTO TABLE @DATA(lt_vbak_fields)
    WHERE bukrs = '1000'.
  ```

#### 4. Code Pushdown (Đẩy logic tính toán xuống DB)

* Tận dụng tối đa các hàm toán học, xử lý chuỗi và gom nhóm trực tiếp bằng SQL Expressions (ví dụ: `CASE WHEN`, `CONCAT`, `COALESCE`, `SUM`, `AVG`) hoặc sử dụng Core Data Services (CDS Views) và ABAP Managed Database Procedures (AMDP) để xử lý logic nặng ngay tại Database Server thay vì kéo dữ liệu thô về xử lý bằng vòng lặp trong ABAP.

---

### 11.2. Quy tắc tương thích S/4HANA (S/4HANA Technical Compatibility)

#### 1. Thay đổi cấu trúc dữ liệu Tài chính (Universal Journal - ACDOCA)

* Trên S/4HANA, các bảng dữ liệu Tài chính cũ (như `GLT0`, `COEP`, `BSIS`, `BSAS`, `BSIK`, `BSAK`, `BSID`, `BSAD`) đã được đơn giản hóa cấu trúc vật lý và chuyển hướng về bảng hợp nhất **`ACDOCA`**.
* Mặc dù SAP có cung cấp các Compatibility Views để tự động ánh xạ, nhưng để tối ưu hiệu năng và tránh sai lệch số liệu, khuyến khích lập trình viên chuyển hướng trực tiếp truy vấn dữ liệu từ bảng **`ACDOCA`** khi phát triển các báo cáo tài chính mới.

#### 2. Hợp nhất Master Data Khách hàng/Nhà cung cấp (Business Partner)

* Các bảng lưu thông tin Customer (`KNA1`) và Vendor (`LFA1`) vẫn tồn tại để lưu thông tin chung, nhưng quy trình cập nhật/tạo mới bắt buộc phải thông qua API/BAPI thuộc phân hệ **Business Partner (BP)** thay vì gọi trực tiếp Batch Input hoặc ghi dữ liệu thẳng vào các bảng Master Data.

#### 3. Bắt buộc kích hoạt Unicode Check

* Mọi chương trình ABAP khi di trú lên S/4HANA phải tích chọn thuộc tính **Unicode Checks Active**.
* Các câu lệnh xử lý chuỗi dựa trên số byte cứng (Byte-oriented) phải chuyển sang hướng ký tự (Character-oriented) bằng cách sử dụng các lệnh như `DESCRIBE FIELD ... IN CHARACTER MODE` để đảm bảo tương thích hoàn toàn.

---

### 11.3. Thay thế các câu lệnh lỗi thời (Obsolete Syntax Replacement)

Nhằm đảm bảo chất lượng code sạch (Clean ABAP) và tương thích lâu dài, lập trình viên cần loại bỏ và thay thế các từ khóa ABAP lỗi thời:

* **Ưu tiên cú pháp ABAP mới:**
  * Khi viết code sửa đổi, luôn ưu tiên constructor expression và syntax mới như `VALUE #( FOR ... IN ... )`, inline declaration, `CORRESPONDING`, table expression và `@` host variables trong Open SQL.
  * Chỉ quay về syntax cũ khi cần tương thích hệ thống hoặc cần đặt pseudo-comment rõ ràng trên từng dòng lỗi ATC.

* **Lệnh `REFRESH <internal_table>`:**
  * *Mô tả:* Lệnh cũ để xóa bảng nội bộ.
  * *Thay thế:* Sử dụng **`CLEAR <internal_table>`** hoặc **`FREE <internal_table>`** (để giải phóng hoàn toàn vùng nhớ).
* **Lệnh `MOVE <source> TO <target>`:**
  * *Mô tả:* Lệnh gán giá trị cũ.
  * *Thay thế:* Sử dụng toán tử gán trực tiếp **`<target> = <source>`** hoặc toán tử gán có điều kiện `CORRESPONDING` / `VALUE`.
* **Lệnh `TABLES`:**
  * *Mô tả:* Khai báo vùng nhớ bảng dùng chung.
  * *Thay thế:* Chỉ dùng `TABLES` cho các biến gắn với màn hình Dynpro tiêu chuẩn SAP. Các khai báo dữ liệu trong code logic phải dùng cấu trúc `DATA` chuẩn.
* **Lệnh `DESCRIBE TABLE ... OCCURS 0`:**
  * *Thay thế:* Sử dụng `DATA lt_table TYPE TABLE OF ...` và lấy số dòng bằng hàm `lines( lt_table )` thay vì dùng các từ khóa liên quan đến bộ nhớ đệm `OCCURS`.
