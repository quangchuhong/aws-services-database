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




#### 2.9 So sánh: RDS Multi‑AZ (instance), RDS Multi‑AZ DB Cluster và Aurora Cluster

| Thuộc tính                               | RDS Multi‑AZ (instance-based)                      | RDS Multi‑AZ DB Cluster (MySQL/PG)                        | Aurora Cluster (Aurora MySQL/PG)                    |
|------------------------------------------|----------------------------------------------------|-----------------------------------------------------------|-----------------------------------------------------|
| Kiểu triển khai                          | 1 Primary + 1 Standby                              | Cluster 1 writer + 2 reader (Multi‑AZ)                   | Cluster 1 writer + đến 15 replicas                  |
| Số AZ tham gia                           | 2 AZ (Primary + Standby)                           | 3 AZ (3 instances, shared storage kiểu mới)              | 3 AZ (shared storage 6 bản sao)                     |
| Standby/read node có đọc được không?     | **Không** – standby chỉ để failover                | **Có** – 2 reader có thể dùng để đọc                     | **Có** – tất cả Aurora Replicas đọc được            |
| Endpoint                                 | 1 endpoint chung (primary)                         | Cluster endpoint (writer) + reader endpoint              | Cluster endpoint (writer) + reader endpoint         |
| Failover                                 | Tự động, thường chậm hơn cluster                   | Tự động, nhanh hơn (giống Aurora‑like)                   | Tự động, rất nhanh                                  |
| Dùng để làm gì chính                     | HA trong 1 Region (không scale read)               | HA + limited read scaling (3 instance trong cluster)      | HA + scale read mạnh (tới 15 replicas)              |
| Engine                                   | RDS MySQL/PG/MariaDB/Oracle/SQL Server             | RDS MySQL, RDS PostgreSQL (phiên bản được hỗ trợ)        | Aurora MySQL, Aurora PostgreSQL                     |

> Nên hiểu:  
> - “Multi‑AZ” **không đồng nghĩa** với “cluster”, trừ kiểu **Multi‑AZ DB Cluster** mới.  
> - Aurora luôn là cluster Multi‑AZ theo thiết kế.


---

### 3. Automated Backups vs Manual Snapshots (RDS)

RDS cung cấp **2 cơ chế chính** để backup: **Automated Backups** và **Manual DB Snapshots**. Cả hai đều dùng cho **restore** nhưng khác nhau về cách tạo, thời gian giữ, và use case.

#### 3.1. Automated Backups

- Được **bật tự động** khi tạo RDS (trừ khi bạn tắt).
- Gồm:
  - **Daily backup** (full snapshot tự động).
  - **Transaction logs** để hỗ trợ **Point‑in‑Time Recovery (PITR)**.
- **Retention**: cấu hình từ **1–35 ngày**.
- Dùng để:
  - Khôi phục DB về **một thời điểm cụ thể** trong khoảng retention.
  - Tạo **DB mới** từ backup (không overwrite DB cũ).
- Lưu trữ trên S3 (ẩn với người dùng), thường **miễn phí** đến dung lượng bằng với dung lượng DB (vượt quá sẽ tính phí).

**Ví dụ (CLI – chỉnh retention & backup window):**

```bash
aws rds modify-db-instance \
  --db-instance-identifier my-rds-pg \
  --backup-retention-period 7 \
  --preferred-backup-window 02:00-03:00 \
  --apply-immediately
```

#### 3.2. Manual DB Snapshots

    - Bạn tự tay tạo, gọi là DB Snapshot.
    - Không bị xóa tự động – tồn tại cho đến khi bạn tự xóa.
    - Thích hợp cho:
        - Trước khi nâng cấp phiên bản DB.
        - Trước khi thay đổi schema lớn (migration, refactor).
        - Lưu trữ dài hạn phục vụ audit/compliance.
        - Copy snapshot cross‑Region để làm DR.

**Ví dụ (CLI – tạo và xóa snapshot):**
```bash
# Tạo snapshot thủ công
aws rds create-db-snapshot \
  --db-instance-identifier my-rds-pg \
  --db-snapshot-identifier my-rds-pg-pre-upgrade-2026-04-21

# Xóa snapshot khi không cần nữa
aws rds delete-db-snapshot \
  --db-snapshot-identifier my-rds-pg-pre-upgrade-2026-04-21

```

#### 3.3. Export Snapshot to S3

Export snapshot to S3 là chức năng:

    - Lấy dữ liệu từ một RDS snapshot (manual hoặc automated),
    - Chuyển đổi sang định dạng file trên S3 (thường là Parquet, đôi khi CSV/JSON tùy engine),
    - Để dùng cho analytics / data lake, không phải để restore trực tiếp thành RDS.

**Use case:**

    - Phân tích dữ liệu RDS bằng:
        - Athena, Glue, EMR, Spark, Redshift Spectrum, v.v.
    - Kết hợp dữ liệu DB với data lake trên S3.
    - Lưu trữ dữ liệu dạng file (archive) thay vì snapshot DB.

**Quan trọng:**

- Export từ snapshot ra S3 **KHÔNG** dùng trực tiếp để restore lại RDS.
- Nếu muốn “đưa ngược lại vào RDS”, bạn phải:
  1. Tạo RDS mới (hoặc tạo sẵn bảng trống).
  2. Dùng ETL (psql/mysql, lệnh `COPY`, AWS Glue, ứng dụng riêng…) để **load dữ liệu từ S3 vào RDS**.

**Ví dụ (CLI – start export task):**

```bash
aws rds start-export-task \
  --export-task-identifier export-my-rds-pg-2026-04-21 \
  --source-arn arn:aws:rds:ap-southeast-1:123456789012:snapshot:my-rds-pg-snap-2026-04-21 \
  --s3-bucket-name my-rds-exports \
  --iam-role-arn arn:aws:iam::123456789012:role/rds-s3-export-role \
  --kms-key-id arn:aws:kms:ap-southeast-1:123456789012:key/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```
---

#### 4. Cross-account restore từ RDS Snapshot

Để dùng một RDS snapshot ở **Account A** để restore DB ở **Account B**:

4.1. **Chỉ dùng được Manual Snapshot**  
   - Nếu đang có automated backup → tạo manual snapshot từ DB trước.

4.2. **Ở Account A (source)**  
   - Tạo manual snapshot (nếu chưa có):
     ```bash
     aws rds create-db-snapshot \
       --db-instance-identifier my-rds \
       --db-snapshot-identifier my-rds-share-to-accB
     ```
   - Share snapshot cho Account B (`111122223333`):
     ```bash
     aws rds modify-db-snapshot-attribute \
       --db-snapshot-identifier my-rds-share-to-accB \
       --attribute-name restore \
       --values-to-add 111122223333
     ```
   - Nếu snapshot **encrypted (KMS)**:  
     - Phải share **KMS CMK** cho Account B (chỉnh key policy / grant trong KMS).

4.3. **Ở Account B (target)**  
   - Vào RDS → Snapshots → tab **Shared with me** → chọn snapshot.  
   - Restore:
     ```bash
     aws rds restore-db-instance-from-db-snapshot \
       --db-instance-identifier restored-from-other-account \
       --db-snapshot-identifier my-rds-share-to-accB
     ```

> Lưu ý: cross-account share chỉ áp dụng cho **snapshot**, không chia sẻ trực tiếp “running instance”.

---

### 5. Bảo mật (Security Best Practices)

#### 5.1. Network & Access

- Đặt RDS trong **private subnet**, **không public Internet**
- **Security Group**:
  - Chỉ mở port DB cho:
    - App server SG
    - Bastion host SG (nếu cần)
  - Không mở 0.0.0.0/0 cho production

#### 5.2. Encryption

- **At rest**:
  - Bật khi tạo DB (KMS key)
  - Mã hóa: data file, backups, snapshots, logs
- **In transit**:
  - Dùng **SSL/TLS**
  - Bật `require SSL` / `rds.force_ssl` (tùy engine)
  - Cấu hình client/ORM kết nối qua SSL

#### 5.3. Authentication & Credentials

- Không dùng **master user** cho ứng dụng
- Tạo user riêng cho app, phân quyền **least privilege**
- Lưu credentials trong:
  - **AWS Secrets Manager**
    - Hỗ trợ **rotation tự động**

#### 5.4. IAM & Audit

- Dùng **IAM policies** để:
  - Giới hạn ai được create/modify/delete RDS
  - Giới hạn ai được copy/xóa snapshot
- Bật **CloudTrail**:
  - Theo dõi mọi API call liên quan RDS & KMS

#### 5.5. Best Practices tóm tắt

- Đặt RDS trong **private subnet**, không public Internet  
- Hạn chế port trong **Security Group**, chỉ cho phép từ app/bastion  
- Bật **encryption at rest** + **SSL/TLS in transit**  
- Không dùng **master user** cho app; tạo user riêng, phân quyền hạn chế  
- Lưu credentials trong **Secrets Manager** và **tự động rotate**  
- Bật **automated backups**, dùng **Multi-AZ** cho HA  
- Dùng **IAM** để giới hạn ai được thao tác với RDS & snapshot 

---

## 6. Monitoring & Query Tuning

### 6.1. Performance Insights

- Bật trên RDS/Aurora
- Cho biết:
  - DB load
  - Top SQL
  - Top waits / users / hosts
- Dùng để:
  - Tìm **query chậm/nặng**
  - Phân tích bottleneck

### 6.2. CloudWatch Metrics

- Giám sát:
  - CPU, Memory, IOPS, Disk Queue, Connections
- Khi bất thường:
  - Vào Performance Insights / DB logs để soi chi tiết

### 6.3. Slow Query Logs

**MySQL / MariaDB / Aurora MySQL**

- Parameter Group:
  ```ini
  slow_query_log = 1
  long_query_time = 1        # giây – chỉnh theo nhu cầu
  log_output = FILE
  ```

- Xem log:
    - RDS Console → DB → Logs & events
    - Hoặc đẩy sang CloudWatch Logs


**PostgreSQL / Aurora PostgreSQL**

    - Parameter Group:
    
    ```ini
    log_min_duration_statement = 1000   # ms
    log_connections = on
    log_disconnections = on
    ```
    
    - Nếu dùng pg_stat_statements:
    
    ```ini
    SELECT query, calls, total_time, mean_time
    FROM pg_stat_statements
    ORDER BY total_time DESC
    LIMIT 20;

    ```

### 7. Parameter quan trọng (MySQL/PostgreSQL)

#### 7.1. Nhóm kết nối

    - max_connections
    - wait_timeout / interactive_timeout (MySQL)
    - work_mem, max_connections (PostgreSQL)
    
#### 7.2. Logging

    - MySQL / Aurora MySQL:
        - slow_query_log, long_query_time, log_output
    - PostgreSQL / Aurora PG:
        - log_min_duration_statement
        - log_connections, log_disconnections
        - shared_preload_libraries = 'pg_stat_statements' (nếu dùng)
        
#### 7.3. SSL/TLS

    - MySQL:
        - require_secure_transport = ON (tùy version)
    - PostgreSQL:
        - rds.force_ssl = 1
        
---


### 8. Lỗi thường gặp từ phía ứng dụng

Một số lỗi “kinh điển” làm RDS quá tải / lỗi:

1. **Connection handling kém**

    - Không dùng connection pool
    - Connection leak → chạm max_connections
    - Connection idle quá nhiều / quá lâu

2. **Query & ORM không tối ưu**

    - SELECT * trên bảng lớn, không LIMIT
    - Thiếu index → full table scan
    - N+1 query (ORM)
    - Transaction giữ lâu → lock, deadlock
      
3. **Pattern đọc/ghi không hợp lý**

    - Không dùng cache (ElastiCache/DAX) cho dữ liệu đọc lặp
    - Ghi quá nhiều record nhỏ liên tục, không batch
      
4. **Migration / deploy thiếu cẩn trọng**

    - ALTER TABLE nặng giờ cao điểm
    - Script update lớn không chia batch
---

### 9. DMS Migration Oracle on‑prem → RDS PostgreSQL

Phần này tóm tắt **quy trình chuẩn** migrate từ **Oracle on‑prem** lên **Amazon RDS for PostgreSQL** bằng:

- **AWS Schema Conversion Tool (SCT)** – chuyển **schema** Oracle → PostgreSQL.
- **AWS Database Migration Service (DMS)** – chuyển **data** (full load + CDC).

---

#### 9.1. Kiến trúc tổng quan

```text
+--------------------+        VPN / Direct Connect       +----------------------+
| Oracle on-prem     |  <------------------------------> |  AWS VPC             |
| (Source DB)        |                                   |                      |
+---------+----------+                                   |  +----------------+  |
          |                                              |  | RDS PostgreSQL |  |
          |                                              |  | (Target DB)    |  |
          |                                              |  +--------+-------+  |
          |                                              |           ^          |
          v                                              |           |          |
   AWS SCT (Schema Conversion)                           |  +--------+-------+  |
                                                         |  | DMS Replication|  |
                                                         |  |   Instance     |  |
                                                         |  +----------------+  |
                                                         +----------------------+
```

#### 9.2. Bước 1 – Chuẩn bị Network & Quyền

**Network:**

- Thiết lập VPN site‑to‑site hoặc AWS Direct Connect từ on‑prem → VPC AWS.
- Đảm bảo:
    - DMS Replication Instance truy cập được:
        - Oracle on‑prem (port 1521).
        - RDS PostgreSQL (port 5432).
          
**Quyền trên Oracle:**

- Tạo user cho DMS với quyền tối thiểu:
    - CONNECT
    - SELECT trên các schema/tables cần migrate.
    - Quyền đọc redo/archivelog nếu dùng CDC (Change Data Capture).
      
**Quyền trên PostgreSQL:**

- User có quyền:
    - Tạo schema/table/index (nếu để DMS/SCT tạo).
    - Hoặc quyền INSERT/UPDATE/DELETE trên các bảng target.

#### 9.3. Bước 2 – Tạo RDS PostgreSQL Target

- Tạo RDS PostgreSQL (hoặc Aurora PostgreSQL) với:
    - Cỡ instance & storage đủ cho toàn bộ dữ liệu + tăng trưởng.
    - Bật:
        - Multi‑AZ nếu là production.
        - Automated Backups.

#### 9.4. Bước 3 – Convert Schema với AWS SCT

1. Cài AWS SCT trên máy client (Windows/Linux/Mac).
   
2. Trong SCT:
   
    - Tạo kết nối Source: Oracle on‑prem.
    - Tạo kết nối Target: RDS PostgreSQL.
      
3. Chạy Assessment Report:
    - SCT cho biết:
        - Bao nhiêu % object tự convert được.
        - Cái nào cần chỉnh tay (PL/SQL phức tạp, package, function…).
          
4. Convert schema:
   
    - Chọn schema Oracle → chuột phải → Convert schema.
    - Xem & chỉnh sửa mapping (data types, tên schema… nếu cần).
      
5. Apply schema lên RDS PostgreSQL:
   
    - Direct apply từ SCT, hoặc
    - Export SQL script → chạy bằng psql.
      
_Kết quả: RDS PostgreSQL có schema tương đương Oracle, sẵn sàng nhận dữ liệu._
