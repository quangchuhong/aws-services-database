# AWS RDS – Summary, Architecture Models & Examples

Tài liệu tóm tắt & ví dụ cụ thể về AWS RDS, dùng làm `README.md` trên GitHub.

---

## 1. Amazon RDS Overview

**Amazon RDS (Relational Database Service)** là dịch vụ cơ sở dữ liệu quan hệ managed trên AWS.

AWS chịu trách nhiệm:

- Provisioning, cài đặt & cấu hình DB
- Backup tự động, snapshot
- Patching OS & engine DB
- Monitoring cơ bản (CloudWatch)
- Failover tự động (với Multi-AZ)

Bạn tập trung vào:

- Thiết kế schema & index
- Viết & tối ưu query
- Tối ưu ứng dụng

### 1.1. Database engines được hỗ trợ

- **Amazon Aurora**
- **MySQL**
- **MariaDB**
- **PostgreSQL**
- **Oracle**
- **Microsoft SQL Server**

---

## 2. Các mô hình kiến trúc RDS (áp dụng cho mọi engine)

Các mô hình dưới đây áp dụng cho hầu hết engine RDS (MySQL, MariaDB, PostgreSQL, Oracle, SQL Server).  
Chi tiết hỗ trợ (ví dụ: cross‑Region read replica) tùy engine/version.


### 2.1. Mô hình 1 – Single-AZ, Single Instance

**Use case:** Dev/Test, non‑critical, chi phí thấp.

- 1 instance, 1 AZ.
- Không standby, không read replica.

Luồng đơn giản:

```text
Application
    |
    v
+-------------------------+
|  RDS DB Instance        |
|  (PostgreSQL / MySQL…) |
|  Single-AZ              |
+-------------------------+
            |
            v
   Automated Backups (S3)
```

### 2.2. Mô hình 2 – Multi-AZ (High Availability)

**Use case:** Production cần HA trong 1 Region, RTO thấp.

- 1 Primary (read/write) + 1 Standby (ẩn) ở AZ khác.
- Replication synchronous.
- Standby không dùng để đọc.

Luồng đơn giản:

```text
               +----------------------+
               |   Application        |
               +----------+-----------+
                          |
                          v
                  (DB Endpoint chung)
                          |
                          v
        +-----------------+-----------------+
        |                                   |
        v                                   v
+--------------------+             +--------------------+
| Primary RDS DB     |  <=====>    | Standby RDS DB     |
| (AZ-A, Read/Write) |  Sync repl  | (AZ-B, Failover)   |
+--------------------+             +--------------------+
                                           |
                                           v
                                Automated Backups (S3)
```

### 2.3. Mô hình 3 – Multi-AZ + Read Replicas (Read Scaling + HA)

**Use case:** Production, read-heavy trong 1 Region.

Multi-AZ cho HA.

Nhiều Read Replicas để scale đọc.

```text
                    +----------------------+
                    |     Application      |
                    +----------+-----------+
                               |
         +---------------------+----------------------+
         |                     |                      |
         v                     v                      v
+--------------------+  +--------------------+  +--------------------+
|  Primary RDS DB    |  | Read Replica 1     |  | Read Replica 2     |
|  (Multi-AZ, RW)    |  | (AZ-B, Read-only)  |  | (AZ-C, Read-only)  |
+----------+---------+  +--------------------+  +--------------------+
           |
           |  Synchronous
           v  replication (Multi-AZ)
+--------------------+
| Standby RDS DB     |
| (AZ-B, Failover)   |
+--------------------+
           |
           v
  Automated Backups (S3)

```

### 2.4. Mô hình 4 – Cross-Region Read Replicas (DR & Global Read)

**Use case:** DR cấp Region, low‑latency read cho user nhiều Region.

(Engine hỗ trợ: MySQL/PostgreSQL/MariaDB, Aurora…)

```text
                 Region A (Primary)
+--------------------+
| Primary RDS DB     |
| (Multi-AZ optional)|
+---------+----------+
          |   Async replication
          |   (cross-Region)
          v
   -------------------------------
   |                             |
   v                             v

Region B (Replica)        Region C (Replica)
+--------------------+    +--------------------+
| Read Replica (B)   |    | Read Replica (C)   |
| Read-only          |    | Read-only          |
+----------+---------+    +----------+---------+
           ^                         ^
           |                         |
   Users / Apps in B         Users / Apps in C

```

### 2.5. Mô hình 5 – Offload Read/Analytics với Read Replica + BI/ETL

Use case: Tách OLTP và reporting/ETL để giảm tải primary.

```text
          +---------------------+
          |   OLTP Application  |
          +----------+----------+
                     |
                     v
             +---------------+
             | Primary DB    |
             | (RDS engine)  |
             +-------+-------+
                     |
                     |  Async replication
                     v
             +---------------+
             | Read Replica  |
             | (Read-only)   |
             +-------+-------+
                     |
          +----------+-------------------------------+
          |                                          |
          v                                          v
 +--------------------+                  +------------------------+
 | BI / Reporting     |                  | ETL / Glue / Lambda    |
 | Dashboards, tools  |                  | (export to S3/Redshift)|
 +--------------------+                  +------------------------+

```

### 2.6. Mô hình 6 – RDS + ElastiCache (Cache-Aside)

**Use case:** Giảm latency & tải đọc cho RDS (mọi engine).
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
+-------------+            +--------------------+
|  Cache      |            |   RDS Database     |
| (Redis/Mem) |            | (MySQL/PG/...)     |
+------+------+            +---------+----------+
       ^                             |
       |                             v
       |                     Automated Backups (S3)
       |
 Notes:
 1. App đọc: Cache -> (miss) -> DB -> ghi vào Cache.
 2. App ghi: DB (tùy logic có thể update/invalid cache).

```

### 2.7. Mô hình 7 – Nhiều RDS trong Microservices Architecture

**Use case:** Mỗi service có DB riêng (có thể là engine khác nhau).

```text
        +----------------+         +----------------+
        | Order Service  |         | User Service   |
        +--------+-------+         +--------+-------+
                 |                          |
                 v                          v
        +----------------+         +----------------+
        | orders_db      |         | users_db       |
        | (RDS MySQL)    |         | (RDS Postgres) |
        +----------------+         +----------------+

        +----------------+ 
        | Reporting Svc  |
        +--------+-------+
                 |
                 v
        +----------------+
        | reporting_db   |
        | (RDS SQLServer)|
        +----------------+

Nguyên tắc:
- Mỗi service own DB riêng.
- Không cho service khác truy cập DB trực tiếp (giao tiếp qua API/message).

```

### 2.8. Khi nào chọn mô hình nào?

| Yêu cầu / Bối cảnh                                                                 | Mô hình gợi ý                                                | Ghi chú ngắn                                                                                 |
|------------------------------------------------------------------------------------|--------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| Dev/Test, môi trường lab, POC, chi phí thấp                                       | **2.1 – Single-AZ, Single Instance**                        | Đơn giản, rẻ, không HA; chấp nhận downtime khi instance/AZ gặp sự cố                        |
| Production nhỏ, yêu cầu sẵn sàng (HA) trong cùng 1 Region                          | **2.2 – Multi-AZ**                                          | Tự động failover AZ; không scale read; phù hợp OLTP phổ thông                               |
| Production, read-heavy trong 1 Region (nhiều báo cáo / dashboard / API đọc nhiều) | **2.3 – Multi-AZ + Read Replicas**                          | Multi-AZ cho HA, replicas để chia tải đọc                                                   |
| Cần DR cấp Region, hoặc người dùng ở nhiều châu lục cần đọc nhanh                  | **2.4 – Cross-Region Read Replicas**                        | Primary 1 Region, replicas Region khác; DR manual (promote replica khi Region chính lỗi)    |
| Muốn tách OLTP và Analytics/ETL để query nặng không ảnh hưởng giao dịch            | **2.5 – Read Replica cho BI/ETL**                           | Chạy báo cáo, ETL, truy vấn nặng trên replica                                               |
| Ứng dụng đọc rất nhiều, dữ liệu ít thay đổi, cần latency thấp (ms → sub‑ms)       | **2.6 – RDS + ElastiCache (Cache-Aside)**                    | RDS làm nguồn dữ liệu chính, cache phía trước để giảm tải & tăng tốc độ                     |
| Kiến trúc microservices, nhiều domain nghiệp vụ khác nhau                          | **2.7 – Nhiều RDS riêng cho từng service (polyglot)**       | Mỗi service own DB riêng (có thể khác engine), giảm coupling, dễ scale & triển khai độc lập |
| Hệ thống đã dùng Aurora, cần HA + scale read trong 1 Region                        | Aurora cluster (Writer + Aurora Replicas, tương tự 2.2–2.3) | Kiến trúc tương tự Multi-AZ + Read Replicas nhưng dùng Aurora-specific features             |



