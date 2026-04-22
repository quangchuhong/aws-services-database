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

```

App:
- Đọc mọi thứ liên quan user: PK = USER#<user_id>, query range SK.
- Đọc orders của user: PK + SK begins_with("ORDER#").

#### Giải thích thêm: 3.1. Mô hình 1 – Single Table Design cho Microservices

**Ý tưởng chính:**

- Chỉ dùng **một bảng DynamoDB duy nhất** (ví dụ: `app_main`).
- Trong bảng đó chứa **nhiều loại item khác nhau**:
  - USER, ORDER, SESSION, PRODUCT, v.v.
- Phân biệt từng loại item bằng:
  - Cách đặt **Partition key (PK)** / **Sort key (SK)** (dùng prefix), và/hoặc
  - Một field `type`.

**Ví dụ bảng `app_main` (một phần dữ liệu):**

```text
PK                | SK                 | type     | Các field khác
------------------+--------------------+----------+-------------------------
USER#u1           | PROFILE            | USER     | name, email, ...
USER#u1           | ORDER#o100         | ORDER    | amount, status, ...
USER#u1           | ORDER#o101         | ORDER    | amount, status, ...
USER#u1           | SESSION#sabc       | SESSION  | ip, expires_at, ...
USER#u2           | PROFILE            | USER     | name, email, ...
PRODUCT#p1        | META               | PRODUCT  | name, price, ...
PRODUCT#p2        | META               | PRODUCT  | name, price, ...

```

**Ưu điểm:**

  - Giảm số bảng.
  - Tối ưu query theo access pattern chuẩn.

**Access pattern ví dụ:**

- Lấy profile + tất cả order + session của user u1:
  - Query với PK = "USER#u1" → trả về mọi item của user đó (PROFILE, ORDER#, SESSION#) trong 1 lần query.
- Lấy toàn bộ order của user u1:
  - Query PK = "USER#u1" + SK begins_with("ORDER#").
    
**So với RDBMS:**

- SQL truyền thống thường: users, orders, sessions, products là 4 bảng khác nhau.
- DynamoDB single table design: gom chúng vào 1 table, tận dụng PK/SK để:
  - Giảm số query.
  - Tránh join phức tạp (DynamoDB không hỗ trợ join như SQL).
    

### 3.2. Mô hình 2 – Bảng chuyên cho 1 loại entity (Multi-Table)

**Use case:** Đơn giản, mỗi loại entity 1 bảng: USERS, ORDERS, PRODUCTS, …
```text
Table USERS
- PK: user_id

Table ORDERS
- PK: order_id
- GSI: user_id + order_date (để query đơn theo user)

Table PRODUCTS
- PK: product_id

```
**Ưu điểm:**

  - Dễ hiểu với người quen RDBMS.
  - Phù hợp hệ thống đơn giản, ít access pattern phức tạp.

### 3.3. Mô hình 3 – DynamoDB + DAX (DynamoDB Accelerator)

**Use case:** Read-heavy, muốn latency micro‑second & giảm chi phí read.

```text
       +------------------------+
       |      Clients           |
       +------------+-----------+
                    |
                    v
             +--------------+
             | Application  |
             +------+-------+
                    |
            DAX client SDK
                    |
         +----------+----------+
         |  DAX Cluster        |  (in-memory cache, API-compatible)
         +----------+----------+
                    |
                    v
             +--------------+
             | DynamoDB     |
             | Table(s)     |
             +--------------+

Flow:
- App gọi DAX endpoint thay vì DynamoDB trực tiếp.
- DAX:
  - Cache get item / query.
  - Miss → đọc DynamoDB → cache → trả kết quả.

```
### 3.4. Mô hình 4 – DynamoDB Global Tables (Multi-Region Active-Active)

**Use case:** Ứng dụng global, user ở nhiều châu lục, cần multi‑Region active‑active.
```text
Region A (us-east-1)              Region B (eu-west-1)
----------------------            ----------------------
+------------------+              +------------------+
| Table: users     | <==========> | Table: users     |
| (replica region) |   Replicate  | (replica region) |
+------------------+   2 chiều    +------------------+

- App ở Region A đọc/ghi bảng users trong Region A.
- App ở Region B đọc/ghi bảng users trong Region B.
- DynamoDB Global Tables tự replicate dữ liệu 2 chiều.

```

### 3.5. Khi nào dùng mô hình nào?

| Bối cảnh / Yêu cầu                                                | Mô hình gợi ý                    | Ghi chú ngắn                                                                              |
|-------------------------------------------------------------------|----------------------------------|-------------------------------------------------------------------------------------------|
| Ứng dụng mới, nhiều entity liên quan, muốn tối ưu read/query      | **3.1 – Single Table Design**    | Cần đầu tư thiết kế key & access pattern kỹ; đổi mindset từ RDBMS sang “query-first”      |
| Hệ thống đơn giản, gần giống RDBMS (1 entity = 1 bảng)           | **3.2 – Multi-Table**            | Dễ hiểu, dễ migrate từ SQL; nhưng sẽ hạn chế nếu access pattern phức tạp về sau           |
| Read-heavy, latency cần rất thấp, chi phí read cao                | **3.3 – DynamoDB + DAX**         | DAX cache cho các GetItem/Query lặp lại, giảm RCU & giảm độ trễ                           |
| Ứng dụng global, multi‑Region active‑active                       | **3.4 – DynamoDB Global Tables** | Mỗi Region có bảng replica, đọc/ghi local; phù hợp game, API global, fintech đa khu vực  |

---

## 4. Capacity & Pricing: Provisioned vs On-Demand

### 4.1. RCU & WCU

- RCU (Read Capacity Unit):
  - 1 RCU = 1 strongly consistent read/giây cho item 4 KB hoặc 2 eventually consistent reads/giây (4 KB).
- WCU (Write Capacity Unit):
 - 1 WCU = 1 write/giây cho item 1 KB.

### 4.2. Provisioned Capacity

- Bạn đặt sẵn:
  - Read capacity (RCU).
  - Write capacity (WCU).
- Có thể bật Auto Scaling:
  - Tự tăng/giảm RCU/WCU theo CloudWatch.
    
**Dùng khi:**

- Biết tương đối ổn định workload.
- Muốn tối ưu chi phí (so với On‑Demand khi traffic cao).

### 4.3. On-Demand Capacity

- Không set RCU/WCU.
- Trả tiền theo số request thực (read/write).
- Dùng khi:
  - Traffic khó đoán, burst thất thường.

---

## 5. Time to Live (TTL) & Dọn dẹp dữ liệu

- TTL = attribute dạng timestamp (epoch) trên item (ví dụ expires_at).
- Sau thời điểm TTL, DynamoDB tự động xóa item (background).
- Thường dùng cho:
  - Session, token, cache, log cũ, data tạm.
- Nếu bật DynamoDB Streams, item bị xóa bởi TTL có thể xuất hiện trong stream.
  
---

## 6. Secondary Indexes (LSI & GSI)

### 6.1. LSI – Local Secondary Index

- Cùng Partition Key với bảng gốc.
- Sort key khác.
- Phải tạo khi tạo bảng, không thêm xóa sau được.
- Dùng cho:
  - Query các view khác nhau trên cùng 1 PK.
    
### 6.2. GSI – Global Secondary Index

- Có Partition Key (và Sort Key) riêng.
- Có thể tạo/xóa sau khi bảng đã tồn tại.
- Dùng cho:
  - Query theo “chiều” khác, ví dụ:
    - Bảng chính PK=user_id, GSI PK=email.
---

## 7. DynamoDB Streams

- Ghi lại chuỗi thay đổi trên bảng:
  - INSERT, MODIFY, REMOVE.
- Lưu tối đa 24 giờ.
- Có thể cấu hình NEW_IMAGE, OLD_IMAGE, v.v.
- Use case:
  - Trigger AWS Lambda theo event.
  - Đồng bộ dữ liệu sang:
    - S3, Elasticsearch/OpenSearch, RDS, Redshift.
  - Là nền cho DynamoDB Global Tables.
---

## 8. DynamoDB Accelerator (DAX)

- DAX: in-memory cache được thiết kế riêng cho DynamoDB.
- API-compatible: app chỉ đổi endpoint (DAX endpoint thay DynamoDB).
- Tính năng:
  - Micro‑second read latency.
  - Multi‑AZ cluster, replication.
- Use case:
  - Read-heavy, nhiều GetItem/Query lặp lại.
  - Muốn giảm RCU & latency mà không thay đổi logic cache nhiều.

**DAX vs ElastiCache:**



| Tiêu chí              | DAX                                          | ElastiCache (Redis/Memcached)                          |
|-----------------------|----------------------------------------------|--------------------------------------------------------|
| Tích hợp với DynamoDB | **Rất chặt** – API-compatible với DynamoDB   | Không; bạn phải tự thiết kế key + logic cache         |
| Cách dùng             | Đổi endpoint (DynamoDB → DAX) + dùng DAX client SDK | Dùng như cache tổng quát phía trước DB/API bất kỳ |
| Use case chính        | Tăng tốc DynamoDB (GetItem/Query)           | Cache cho RDS, DynamoDB, HTTP API, session, leaderboard… |
| Logic cache           | Ít phải tự code; DAX xử lý nhiều chi tiết   | Tự implement cache-aside / write-through / write-back  |
| Data model            | Theo DynamoDB (item/document)               | Key–value, data structures Redis (list, set, hash…)   |
