# AWS DynamoDB – Summary, Architecture Models & Examples

Tài liệu tóm tắt & ví dụ cụ thể về **Amazon DynamoDB**, dùng làm `README.md` trên GitHub.  
Phong cách giống README RDS/Aurora/ElastiCache bạn đã làm.

---

## 1. Amazon DynamoDB Overview

**Amazon DynamoDB** là dịch vụ **NoSQL key–value & document**, fully managed, serverless:

- Scale gần như không giới hạn.
- **Single-digit millisecond latency** ở mọi scale.
- Tự động:
  - Partition dữ liệu.
  - Replicate Multi‑AZ trong 1 Region.
  - Quản lý hạ tầng (server, patch, backup tích hợp).

Bạn tập trung vào:

- Thiết kế **bảng, partition key, sort key**.
- Thiết kế **access pattern** (query như thế nào).
- Chọn **capacity mode** (Provisioned vs On‑Demand).

---

## 2. Kiến trúc & khái niệm lõi

### 2.1. Cấu trúc cơ bản

- **Table** → chứa các **Item (bản ghi)** → mỗi item có **Attributes (cột)**.
- Item không cần cùng schema cứng (NoSQL, schema flexible).
- **Primary key**:
  - **Partition key** (hash key)  
    → Ví dụ: `user_id`.
  - Hoặc **Partition key + Sort key** (composite)  
    → Ví dụ: `user_id` + `order_id`.

### 2.2. Partition & Scaling

- DynamoDB tự chia bảng thành nhiều **partition** dựa trên:
  - Giá trị partition key.
  - Throughput (RCU/WCU) & dung lượng.
- Thiết kế **partition key tốt** là cực quan trọng:
  - Tránh “hot partition” (tất cả request dồn vào 1 key).

---

## 3. Mô hình kiến trúc DynamoDB phổ biến

### 3.1. Mô hình 1 – Single Table Design cho Microservices

**Use case:** Một domain (hoặc cả system) dùng **một bảng DynamoDB**, lưu nhiều loại entity, dùng sort key + type field để phân biệt.

```text
Table: app_main

Partition key (PK) | Sort key (SK)      | Type        | Các attribute khác
-------------------+--------------------+------------+----------------------
USER#<user_id>     | PROFILE            | USER       | name, email, ...
USER#<user_id>     | ORDER#<order_id>   | ORDER      | amount, status, ...
USER#<user_id>     | SESSION#<session>  | SESSION    | expires_at, ip, ...
PRODUCT#<sku>      | META               | PRODUCT    | name, price, ...
...

App:
- Đọc mọi thứ liên quan user: PK = USER#<user_id>, query range SK.
- Đọc orders của user: PK + SK begins_with("ORDER#").
