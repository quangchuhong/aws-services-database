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
```
_Lưu ý: Aurora Serverless v1 không tạo aws_rds_cluster_instance, chỉ có aws_rds_cluster._


### 4.3. Terraform – Aurora Serverless v2 (MySQL/PG)

Aurora Serverless v2 dùng engine_mode = "provisioned" nhưng cấu hình scaling_configuration trên từng instance.

Ví dụ: Aurora Serverless v2 – Aurora PostgreSQL

```hcl
resource "aws_rds_cluster" "aurora_pg_serverless_v2" {
  cluster_identifier      = "aurora-pg-serverless-v2"
  engine                  = "aurora-postgresql"
  engine_version          = "15.3" # kiểm tra version hỗ trợ serverless v2
  database_name           = "myappdb"
  master_username         = "masteruser"
  master_password         = "StrongPassw0rd!"
  backup_retention_period = 7
  preferred_backup_window = "02:00-03:00"
  storage_encrypted       = true
  kms_key_id              = aws_kms_key.rds.arn
  db_subnet_group_name    = aws_db_subnet_group.aurora.name
  vpc_security_group_ids  = [aws_security_group.aurora.id]

  tags = {
    Name = "aurora-pg-serverless-v2"
  }
}

resource "aws_rds_cluster_instance" "aurora_pg_serverless_v2_instance" {
  count                = 1 # 1 writer; thêm reader bằng tăng count hoặc tạo resource khác với different instance_class
  identifier           = "aurora-pg-slv2-${count.index}"
  cluster_identifier   = aws_rds_cluster.aurora_pg_serverless_v2.id
  engine               = aws_rds_cluster.aurora_pg_serverless_v2.engine
  engine_version       = aws_rds_cluster.aurora_pg_serverless_v2.engine_version
  instance_class       = "db.serverless" # kiểu đặc biệt cho serverless v2
  publicly_accessible  = false

  # cấu hình auto scaling compute cho serverless v2
  serverlessv2_scaling_configuration {
    min_capacity = 0.5  # ACUs
    max_capacity = 8.0  # ACUs
  }

  tags = {
    Name = "aurora-pg-serverless-v2-instance-${count.index}"
  }
}

```

_Với serverless v2: _ 

- Dùng instance_class = "db.serverless".
- Dùng block serverlessv2_scaling_configuration trên aws_rds_cluster_instance.


### 4.4. Terraform – Aurora Provisioned Cluster (Writer + Readers)

Ví dụ: Aurora MySQL provisioned, 1 writer + 2 readers.
```hcl
resource "aws_rds_cluster" "aurora_mysql" {
  cluster_identifier      = "aurora-mysql-cluster"
  engine                  = "aurora-mysql"
  engine_version          = "5.7.mysql_aurora.2.11.2"
  database_name           = "myappdb"
  master_username         = "masteruser"
  master_password         = "StrongPassw0rd!"
  backup_retention_period = 7
  preferred_backup_window = "02:00-03:00"
  storage_encrypted       = true
  kms_key_id              = aws_kms_key.rds.arn
  db_subnet_group_name    = aws_db_subnet_group.aurora.name
  vpc_security_group_ids  = [aws_security_group.aurora.id]

  tags = {
    Name = "aurora-mysql-cluster"
  }
}

# Writer
resource "aws_rds_cluster_instance" "aurora_mysql_writer" {
  identifier         = "aurora-mysql-writer-0"
  cluster_identifier = aws_rds_cluster.aurora_mysql.id
  engine             = aws_rds_cluster.aurora_mysql.engine
  engine_version     = aws_rds_cluster.aurora_mysql.engine_version
  instance_class     = "db.m6g.large"
  publicly_accessible = false

  tags = {
    Role = "writer"
  }
}

# Reader 1
resource "aws_rds_cluster_instance" "aurora_mysql_reader_1" {
  identifier         = "aurora-mysql-reader-1"
  cluster_identifier = aws_rds_cluster.aurora_mysql.id
  engine             = aws_rds_cluster.aurora_mysql.engine
  engine_version     = aws_rds_cluster.aurora_mysql.engine_version
  instance_class     = "db.m6g.large"
  publicly_accessible = false

  tags = {
    Role = "reader"
  }
}

# Reader 2
resource "aws_rds_cluster_instance" "aurora_mysql_reader_2" {
  identifier         = "aurora-mysql-reader-2"
  cluster_identifier = aws_rds_cluster.aurora_mysql.id
  engine             = aws_rds_cluster.aurora_mysql.engine
  engine_version     = aws_rds_cluster.aurora_mysql.engine_version
  instance_class     = "db.m6g.large"
  publicly_accessible = false

  tags = {
    Role = "reader"
  }
}

```

_Thực tế bạn có thể dùng count để tạo nhiều readers hơn, hoặc tách resource nếu muốn gán tag/setting riêng._


Tạo nhiều Aurora Readers bằng `count` trong Terraform

```hcl
# Aurora MySQL cluster (giữ nguyên như trước)
resource "aws_rds_cluster" "aurora_mysql" {
  cluster_identifier      = "aurora-mysql-cluster"
  engine                  = "aurora-mysql"
  engine_version          = "5.7.mysql_aurora.2.11.2"
  database_name           = "myappdb"
  master_username         = "masteruser"
  master_password         = "StrongPassw0rd!"
  backup_retention_period = 7
  preferred_backup_window = "02:00-03:00"
  storage_encrypted       = true
  kms_key_id              = aws_kms_key.rds.arn
  db_subnet_group_name    = aws_db_subnet_group.aurora.name
  vpc_security_group_ids  = [aws_security_group.aurora.id]

  tags = {
    Name = "aurora-mysql-cluster"
  }
}

# Writer (1 instance)
resource "aws_rds_cluster_instance" "aurora_mysql_writer" {
  identifier          = "aurora-mysql-writer-0"
  cluster_identifier  = aws_rds_cluster.aurora_mysql.id
  engine              = aws_rds_cluster.aurora_mysql.engine
  engine_version      = aws_rds_cluster.aurora_mysql.engine_version
  instance_class      = "db.m6g.large"
  publicly_accessible = false

  tags = {
    Role = "writer"
  }
}

# Readers (dùng count)
variable "aurora_reader_count" {
  type    = number
  default = 2
}

resource "aws_rds_cluster_instance" "aurora_mysql_reader" {
  count              = var.aurora_reader_count
  identifier         = "aurora-mysql-reader-${count.index}"
  cluster_identifier = aws_rds_cluster.aurora_mysql.id
  engine             = aws_rds_cluster.aurora_mysql.engine
  engine_version     = aws_rds_cluster.aurora_mysql.engine_version
  instance_class     = "db.m6g.large"
  publicly_accessible = false

  tags = {
    Role  = "reader"
    Index = count.index
  }
}

```
---

### 5. Aurora Storage, Backups & Fault Tolerance

#### 5.1. Aurora Storage & Fault Tolerance

- Storage layer tự động:
  - Scale tới 64 TB/cluster.
  - 6 copies dữ liệu trên 3 AZ (2 bản/AZ).
- Chịu lỗi:
  - Mất 2 bản sao vẫn không mất khả năng ghi.
  - Mất 3 bản sao vẫn đọc được.
- Compute instances (writer/reader) stateless về storage → failover nhanh.
  
#### 5.2. Backups & PITR

- Continuous backup to S3:
  - Dùng để:
    - Point‑in‑Time Recovery (PITR) trong retention (giống RDS).
- Bạn cấu hình:
  - Backup retention period: 1–35 ngày.
- Có thể tạo manual snapshots Aurora để:
  - Giữ dài hạn.
  - Copy cross‑Region, cross‑account.
  - Restore cluster mới từ snapshot.

#### 5.3 Aurora (MySQL & PostgreSQL) hỗ trợ Export snapshot to S3 tương tự RDS:

- Bạn chọn Aurora snapshot (manual hoặc automated).
- Dùng tính năng Start export task để:
  - Đọc dữ liệu trong snapshot.
  - Ghi ra S3 (thường dạng Parquet, chia theo database/schema/table/partition).
- Mục đích:
  - Phân tích bằng Athena, Glue, EMR, Redshift Spectrum.
  - Đưa dữ liệu Aurora vào data lake trên S3.
- **Không dùng trực tiếp để restore Aurora**; muốn “quay lại Aurora”, phải load lại bằng ETL.

Ví dụ CLI:
```bash
aws rds start-export-task \
  --export-task-identifier export-aurora-mysql-2026-04-21 \
  --source-arn arn:aws:rds:ap-southeast-1:123456789012:snapshot:my-aurora-snap-2026-04-21 \
  --s3-bucket-name my-aurora-exports \
  --iam-role-arn arn:aws:iam::123456789012:role/rds-s3-export-role \
  --kms-key-id arn:aws:kms:ap-southeast-1:123456789012:key/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

```

### 6. Security Best Practices (Aurora)

Giống RDS, nhưng nhớ là Aurora luôn **chạy trong cluster**:

- **Network:**
  - Private subnet, không public.
  - SG chỉ mở cho app/bastion SG.
- **Encryption:**
  - At rest: bật KMS khi tạo cluster.
  - In transit: SSL/TLS, cấu hình client dùng SSL; ép SSL qua parameter (rds.force_ssl cho PG).
- **User & quyền:**
  - Không dùng master user cho app.
  - Tạo user riêng từng app/service; phân quyền theo schema.
- **Secrets Manager:**
  - Lưu & auto‑rotate password.
- **Audit & IAM:**
  - CloudTrail cho API Aurora/RDS.
  - IAM để kiểm soát ai được tạo/modify/delete cluster & snapshot.

---

### 7. Monitoring & Tuning cho Aurora

- **Performance Insights:**
  - Bật cho writer/reader.
  - Xem DB load, top SQL, waits.
- **CloudWatch:**
  - CPU, connections, disk queue, replica lag, freeable memory.
- **Query tuning:**
  - Aurora MySQL:
    - slow query log, performance_schema, sys schema.
- Aurora PostgreSQL:
  - pg_stat_statements, log_min_duration_statement, autovacuum tuning.
---

## 8. Parameter quan trọng cho Aurora (MySQL / PostgreSQL)

Aurora cũng dùng **Parameter Group**, nhưng tách 2 loại:

- **Cluster Parameter Group**: áp dụng cho **cả cluster** (settings cấp logical DB/engine).
- **Instance Parameter Group**: áp dụng cho **từng instance** (writer/reader).

Khi chỉnh parameter, luôn tạo **custom parameter group**, không sửa default.

---

### 8.1. Aurora MySQL – Parameter thường dùng

#### 8.1.1. Kết nối & session

*Cluster hoặc Instance Parameter Group (tùy tham số)*

- `max_connections`  
  - Giới hạn số connection; phải khớp với tổng connection pool app.
- `wait_timeout`, `interactive_timeout`  
  - Thời gian idle trước khi đóng connection.
- `max_allowed_packet`  
  - Kích thước tối đa 1 gói (dùng cho BLOB/Large result).

#### 8.1.2. InnoDB & memory

(Aurora tự tối ưu nhiều, thường không cần chỉnh nhiều như MySQL on‑prem, nhưng nên biết:)

- `innodb_buffer_pool_size` (đa phần auto, nhưng là thành phần chính cho cache dữ liệu).
- `innodb_log_file_size`, `innodb_flush_log_at_trx_commit`  
  - Ảnh hưởng hiệu năng ghi & durability (thường để mặc định nếu không có lý do cụ thể).

#### 8.1.3. Logging & slow query

- `slow_query_log = 1`
- `long_query_time = 1` (giây – chỉnh theo nhu cầu, ví dụ 0.5–2s)
- `log_output = FILE`

> Sau khi bật, đọc log trong **RDS Console → Logs & events** hoặc CloudWatch Logs.

#### 8.1.4. SSL/TLS

- `require_secure_transport = ON`  
  - Ép client dùng SSL khi kết nối.

---

### 8.2. Aurora PostgreSQL – Parameter thường dùng

#### 8.2.1. Kết nối & memory

- `max_connections`  
  - Đặt theo tổng connection pool app, tránh quá cao gây thiếu RAM.
- `work_mem`  
  - Bộ nhớ cho sort/hash mỗi operation; quá thấp → nhiều disk spill; quá cao → OOM khi nhiều session.
- `maintenance_work_mem`  
  - Dùng cho VACUUM, CREATE INDEX; đặt cao hơn `work_mem`.

#### 8.2.2. Shared buffers & cache

(Aurora PG cũng auto‑tune nhiều; chỉ nên chỉnh khi hiểu rõ)

- `shared_buffers`  
  - Thường ~25% RAM ở PG truyền thống; với Aurora PG, AWS đã tối ưu, ít chỉnh tay.
- `effective_cache_size`  
  - Gợi ý cho planner về size cache hiệu dụng.

#### 8.2.3. Logging & `pg_stat_statements`

**Instance Parameter Group:**

- `log_min_duration_statement = 1000`  (log query chạy > 1s)
- `log_connections = on`
- `log_disconnections = on`

**Bật extension `pg_stat_statements` (Cluster/Instance tùy version):**

- `shared_preload_libraries = 'pg_stat_statements'`

Sau khi restart:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT query,
       calls,
       round(total_time::numeric, 2) AS total_ms,
       round(mean_time::numeric, 2)  AS mean_ms
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 20;
```

#### 8.2.4. SSL/TLS

- rds.force_ssl = 1
  - Ép tất cả session dùng SSL.
    
---

### 8.3. Aurora – Cluster vs Instance Parameter Group

- **Aurora Cluster Parameter Group:**
  - Tham số áp dụng cho toàn cluster (ví dụ: logical replication, some engine settings).
**Aurora DB Parameter Group (Instance):**
  - Tham số áp dụng cho từng instance (writer/reader):
    - max_connections, work_mem, logging, SSL, v.v.
      
**Best practice:**

  1. Tạo 1 Cluster Parameter Group cho cluster Aurora.
  2. Tạo 1 (hoặc vài) Instance Parameter Group:
    - Ví dụ:
      - aurora-pg-writer-params
      - aurora-pg-reader-params (nếu muốn khác, nhưng đa số dùng chung).
  3. Gán đúng:
    - Cluster → Cluster Parameter Group.
    - Mỗi instance → Instance Parameter Group tương ứng.
  4. Khi chỉnh tham số:
    - Kiểm tra xem cần reboot (apply after reboot) hay dynamic (apply immediately).
---

### 8.4. Gợi ý bộ parameter “baseline” cho Aurora

**Aurora MySQL (baseline gợi ý)**

  - Cluster / Instance PG:
    - slow_query_log = 1
    - long_query_time = 1
    - log_output = FILE
    - require_secure_transport = ON
  - Tuỳ app:
    - max_connections theo connection pool.
      
**Aurora PostgreSQL (baseline gợi ý)**

  - Instance PG:
    - log_min_duration_statement = 1000
    - log_connections = on
    - log_disconnections = on
    - shared_preload_libraries = 'pg_stat_statements'
    - rds.force_ssl = 1
  - Sau đó bật extension pg_stat_statements trong DB.
    
_Bạn có thể tạo file riêng examples/aurora-params-baseline.md để lưu lại các PG mẫu cho MySQL/PG, và link từ README này._

---

### 9. Migration lên Aurora

Các đường đi phổ biến:

  - MySQL RDS / MySQL on‑prem → Aurora MySQL
  - PostgreSQL RDS / on‑prem → Aurora PostgreSQL
  - Oracle / SQL Server → Aurora MySQL/PG:
    - Dùng SCT + DMS giống RDS.
      
Chiến lược tiêu chuẩn:

  1. Chuẩn bị schema (convert nếu heterogenous).
  2. Dùng DMS Full load + CDC.
  3. Cutover với downtime thấp.
---

