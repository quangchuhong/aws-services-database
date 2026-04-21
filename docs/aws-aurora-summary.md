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
