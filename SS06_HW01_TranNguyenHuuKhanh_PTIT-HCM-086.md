# BÀI 1: Thực hành Tối ưu hóa Lưu trữ Giỏ hàng


## 1. Phân tích bối cảnh

Giỏ hàng tạm thời là dữ liệu có tần suất đọc/ghi rất cao, thay đổi liên tục khi người dùng thêm, xóa hoặc cập nhật số lượng sản phẩm. Nếu mọi thao tác đều ghi trực tiếp xuống SQL Database, hệ thống dễ bị nghẽn do I/O, lock, transaction và độ trễ truy vấn. Vì vậy cần thiết kế prompt đa chiều để AI không chỉ đưa ra một giải pháp đơn lẻ, mà phải so sánh nhiều phương án, phân tích trade-off, xử lý kịch bản lỗi và đề xuất cấu hình triển khai an toàn.

## 2. Prompt tối ưu

```text
Bạn là System Architect cấp cao cho hệ thống thương mại điện tử SpeedyCart.

Bối cảnh:
Shopping Cart là dữ liệu tạm thời, có tần suất đọc/ghi rất lớn khi người dùng thêm, bớt hoặc thay đổi số lượng sản phẩm. Ghi trực tiếp toàn bộ thao tác vào SQL Database đang làm nghẽn hệ thống và tăng độ trễ.

Mục tiêu:
Hãy thiết kế giải pháp lưu trữ tối ưu cho dữ liệu giỏ hàng tạm thời.

Yêu cầu bắt buộc:
1. Role: Đóng vai System Architect.
2. Context: Phân tích bài toán giỏ hàng tạm thời trong hệ thống thương mại điện tử có traffic cao.
3. Task:
   - Đề xuất ít nhất 3 phương án công nghệ lưu trữ giỏ hàng, ví dụ SQL Database, Redis In-memory Cache, Client-side Session/Cookie hoặc phương án kết hợp.
   - Lập bảng so sánh chi tiết theo các tiêu chí: tốc độ đọc/ghi, tính đồng nhất dữ liệu, khả năng chịu lỗi/mất dữ liệu, độ phức tạp triển khai.
   - Phân tích What-if Scenario: Nếu cụm Redis bị crash hoặc mất nguồn điện thì chuyện gì xảy ra? Làm sao để giỏ hàng không biến mất hoàn toàn?
   - Đề xuất cấu hình Redis Persistence RDB/AOF kết hợp fallback ghi tạm xuống SQL.
   - Cung cấp mã cấu hình Java Spring Boot RedisTemplate an toàn.
4. Format:
   - Trình bày có tiêu đề rõ ràng.
   - Có bảng so sánh Markdown.
   - Có phần khuyến nghị kiến trúc.
   - Có code Java hoàn chỉnh.
5. Constraints:
   - Ưu tiên giải pháp hiệu năng cao, có fallback, có TTL cho cart, serialize JSON an toàn, không dùng Java native serialization.
```

## 3. Minh chứng chạy thực tế - Text log phản hồi của AI

```markdown
# Thiết kế lưu trữ Shopping Cart cho SpeedyCart

## 1. Các phương án công nghệ

### Phương án 1: SQL Database
Lưu cart trực tiếp vào bảng `shopping_cart` và `shopping_cart_item`. Phù hợp nếu traffic thấp hoặc cần nhất quán mạnh, nhưng không tối ưu cho thao tác đọc/ghi liên tục.

### Phương án 2: Redis In-memory Cache
Lưu cart theo key như `cart:{customerId}` hoặc `cart:{sessionId}`. Redis có tốc độ đọc/ghi rất cao vì dữ liệu nằm trong bộ nhớ. Đây là lựa chọn phù hợp cho dữ liệu tạm thời có TTL.

### Phương án 3: Client-side Session/Cookie
Lưu thông tin cart ở phía client. Giảm tải server nhưng bị giới hạn dung lượng, rủi ro bị chỉnh sửa dữ liệu nếu không ký hoặc mã hóa.

### Phương án 4: Hybrid Redis + SQL Fallback
Redis là storage chính cho cart tạm thời, SQL dùng để snapshot/fallback hoặc phục hồi khi Redis gặp sự cố. Đây là phương án cân bằng giữa hiệu năng và an toàn dữ liệu.

## 2. Bảng so sánh

| Phương án | Tốc độ đọc/ghi | Tính đồng nhất dữ liệu | Khả năng chịu lỗi/mất dữ liệu | Độ phức tạp triển khai |
|---|---:|---|---|---|
| SQL Database | Trung bình đến chậm khi tải cao | Cao, hỗ trợ ACID | Cao nếu có backup/replication | Thấp đến trung bình |
| Redis In-memory Cache | Rất nhanh | Eventual consistency nếu đồng bộ SQL sau | Trung bình, phụ thuộc RDB/AOF/replication | Trung bình |
| Client-side Cookie/Session | Nhanh, giảm tải backend | Thấp, dễ lệch dữ liệu nếu client sửa | Thấp đến trung bình | Thấp |
| Redis + SQL Fallback | Rất nhanh ở luồng chính | Khá cao nếu có chiến lược sync hợp lý | Cao hơn Redis đơn lẻ | Cao hơn do cần fallback/sync |

## 3. What-if Scenario: Redis crash hoặc mất nguồn

Nếu Redis crash mà không bật persistence, dữ liệu cart đang nằm trong RAM có thể bị mất. Với shopping cart, mất dữ liệu gây trải nghiệm xấu vì khách hàng phải chọn lại sản phẩm.

Giải pháp đề xuất:

1. Bật Redis Persistence:
   - RDB: tạo snapshot định kỳ, phục hồi nhanh nhưng có thể mất dữ liệu giữa hai lần snapshot.
   - AOF: ghi lại từng thao tác write, độ an toàn dữ liệu tốt hơn.
   - Khuyến nghị dùng AOF `appendfsync everysec` để cân bằng hiệu năng và độ an toàn.

2. Cấu hình Redis Replication/Sentinel hoặc Redis Cluster:
   - Có replica để failover khi master lỗi.
   - Sentinel/Cluster giúp tự động chuyển node.

3. Fallback xuống SQL:
   - Khi Redis write lỗi, ghi tạm cart xuống SQL bảng `cart_fallback`.
   - Có background job đồng bộ ngược lại Redis khi Redis hoạt động bình thường.
   - Khi đọc cart: ưu tiên Redis, nếu miss thì đọc SQL fallback.

4. TTL:
   - Key cart nên có TTL, ví dụ 7 ngày hoặc 30 ngày.
   - Tránh Redis bị đầy bộ nhớ bởi cart cũ.

Ví dụ cấu hình Redis persistence:

```conf
appendonly yes
appendfsync everysec
save 900 1
save 300 10
save 60 10000
```

## 4. Kiến trúc khuyến nghị

Luồng ghi cart:

1. User cập nhật cart.
2. Backend ghi vào Redis với TTL.
3. Nếu Redis lỗi, ghi fallback xuống SQL.
4. Background worker đồng bộ SQL fallback về Redis.

Luồng đọc cart:

1. Đọc từ Redis.
2. Nếu Redis miss, đọc từ SQL fallback.
3. Nếu có dữ liệu SQL, hydrate lại Redis.

## 5. Java Spring Boot RedisTemplate an toàn

```java
package com.speedycart.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(new JavaTimeModule());
        objectMapper.activateDefaultTyping(
                objectMapper.getPolymorphicTypeValidator(),
                ObjectMapper.DefaultTyping.NON_FINAL,
                JsonTypeInfo.As.PROPERTY
        );

        GenericJackson2JsonRedisSerializer jsonSerializer =
                new GenericJackson2JsonRedisSerializer(objectMapper);
        StringRedisSerializer stringSerializer = new StringRedisSerializer();

        template.setKeySerializer(stringSerializer);
        template.setHashKeySerializer(stringSerializer);
        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);
        template.afterPropertiesSet();

        return template;
    }
}
```

## 6. Kết luận

Với SpeedyCart, giải pháp tốt nhất là Redis + SQL Fallback. Redis xử lý nhanh các thao tác đọc/ghi giỏ hàng, còn SQL fallback và Redis persistence giúp giảm rủi ro mất dữ liệu khi có sự cố.
```
