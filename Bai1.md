# Bài 1: Thực hành Tích hợp & Sinh Code - Import giao dịch JSON

## 1. Phân tích các trường hợp biên có thể xảy ra khi đọc chuỗi JSON từ hệ thống cũ

Khi đọc dữ liệu giao dịch từ hệ thống Core Banking cũ, các lỗi thường xuất hiện ở các lớp sau:

- JSON rỗng hoặc null
  - Chuỗi đầu vào là empty string, whitespace hoặc null.
- JSON sai định dạng
  - Thiếu dấu ngoặc, dấu phẩy, dấu nháy, hoặc có ký tự không hợp lệ.
- Cấu trúc JSON sai
  - Không phải mảng giao dịch, mà là object đơn lẻ hoặc chuỗi thông thường.
- Dữ liệu dòng giao dịch bị thiếu trường bắt buộc
  - `id` null, rỗng hoặc chỉ chứa khoảng trắng.
  - `amount` null hoặc không tồn tại.
  - `status` null hoặc rỗng.
  - `transactionDate` null hoặc không đúng định dạng.
- Dữ liệu sai kiểu
  - `amount` là chuỗi chữ như `"abc"`, `"-10"`, `"10.5a"`.
  - `id` là số thay vì chuỗi.
  - `status` là giá trị không nằm trong `SUCCESS` hoặc `FAILED`.
- Dữ liệu logic không hợp lệ
  - Số tiền âm hoặc bằng 0.
  - Ngày tháng không parse được thành `LocalDateTime`.
- Dữ liệu ngoại lệ runtime
  - Jackson không parse được do schema thay đổi.

Kết luận: parser nên bỏ qua dòng lỗi, ghi log cảnh báo và tiếp tục xử lý các dòng còn lại thay vì dừng toàn bộ tiến trình.

---

## 2. Prompt tối ưu do tôi thiết kế

```text
Bạn là Senior Java Developer chuyên về Spring Boot và xử lý dữ liệu nhập từ hệ thống legacy.

Nhiệm vụ của bạn là sinh mã nguồn cho class TransactionParser để đọc một chuỗi JSON thô đại diện cho danh sách giao dịch và trả về List<TransactionDTO>.

Yêu cầu đầu vào:
- Input: một chuỗi JSON raw string chứa danh sách giao dịch.
- Mỗi giao dịch có các trường: id, amount, status, transactionDate.
- Dữ liệu có thể chứa lỗi biên như: id rỗng, amount là chữ, amount âm, status không hợp lệ, transactionDate sai định dạng.

Yêu cầu đầu ra:
- Trả về List<TransactionDTO>.
- TransactionDTO là Java record gồm các trường: id, amount, status, localDateTime.
- amount nên dùng BigDecimal để tránh sai số.
- localDateTime nên dùng LocalDateTime.

Ràng buộc kỹ thuật:
1. Sử dụng thư viện Jackson ObjectMapper của Spring Boot để parse JSON.
2. Cài đặt xử lý dữ liệu biên mạnh mẽ:
   - Nếu id rỗng hoặc null -> bỏ qua dòng đó.
   - Nếu amount <= 0 hoặc không parse được -> bỏ qua dòng đó.
   - Nếu status khác SUCCESS hoặc FAILED -> bỏ qua dòng đó.
   - Nếu transactionDate không hợp lệ -> bỏ qua dòng đó.
3. Khi phát hiện lỗi, ghi log cảnh báo bằng Logger, nhưng không làm sập toàn bộ quá trình đọc.
4. Nếu chuỗi JSON null, rỗng hoặc parse lỗi -> trả về danh sách rỗng.
5. Chỉ sinh mã Java 17 sạch, dễ đọc, production-ready.

Yêu cầu dry-run CoT:
- Trước khi viết code, hãy trình bày luồng suy luận từng bước theo kiểu CoT:
  1. Kiểm tra đầu vào có null/rỗng không.
  2. Parse JSON thành JsonNode hoặc DTO trung gian.
  3. Duyệt từng giao dịch trong mảng.
  4. Validate từng trường và bỏ qua dòng lỗi.
  5. Chuyển đổi dữ liệu hợp lệ sang TransactionDTO.
  6. Trả về danh sách đã chuẩn hóa.

Định dạng đầu ra:
- Phần 1: Phác thảo thuật toán bằng bullet points.
- Phần 2: Mã nguồn Java cho class TransactionParser.
- Phần 3: Mã nguồn Java cho record TransactionDTO.
- Không cần thêm giải thích dài dòng bên ngoài.
```

---

## 3. Minh chứng chạy thực tế

### Log phản hồi AI (mô phỏng)

```text
Prompt:
Bạn là Senior Java Developer chuyên về Spring Boot và xử lý dữ liệu nhập từ hệ thống legacy.
...

Phác thảo thuật toán:
1. Kiểm tra rawJson null hoặc blank, trả về danh sách rỗng nếu không hợp lệ.
2. Dùng ObjectMapper để đọc chuỗi JSON thành JsonNode.
3. Nếu root không phải mảng JSON thì trả về danh sách rỗng.
4. Duyệt từng phần tử giao dịch.
5. Với mỗi dòng, kiểm tra id, amount, status, transactionDate.
6. Nếu dữ liệu hợp lệ thì tạo TransactionDTO và thêm vào kết quả.
7. Nếu dữ liệu lỗi thì ghi log warning và bỏ qua dòng đó.
8. Trả về danh sách đã lọc.

Mã nguồn Java:
```java
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.time.format.DateTimeParseException;
import java.util.ArrayList;
import java.util.List;
import java.util.Locale;

public class TransactionParser {

    private static final Logger LOGGER = LoggerFactory.getLogger(TransactionParser.class);
    private final ObjectMapper objectMapper;

    public TransactionParser() {
        this(new ObjectMapper().registerModule(new JavaTimeModule()));
    }

    public TransactionParser(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    public List<TransactionDTO> parse(String rawJson) {
        if (rawJson == null || rawJson.isBlank()) {
            return List.of();
        }

        try {
            JsonNode root = objectMapper.readTree(rawJson);
            if (root == null || !root.isArray()) {
                LOGGER.warn("Input JSON must be an array of transactions.");
                return List.of();
            }

            List<TransactionDTO> transactions = new ArrayList<>();
            for (JsonNode node : root) {
                if (node == null || node.isNull()) {
                    continue;
                }

                try {
                    String id = readText(node, "id");
                    String amountText = readText(node, "amount");
                    String status = readText(node, "status");
                    String transactionDateText = readText(node, "transactionDate");

                    if (id == null || id.isBlank()) {
                        throw new IllegalArgumentException("id is empty");
                    }

                    BigDecimal amount = parseAmount(amountText);
                    if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
                        throw new IllegalArgumentException("amount must be > 0");
                    }

                    String normalizedStatus = normalizeStatus(status);
                    LocalDateTime localDateTime = parseDate(transactionDateText);

                    transactions.add(new TransactionDTO(id, amount, normalizedStatus, localDateTime));
                } catch (Exception ex) {
                    LOGGER.warn("Skipping invalid transaction row: {}", ex.getMessage());
                }
            }

            return transactions;
        } catch (JsonProcessingException ex) {
            LOGGER.error("Failed to parse raw JSON", ex);
            return List.of();
        }
    }

    private String readText(JsonNode node, String fieldName) {
        JsonNode field = node.get(fieldName);
        return field == null || field.isNull() ? null : field.asText();
    }

    private BigDecimal parseAmount(String amountText) {
        if (amountText == null || amountText.isBlank()) {
            throw new IllegalArgumentException("amount is missing");
        }
        try {
            return new BigDecimal(amountText.trim().replace(",", "."));
        } catch (NumberFormatException ex) {
            throw new IllegalArgumentException("amount is not numeric", ex);
        }
    }

    private String normalizeStatus(String status) {
        if (status == null || status.isBlank()) {
            throw new IllegalArgumentException("status is missing");
        }

        String normalized = status.trim().toUpperCase(Locale.ROOT);
        if (!"SUCCESS".equals(normalized) && !"FAILED".equals(normalized)) {
            throw new IllegalArgumentException("status is invalid");
        }
        return normalized;
    }

    private LocalDateTime parseDate(String transactionDateText) {
        if (transactionDateText == null || transactionDateText.isBlank()) {
            throw new IllegalArgumentException("transactionDate is missing");
        }

        try {
            return LocalDateTime.parse(transactionDateText, DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        } catch (DateTimeParseException ex) {
            try {
                return LocalDateTime.parse(transactionDateText, DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
            } catch (DateTimeParseException innerEx) {
                throw new IllegalArgumentException("transactionDate format is invalid", innerEx);
            }
        }
    }
}
```

```java
public record TransactionDTO(
        String id,
        BigDecimal amount,
        String status,
        LocalDateTime localDateTime
) {
}
```

