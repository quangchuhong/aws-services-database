# AWS Aurora – Summary, Architecture Models & Examples

Tài liệu tóm tắt & ví dụ cụ thể về **Amazon Aurora**, dùng làm `README.md` trên GitHub.  
Cấu trúc tương tự README của RDS nhưng tập trung vào **Aurora MySQL / Aurora PostgreSQL**.

---

## 1. Amazon Aurora Overview

**Amazon Aurora** là engine cơ sở dữ liệu quan hệ do AWS phát triển:

- Tương thích **MySQL** và **PostgreSQL** (API, driver, SQL).
- Được thiết kế cho:
  - **Hiệu năng cao** (3x PostgreSQL, 5x MySQL managed – theo AWS, tùy workload).
  - **Khả năng mở rộng & HA** tích hợp sẵn.
- Kiến trúc tách:
  - **Compute** (Aurora instances: writer/reader).
  - **Storage** (storage layer phân tán, tự scale).

AWS chịu trách nhiệm:

- Quản lý **cluster**, **replicas**, storage phân tán nhiều AZ.
- Backup liên tục ra S3, Point‑in‑Time Recovery.
- Patching, monitoring cơ bản, failover tự động.

Bạn tập trung vào:

- Thiết kế schema/index.
- Viết & tối ưu query.
- Thiết kế kiến trúc **writer/reader**, global, serverless.

### 1.1. Các biến thể Aurora

- **Aurora MySQL-Compatible Edition**
- **Aurora PostgreSQL-Compatible Edition**
- Một số tính năng đặc biệt:
  - Aurora Replicas (tối đa 15, in‑region).
  - Aurora Global Database (multi‑region).
  - Aurora Serverless (v1, v2 – auto scaling compute).

---

## 2. Các mô hình kiến trúc Aurora

### 2.1. Mô hình 1 – Aurora Single-Writer, 1 Reader (In-Region)

**Use case:** Prod nhỏ hoặc staging, cần HA cơ bản + ít scale đọc.

- 1 **Writer** (read/write).
- 1 **Reader** (read-only).
- Cả 2 dùng chung **distributed storage** (6 copies / 3 AZ).

```text
           +----------------------+
           |     Application      |
           +----------+-----------+
                      |
          +-----------+----------+
          |                      |
          v                      v
+-----------------+     +------------------+
| Aurora Writer   |     | Aurora Reader 1  |
| (Read/Write)    |     | (Read-only)      |
+--------+--------+     +---------+--------+
         \                    /
          \                  /
           v                v
        +----------------------+
        | Shared Storage Layer |
        | 6 copies / 3 AZs     |
        +----------------------+
                    |
                    v
          Continuous Backups (S3)
```

### 2.2. Mô hình 2 – Aurora Cluster: 1 Writer + N Readers (Read Scaling + HA in-Region)

**Use case:** Production, read-heavy trong 1 Region, yêu cầu HA cao.

- 1 Writer.
- Tối đa 15 Aurora Replicas (Readers) trong cùng Region.
- Tất cả share storage layer Multi‑AZ.
- Failover:
  - Khi writer lỗi → 1 reader được promote thành writer.
    
```text
                    +----------------------+
                    |     Application      |
                    +----------+-----------+
                               |
         +---------------------+-------------------------------+
         |                     |                  ...          |
         v                     v                               v
+-----------------+   +------------------+            +------------------+
| Aurora Writer   |   | Aurora Reader 1  |   ...      | Aurora Reader N  |
| (Read/Write)    |   | (Read-only)      |            | (Read-only)      |
+--------+--------+   +---------+--------+            +---------+--------+
         \                    |                               /
          \                   |                              /
           \                  v                             /
            +----------------------------------------------+
            |
            v
     +----------------------+
     | Shared Storage Layer |
     | 6 copies / 3 AZs     |
     +----------------------+
                    |
                    v
          Continuous Backups (S3)

```

### 2.3. Mô hình 3 – Aurora Global Database (Multi-Region, Global Read + DR)

**Use case:** Ứng dụng global, user ở nhiều châu lục; cần đọc nhanh & DR cấp Region.

- 1 Primary Region:
  - 1 Writer + (optional) Readers.
- 1+ Secondary Regions:
  - Chỉ chứa Readers (read‑only).
- Replication:
  - Fast cross‑Region (dưới 1 giây trong điều kiện lý tưởng – theo AWS).
- DR:
  - Có thể promote secondary Region thành primary mới khi Region chính lỗi.
 
```text
Region 1 (Primary)                     Region 2 (Secondary)
---------------------                  -----------------------
+-------------------+                  +-------------------+
| Writer + Readers  |   Async (fast)   | Readers only      |
| (Read/Write)      |  replication     | (Read-only)       |
+---------+---------+  ----------->    +---------+---------+
          |                                      |
          v                                      v
   Shared Storage 1                       Shared Storage 2
     (3 AZs)                                 (3 AZs)

Users gần Region 1  --> đọc/ghi Region 1
Users gần Region 2  --> đọc Region 2 (latency thấp)

```
### 2.4. Mô hình 4 – Aurora Serverless (v1/v2) – On-Demand Auto-Scaling

**Use case:** Workload không ổn định, dev/test, POC, hoặc app ít dùng.

- **Aurora Serverless v1:**
  - Aurora MySQL/PG với ACU (Aurora Capacity Units) auto scale.
  - Không phải chọn instance size cố định.
  - Có thể pause khi idle (v1).
  - Không có read replicas riêng, không public IP.
- **Aurora Serverless v2:**
  - Scale mượt hơn, granularity nhỏ, dùng cho production tốt hơn.
```text
        +----------------------+
        |     Application      |
        +----------+-----------+
                   |
                   v
          +----------------------+
          | Aurora Serverless    |
          | (Auto-scale ACU)     |
          +----------+-----------+
                     |
                     v
          Shared Storage Layer
            + Backups to S3

```

### 2.5. Mô hình 5 – Aurora + ElastiCache (Cache-Aside)

**Use case:** Dù Aurora nhanh, vẫn cần cache đọc rất nhiều/ít thay đổi → giảm latency & chi phí.

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
      +-------------+--------------+
      |                            |
      v                            v
+-------------+            +----------------------+
|  Cache      |            | Aurora Cluster       |
| (Redis/Mem) |            | (Writer + Readers)   |
+------+------+            +----------+-----------+
       ^                               |
       |                               v
       |                       Shared Storage Layer
 Notes:
 1. App đọc: Cache -> (miss) -> Aurora -> ghi vào Cache.
 2. App ghi: ghi Aurora, tùy logic update/invalid cache.

```

### 2.6. Mô hình 6 – Aurora trong Microservices Architecture

**Use case:** Một số service dùng Aurora (MySQL/PG), service khác có thể dùng RDS/DynamoDB.
```text
        +----------------+         +----------------+
        | Billing Svc    |         | Account Svc    |
        +--------+-------+         +--------+-------+
                 |                          |
                 v                          v
        +----------------+         +----------------+
        | billing_db     |         | account_db     |
        | (Aurora MySQL) |         | (Aurora PG)    |
        +----------------+         +----------------+

        +----------------+ 
        | Reporting Svc  |
        +--------+-------+
                 |
                 v
        +----------------+
        | reporting_db   |
        | (Aurora Reader |
        |  + BI/ETL)     |
        +----------------+

Nguyên tắc:
- Mỗi service own DB riêng.
- Giao tiếp qua API/message, không cross‑DB trực tiếp.

```

### 2.7. Khi nào chọn mô hình Aurora nào?

| Yêu cầu / Bối cảnh                                                                    | Mô hình gợi ý                                              | Ghi chú ngắn                                                                                             |
|---------------------------------------------------------------------------------------|------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Prod nhỏ/trung bình, cần HA + 1 reader                                                | **2.1 – Aurora 1 Writer + 1 Reader**                      | Đơn giản, ít cost hơn cluster lớn; vẫn có HA & tách read/write cơ bản                                    |
| Prod read-heavy trong 1 Region, cần scale đọc lớn                                     | **2.2 – Aurora 1 Writer + N Readers**                     | Tối đa 15 Aurora Replicas, dùng reader endpoint để scale read                                            |
| Ứng dụng global, user nhiều châu lục, cần DR Region‑level & low‑latency read         | **2.3 – Aurora Global Database**                          | 1 primary Region, 1+ secondary Regions; promote secondary khi DR                                        |
| Workload không ổn định, dev/test, POC, hoặc app ít dùng                               | **2.4 – Aurora Serverless** (v1/v2)                       | Không cần chọn instance size, auto scale ACU; phù hợp khi traffic khó đoán hoặc không chạy 24/7         |
| Đọc rất nhiều, dữ liệu tương đối ổn định, muốn giảm latency & chi phí Aurora queries | **2.5 – Aurora + ElastiCache (Cache-Aside)**              | Aurora là source of truth; Redis/Memcached đứng trước để cache kết quả đọc lặp lại                       |
| Kiến trúc microservices, vài domain cần Aurora (tính năng/hiệu năng)                  | **2.6 – Microservices + nhiều Aurora cluster riêng**      | Mỗi service own cluster riêng (Aurora MySQL/PG), dễ scale độc lập, giảm coupling giữa domain            |
