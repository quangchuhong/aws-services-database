# AWS RDS – Summary & Architecture Workflows

Tài liệu tóm tắt lại những nội dung AWS RDS mà chúng ta đã trao đổi, dùng làm README.md trên GitHub để ôn tập / tham khảo nhanh.

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

## 2. Scaling & High Availability

### 2.1. Scaling Up (Vertical Scaling)

Tăng tài nguyên cho **một** instance:

- Tăng **instance class** (CPU, RAM)
- Tăng **storage** (GB) & **IOPS** (nếu dùng gp3/io1)
- Một số thay đổi cần **reboot** → downtime ngắn

### 2.2. Scaling Out (Horizontal Scaling)

Thêm nhiều instance để chia tải:

- **Read Replicas**:
  - Primary: nhận **read + write**
  - Read replicas: **read-only**
  - Phù hợp **read-heavy workload**, báo cáo, analytics nhẹ
- Aurora:
  - Tối đa **15 Aurora Replicas** trong 1 cluster (read scaling + failover target)

### 2.3. Disaster Recovery (DR) & High Availability

**Multi-AZ Deployments**

- 1 **Primary** (read/write) + 1 **Standby** (chỉ để failover) ở **AZ khác trong cùng Region**
- Replication **synchronous**
- **Automatic failover** khi AZ/instance primary gặp sự cố
- Automated backups được thực hiện từ **standby**
- Dùng cho **High Availability (HA)** ở cấp độ AZ

**Read Replicas**

- Replication **asynchronous**
- Tất cả replica **đều đọc được**
- Có thể:
  - **Cross-AZ**
  - **Cross-Region** (một số engine)
- Dùng để:
  - **Scale read**
  - **DR thủ công** (promote replica thành standalone)

### 2.4. Bảng so sánh Multi-AZ vs Read Replicas

| Feature | Multi-AZ Deployments | Read Replicas |
|--------|----------------------|---------------|
| Replication | Synchronous (durable) | Asynchronous (scalable) |
| Instance trạng thái | Chỉ primary active | Tất cả replicas đọc được |
| Backup | Lấy từ standby | Không bật sẵn, phải cấu hình trên từng instance (nếu hỗ trợ) |
| Phạm vi | Luôn 2 AZ trong 1 Region | Có thể same AZ, cross-AZ, cross-Region |
| Engine upgrade | Diễn ra trên primary rồi replicate sang standby | Upgrade độc lập trên replica |
| Failover | Automatic failover | Manual promotion |

---

## 3. Backup & Maintenance

### 3.1. Automated Backups

- Bật tự động khi tạo RDS (trừ khi tắt)
- Gồm:
  - Daily snapshot
  - Transaction logs → **Point-in-Time Recovery (PITR)**
- Retention: **1–35 ngày**
- Restore:
  - Tạo **DB mới** tại thời điểm chọn trong retention

### 3.2. Manual Snapshots

- Bạn chủ động tạo (DB Snapshots)
- Giữ **vô thời hạn** cho đến khi xóa
- Use cases:
  - Trước khi nâng cấp version DB
  - Trước thay đổi schema lớn
  - Lưu trữ dài hạn, compliance
  - Copy snapshot **cross-Region** cho DR

### 3.3. Maintenance Windows

- Khung giờ để AWS:
  - Áp dụng patch OS/DB
  - Reboot khi cần
- Bạn nên chọn:
  - **Giờ ít traffic** (vd: 2–4 AM)
- Tùy chọn apply change:
  - **Apply immediately**
  - **Apply during next maintenance window**

---

## 4. Bảo mật (Security Best Practices)

### 4.1. Network & Access

- Đặt RDS trong **private subnet**, **không public Internet**
- **Security Group**:
  - Chỉ mở port DB cho:
    - App server SG
    - Bastion host SG (nếu cần)
  - Không mở 0.0.0.0/0 cho production

### 4.2. Encryption

- **At rest**:
  - Bật khi tạo DB (KMS key)
  - Mã hóa: data file, backups, snapshots, logs
- **In transit**:
  - Dùng **SSL/TLS**
  - Bật `require SSL` / `rds.force_ssl` (tùy engine)
  - Cấu hình client/ORM kết nối qua SSL

### 4.3. Authentication & Credentials

- Không dùng **master user** cho ứng dụng
- Tạo user riêng cho app, phân quyền **least privilege**
- Lưu credentials trong:
  - **AWS Secrets Manager**
    - Hỗ trợ **rotation tự động**

### 4.4. IAM & Audit

- Dùng **IAM policies** để:
  - Giới hạn ai được create/modify/delete RDS
  - Giới hạn ai được copy/xóa snapshot
- Bật **CloudTrail**:
  - Theo dõi mọi API call liên quan RDS & KMS

### 4.5. Best Practices tóm tắt

- Đặt RDS trong **private subnet**, không public Internet  
- Hạn chế port trong **Security Group**, chỉ cho phép từ app/bastion  
- Bật **encryption at rest** + **SSL/TLS in transit**  
- Không dùng **master user** cho app; tạo user riêng, phân quyền hạn chế  
- Lưu credentials trong **Secrets Manager** và **tự động rotate**  
- Bật **automated backups**, dùng **Multi-AZ** cho HA  
- Dùng **IAM** để giới hạn ai được thao tác với RDS & snapshot  

---

## 5. Aurora – Core & Features

### 5.1. High Performance & Storage

- Aurora là engine do AWS phát triển, tương thích **MySQL** & **PostgreSQL**
- Storage:
  - Phân tán, **self-healing**, tự động scale đến **64 TB**
  - Lưu 6 bản trên 3 AZ
  - **Continuous backup to S3** + PITR

### 5.2. Aurora Replicas

- Tối đa **15 Aurora Replicas** trong cùng Region
- Replica dùng để:
  - **Read scaling**
  - **Failover target** (gần như **không mất dữ liệu** do shared storage)
- **Replication lag**: thường tính bằng **milliseconds**

### 5.3. Cross-Region & Global

- **Cross-Region Replica with Aurora MySQL**:
  - Tối đa **5 cross-region replica clusters**
  - Mỗi cluster có thể có tối đa **15 Aurora Replicas**
- **Aurora Global Database**:
  - 1 primary Region (read/write)
  - 1+ secondary Regions (read-only)
  - **Fast cross-region replication**, low-latency read
  - Có thể **promote secondary Region** khi primary Region fail (DR)

### 5.4. Aurora Serverless (v1)

- Aurora dạng **on-demand, auto-scaling**
- Trả tiền theo **Aurora Capacity Units (ACUs)** theo thời gian thực dùng
- **Hạn chế quan trọng**:
  - Không hỗ trợ **read replicas**
  - Không có **public IP**:
    - Chỉ truy cập qua **VPC** hoặc **Direct Connect**
- Use cases:
  - Dev/Test, POC
  - Workload không ổn định, ít dùng

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
