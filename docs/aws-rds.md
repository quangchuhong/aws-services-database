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

```mermaid
flowchart LR
    App[Application] -->|SQL| DB[RDS DB Instance (Single-AZ)]
    DB --> B[(Automated Backups to S3)]
```

### 2.2. Mô hình 2 – Multi-AZ (High Availability)

```mermaid
flowchart LR
    App[Application] -->|DB Endpoint| P[(Primary RDS DB - AZ-A)]
    P -->|Synchronous Replication| S[(Standby RDS DB - AZ-B)]
    S --> B[(Automated Backups to S3)]

    classDef primary fill=#4CAF50,stroke=#333,stroke-width=1,color=#fff;
    classDef standby fill=#FFC107,stroke=#333,stroke-width=1;
    class P primary;
    class S standby;
```
