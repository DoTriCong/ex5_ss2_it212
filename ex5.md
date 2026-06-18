# BÀI 5: Sáng tạo (Thiết kế Quy trình & Prompt Kiểm chứng Đầu ra - AI Verifier)

## Bối cảnh & Ý đồ thiết kế

Trong hệ thống bán vé trực tuyến **GreenBus**, tính năng kiểm tra mã vé hợp lệ là nghiệp vụ quan trọng. Để đảm bảo chất lượng và tránh lỗi logic tiềm ẩn, ta thiết kế quy trình 2 bước:

* **Bước 1 (Code Generation Prompt):** Yêu cầu AI sinh mã nguồn lớp `TicketValidator` bằng Java, xử lý đầy đủ quy tắc nghiệp vụ và chặn các lỗi biên.
* **Bước 2 (AI Auditing Prompt):** Yêu cầu AI đóng vai trò kiểm thử/kiểm toán độc lập, rà soát mã nguồn ở Bước 1, phát hiện lỗi logic tiềm ẩn và đề xuất sửa đổi.

Quy trình này giúp **tách biệt vai trò sinh mã và kiểm chứng**, hạn chế rủi ro ảo giác, đồng thời nâng cao độ tin cậy của đầu ra.

---

# Bước 1: Prompt sinh mã nguồn

## Prompt

> Hãy viết cho tôi lớp `TicketValidator` bằng Java để kiểm tra tính hợp lệ của mã vé theo các quy tắc sau:
>
> * Mã vé bắt buộc bắt đầu bằng tiền tố `BUS-`.
> * Tiếp theo là mã tỉnh/thành phố gồm đúng 2 ký tự chữ in hoa.
> * Tiếp theo là dấu gạch ngang `-` và 6 chữ số biểu diễn ngày đi xe theo định dạng `YYMMDD`.
> * Ngày đi xe không được là ngày trong quá khứ.
>
> Ngoài ra, hãy chặn các lỗi biên:
>
> * Mã vé null.
> * Chuỗi rỗng.
> * Định dạng ngày tháng không hợp lệ (ví dụ ngày 32 tháng 13).
> * Ngày đi trong quá khứ.
>
> Viết code đầy đủ với phương thức:
>
> ```java
> isValidTicketCode(String code)
> ```
>
> trả về kiểu `boolean`.

---

## Kết quả sinh mã lần đầu

```java
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

public class TicketValidator {

    public static boolean isValidTicketCode(String code) {
        if (code == null || code.trim().isEmpty()) {
            return false;
        }

        if (!code.startsWith("BUS-")) {
            return false;
        }

        String[] parts = code.split("-");
        if (parts.length != 3) {
            return false;
        }

        String cityCode = parts[1];
        if (!cityCode.matches("[A-Z]{2}")) {
            return false;
        }

        String datePart = parts[2];
        if (!datePart.matches("\\d{6}")) {
            return false;
        }

        SimpleDateFormat sdf = new SimpleDateFormat("yyMMdd");
        sdf.setLenient(false);

        try {
            Date travelDate = sdf.parse(datePart);

            if (travelDate.before(new Date())) {
                return false;
            }

        } catch (ParseException e) {
            return false;
        }

        return true;
    }
}
```

---

# Bước 2: Prompt kiểm chứng độc lập (AI Auditing Prompt)

## Prompt

> Bạn là một kỹ sư kiểm thử/kiểm toán bảo mật độc lập.
>
> Hãy rà soát mã nguồn lớp `TicketValidator` ở trên, phân tích và phản biện các lỗi logic tiềm ẩn.
>
> Ví dụ:
>
> * Cách phân tích ngày tháng có thể gặp lỗi khi chuyển giao năm mới.
> * Xử lý múi giờ hệ thống.
> * Lỗi `NullPointerException` khi cắt chuỗi.
>
> Sau đó đề xuất phiên bản tối ưu hơn bằng thư viện:
>
> ```java
> java.time.LocalDate
> ```
>
> để xử lý ngày tháng an toàn và chính xác.

---

# Kết quả kiểm chứng (AI Verifier)

Sau khi rà soát mã nguồn sinh tự động, hệ thống phát hiện các điểm cần cải thiện:

## 1. Rủi ro xử lý năm với `SimpleDateFormat("yyMMdd")`

`SimpleDateFormat` sử dụng năm 2 chữ số nên có thể ánh xạ sai thế kỷ khi dữ liệu nằm gần mốc chuyển năm.

Ví dụ:

```text
260101
```

có thể bị hiểu khác mong đợi tùy quy tắc nội bộ của Java.

---

## 2. So sánh thời gian chưa chính xác

Đoạn:

```java
travelDate.before(new Date())
```

so sánh cả giờ–phút–giây.

Điều này có thể khiến vé đi **hôm nay** bị đánh giá sai nếu thời điểm kiểm tra đã qua 00:00.

---

## 3. Phụ thuộc múi giờ hệ thống

`Date` và `SimpleDateFormat` chịu ảnh hưởng bởi timezone máy chạy.

Nếu server và người dùng khác múi giờ, kết quả kiểm tra có thể lệch.

---

## 4. Thiết kế API cũ

`Date` và `SimpleDateFormat` là API cũ của Java.

Khuyến nghị sử dụng:

```java
java.time.LocalDate
```

để xử lý ngày tháng rõ ràng hơn.

---

# Phiên bản tối ưu sau kiểm chứng

```java
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.time.format.DateTimeParseException;

public class TicketValidator {

    public static boolean isValidTicketCode(String code) {

        if (code == null || code.trim().isEmpty()) {
            return false;
        }

        if (!code.startsWith("BUS-")) {
            return false;
        }

        String[] parts = code.split("-");

        if (parts.length != 3) {
            return false;
        }

        String cityCode = parts[1];

        if (!cityCode.matches("[A-Z]{2}")) {
            return false;
        }

        String datePart = parts[2];

        if (!datePart.matches("\\d{6}")) {
            return false;
        }

        DateTimeFormatter formatter =
                DateTimeFormatter.ofPattern("yyMMdd");

        try {

            LocalDate travelDate =
                    LocalDate.parse(datePart, formatter);

            LocalDate today =
                    LocalDate.now();

            if (travelDate.isBefore(today)) {
                return false;
            }

        } catch (DateTimeParseException e) {
            return false;
        }

        return true;
    }
}
```

---

# Kết luận

Thiết kế quy trình **Code Generation + AI Verifier** giúp tăng độ tin cậy của mã nguồn sinh tự động.

Lợi ích đạt được:

* Tách biệt vai trò sinh mã và kiểm chứng.
* Giảm lỗi logic khó phát hiện.
* Hạn chế rủi ro do AI sinh mã thiếu kiểm tra biên.
* Nâng cao chất lượng đầu ra trước khi triển khai thực tế.

Mô hình này phù hợp cho các hệ thống có nghiệp vụ quan trọng như:

* Đặt vé trực tuyến.
* Thanh toán.
* Xác thực người dùng.
* Kiểm tra dữ liệu đầu vào.
