# AWS ElastiCache – Summary, Architecture Models & Examples

Tài liệu tóm tắt & ví dụ cụ thể về **Amazon ElastiCache**, dùng làm `README.md` trên GitHub.  
Phong cách giống README RDS/Aurora của bạn.

---

## 1. Amazon ElastiCache Overview

**Amazon ElastiCache** là dịch vụ **in-memory cache** managed trên AWS, giúp:

- Giảm **latency** (µs–ms) cho các truy vấn đọc.
- Giảm tải lên RDS/DynamoDB/Service backend.
- Tăng throughput toàn hệ thống.

Hỗ trợ **2 engine**:

- **Memcached** – cache đơn giản, key–value, không persistence, multithreaded.
- **Redis** – in-memory data store mạnh hơn (nhiều data types, persistence, replication, cluster).

Use case điển hình:

- Cache truy vấn DB / API.
- Session store (web session, auth token).
- Leaderboard, counter, real-time metrics.
- Pub/Sub, queue nhẹ.

---

## 2. Các mô hình kiến trúc ElastiCache (phổ biến)

### 2.1. Mô hình 1 – Cache-aside với RDS / Aurora (Single Node / Small Cluster)

**Use case:** Web app đọc nhiều, viết ít; giảm tải RDS/Aurora.

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
|  Cache      |            |   RDS / Aurora     |
| (Redis/Mem) |            |   Database         |
+------+------+            +---------+----------+
       ^                             |
       |                             v
 Notes:
 1. App đọc: Cache → (miss) → DB → ghi Cache.
 2. App ghi: ghi DB, tùy logic update/invalid cache.
```

### 2.2. Mô hình 2 – Redis Primary + Replicas (Read Scaling + HA trong 1 Region)

**Use case:** Cache/read store lớn, cần HA & scale đọc, phía sau vẫn có RDS/Aurora làm DB chính.

```text
                   +----------------------+
                   |     Application      |
                   +----------+-----------+
                              |
                  +-----------+-----------+
                  |                       |
                  v                       v
          (Đọc/ghi cache)         (Đọc/ghi DB chính)
                  |                       |
        +---------+---------+     +--------------------+
        | Redis Primary    |     |  RDS / Aurora DB   |
        | (Read/Write)     |     |  (nguồn dữ liệu    |
        +----+-------------+     |   chính / persis)  |
             |                   +--------------------+
    Replication (Multi-AZ)
             |
     +-------+--------+
     | Redis Replica1 |
     | (Read-only)    |
     +-------+--------+
             |
     +-------+--------+
     | Redis Replica2 |
     | (Read-only)    |
     +----------------+

Gợi ý luồng:
1. App GHI:
   - Ghi vào DB chính (RDS/Aurora).
   - Tùy logic: update/invalid cache trên Redis primary.
2. App ĐỌC:
   - Đọc từ Redis (primary/replicas qua reader endpoint).
   - Nếu cache miss → đọc DB → ghi cache.
