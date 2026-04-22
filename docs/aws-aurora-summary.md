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

---

## 3. Aurora Auto Scaling (Compute, Readers, Storage)

Aurora có 3 cơ chế “tự co giãn” khác nhau:

1. **Auto scaling compute** (Aurora Serverless).
2. **Auto scaling số lượng Aurora Replicas** (Reader Auto Scaling).
3. **Auto scaling storage** (luôn bật, tới 64 TB).


### 3.1. Auto Scaling Compute – Aurora Serverless

**Áp dụng cho:** Aurora Serverless v1/v2 (MySQL/PG).

- Bạn **không chọn instance size cố định**, mà chọn **range ACU** (Aurora Capacity Units), ví dụ: `minCapacity = 2`, `maxCapacity = 64`.
- Aurora tự động:
  - Tăng ACU khi tải (CPU, connections, throughput) tăng.
  - Giảm ACU khi tải giảm.
- Với v1: có thể **pause** khi idle để không tốn tiền.

**Workflow đơn giản:**

```text
        +---------------------------+
        |  Application Traffic      |
        +-------------+-------------+
                      |
                      v
             +----------------------+
             | Aurora Serverless    |
             |  (ACU auto-scale)    |
             +----------+-----------+
                        |
             Monitor load (CPU, conn,
             throughput, queue length)
                        |
        +---------------+----------------+
        |                                |
        v                                v
   Low load                       High load
(ACU giảm xuống min)          (ACU tăng tới max)

```

Ý chính:

- Bạn định nghĩa range capacity.
- Aurora tự scale bên trong range theo nhu cầu.

### 3.2. Auto Scaling Aurora Replicas (Reader Auto Scaling)

**Áp dụng cho:** Aurora provisioned (không phải Serverless).

- Bạn có 1 Writer + một vài Readers ban đầu.
- Cấu hình Aurora Auto Scaling (thường qua:
  - Application Auto Scaling + policy CloudWatch).
- Khi CPU / connections / replica lag trên readers cao:
  - Tự động thêm Aurora Replicas (tối đa N – mặc định upper limit bạn đặt, không vượt quá 15).
- Khi tải thấp:
  - Tự động giảm số replicas (scale‑in).
    
**Workflow đơn giản:**
```text
                 +-----------------------+
                 |   Application (Read)  |
                 +-----------+-----------+
                             |
                  Reader endpoint (cluster)
                             |
      +----------------------+------------------------+
      |                      |                        |
      v                      v                        v
+-------------+      +-------------+         +-------------+
| Reader 1    |      | Reader 2    |   ...   | Reader N    |
+------+------+      +------+------+         +------+------+
       \                    |                       /
        \                   |                      /
         \                  v                     /
          +--------------------------------------+
                          |
                          v
              Shared Storage (Aurora)

Aurora Auto Scaling:
- Theo dõi metrics (CPU, conn, lag) của reader pool.
- Khi vượt threshold: ADD Reader (tới max).
- Khi dưới threshold: REMOVE Reader (tới min).

```
Ý chính:

- Reader endpoint phân tải đọc giữa các replicas.
- Auto Scaling điều chỉnh số readers trong khoảng min–max bạn định nghĩa.

### 3.3. Auto Scaling Storage (Luôn bật, tới 64 TB)

Áp dụng cho: Mọi Aurora cluster (MySQL/PG, provisioned, serverless).

- Bạn không phải set trước 100 GB, 1 TB…
- Aurora tự tăng dung lượng storage khi dữ liệu lớn dần (đến 64 TB).
- Không cần downtime để “resize volume” như RDS thường.
  
Workflow đơn giản:
```text
        +------------------------+
        |  Aurora Cluster        |
        |  (Writer + Readers)    |
        +-----------+------------+
                    |
                    v
        +------------------------+
        | Shared Storage Layer   |
        |  (Auto grow to 64 TB)  |
        +-----------+------------+
                    ^
                    |
       Khi data size tiến gần ngưỡng hiện tại,
       Aurora tự động allocate thêm storage

```

Ý chính:

- Storage Aurora là “elastic”, tự co giãn, bạn chỉ trả tiền theo GB thực dùng.
- Không phải thao tác “modify DB” để tăng allocated storage.


### 3.4. Khi nào dùng loại Auto Scaling nào?

| Loại auto scaling                | Nên dùng khi                                                                                 | Không phù hợp khi                                                                                  | Ghi chú chính                                                                                      |
|----------------------------------|----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| **Auto scaling compute (Serverless)** | Workload khó đoán, không 24/7, dev/test, POC, batch theo giờ; muốn trả tiền đúng theo usage | Workload cực nhạy về latency, traffic luôn cao và ổn định, cần full control instance size        | Dùng **Aurora Serverless v1/v2**; cấu hình **min/max ACU**, Aurora tự scale compute trong khoảng  |
| **Auto scaling Readers (Replicas)**   | Prod read-heavy, traffic đọc thay đổi theo giờ/ngày; cần tăng/giảm **số reader** linh hoạt | Hầu như không có read load, hoặc read ổn định và thấp, không cần thêm/bớt reader theo thời gian  | Dùng với **Aurora provisioned**; cấu hình Aurora Auto Scaling cho **reader endpoint**             |
| **Auto scaling Storage** (mặc định)  | Mọi Aurora cluster: dữ liệu tăng dần, không muốn quản thủ công dung lượng từng volume      | Không có – đây là cơ chế built‑in, luôn bật, không tắt được                                       | Storage Aurora tự grow đến **64 TB**, bạn chỉ cần theo dõi chi phí **GB/tháng**                   |

---

## 4. Aurora Instance Types & Terraform Examples

### 4.1. DB Instance Type thường dùng cho Aurora

Aurora dùng các **instance type chuyên cho RDS/Aurora**, phổ biến:

- **Tổng quát (burstable, nhỏ/tiết kiệm)**  
  - `db.t3.small`, `db.t3.medium`  
  - `db.t4g.small`, `db.t4g.medium` (Graviton, rẻ & hiệu năng tốt)  
  → Dùng cho dev/test, POC, workload nhẹ.

- **Tổng quát (production, balanced)**  
  - `db.m5.large`, `db.m5.xlarge`, `db.m5.2xlarge`  
  - `db.m6g.large`, `db.m6g.xlarge`, `db.m6g.2xlarge` (Graviton – khuyến nghị mới)  
  → Dùng cho hầu hết workload production web/app.

- **Memory-optimized (nhiều RAM, query nặng, nhiều connection)**  
  - `db.r5.large`, `db.r5.xlarge`, `db.r5.2xlarge`  
  - `db.r6g.large`, `db.r6g.xlarge`, `db.r6g.2xlarge`  
  → Dùng cho report nặng, analytic OLAP nhẹ, nhiều session.

> Tip:  
> - Mới triển khai production: **bắt đầu từ `db.m6g.large` hoặc `db.r6g.large`**, rồi scale up/down dựa trên metrics.  
> - Dev/Test: `db.t3.small` / `db.t4g.small`.

---

### 4.2. Terraform – Aurora Serverless v1 (MySQL/PG)

Aurora Serverless v1 tạo bằng `aws_rds_cluster` với `engine_mode = "serverless"` + `scaling_configuration`.

Ví dụ: Aurora Serverless v1 – MySQL

```hcl
resource "aws_rds_cluster" "aurora_mysql_serverless_v1" {
  cluster_identifier      = "aurora-mysql-serverless-v1"
  engine                  = "aurora-mysql"
  engine_mode             = "serverless"
  engine_version          = "5.7.mysql_aurora.2.07.2" # kiểm tra version hỗ trợ serverless v1
  database_name           = "myappdb"
  master_username         = "masteruser"
  master_password         = "StrongPassw0rd!"
  backup_retention_period = 7
  preferred_backup_window = "02:00-03:00"
  storage_encrypted       = true
  kms_key_id              = aws_kms_key.rds.arn
  db_subnet_group_name    = aws_db_subnet_group.aurora.name
  vpc_security_group_ids  = [aws_security_group.aurora.id]

  scaling_configuration {
    auto_pause               = true
    min_capacity             = 2   # ACUs
    max_capacity             = 64  # ACUs
    seconds_until_auto_pause = 300 # 5 phút idle thì pause
  }

  tags = {
    Name = "aurora-mysql-serverless-v1"
  }
}
