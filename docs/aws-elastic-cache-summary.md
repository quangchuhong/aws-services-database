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
```

### 2.3. Mô hình 3 – Redis Cluster Mode Enabled (Sharding + HA)

**Use case:** Cache/store rất lớn, throughput rất cao → cần **sharding + HA**, phía sau vẫn có DB (RDS/Aurora) làm nguồn dữ liệu chính.

```text
                          +----------------------+
                          |     Application      |
                          +----------+-----------+
                                     |
                             (Redis Cluster client)
                             Tự hash key → chọn shard
                                     |
          +--------------------------+---------------------------+
          |                          |                           |
          v                          v                           v
   +-------------+            +-------------+             +-------------+
   |  Shard 1    |            |  Shard 2    |             |  Shard 3    |
   | (Slot range)|            | (Slot range)|             | (Slot range)|
   +------+------+            +------+------+             +------+------+
          |                          |                           |
   +------+-------+          +-------+------+             +------+-------+
   | Primary 1    |          | Primary 2    |             | Primary 3    |
   | (Read/Write) |          | (Read/Write) |             | (Read/Write) |
   +------+-------+          +-------+------+             +-------+------+
          |                           |                            |
   Replicas (0-5)              Replicas (0-5)                 Replicas (0-5)
 (Read-only, HA)             (Read-only, HA)                (Read-only, HA)
          |                           |                            |
          +-------------+-------------+-------------+--------------+
                        |                           |
                        v                           v
               +------------------+        +------------------+
               |   RDS / Aurora   |  ...   |   (các DB khác)  |
               |  Database chính  |        |  (tuỳ kiến trúc) |
               +------------------+        +------------------+

Luồng gợi ý:
1. App ĐỌC:
   - Gọi Redis Cluster (client) → client hash key → chọn shard (Primary/Replica).
   - Nếu cache hit → trả về data.
   - Nếu cache miss → đọc từ RDS/Aurora → ghi vào Redis shard tương ứng.

2. App GHI:
   - Ghi vào DB chính (RDS/Aurora).
   - Tùy logic: update/invalid cache trên shard tương ứng (qua Redis Cluster client).

Đặc điểm chính:
- Keyspace được chia thành nhiều **hash slot**, mỗi shard chịu trách nhiệm 1 nhóm slot.
- Mỗi shard = 1 primary + 0–5 replicas → **sharding + HA (Multi-AZ + auto-failover)**.
- Scale-out bằng cách **thêm shard**, Redis tự redistribute hash slots.
```

### 2.4. Mô hình 4 – Session Store với Redis

**Use case:** Lưu session đăng nhập / giỏ hàng / token… để web server **stateless**, dễ scale ngang.  
Session **chỉ cần tồn tại tạm thời**, không nhất thiết lưu trong RDS.

```text
                 Internet
                    |
                    v
           +----------------------+
           |   Load Balancer     |
           |      (ALB/NLB)      |
           +----------+----------+
                      |
         +------------+------------+
         |            |            |
         v            v            v
   +-----------+  +-----------+  +-----------+
   |  Web 1    |  |  Web 2    |  |  Web 3    |
   | (EC2/ECS) |  | (EC2/ECS) |  | (EC2/ECS) |
   +-----+-----+  +-----+-----+  +-----+-----+
         |              |              |
         +--------------+--------------+
                        |
                        v
                +----------------+
                |  Redis Cluster |
                |  Session Store |
                +----------------+

Luồng điển hình:
1. User login:
   - Web node xác thực (có thể gọi DB/User service).
   - Tạo session ID / token.
   - Ghi session vào Redis (key = session_id, value = user info, TTL = thời gian session).

2. Request tiếp theo:
   - Web node nhận cookie / token.
   - Đọc session từ Redis:
     - Nếu tìm thấy → user đã login, tiếp tục xử lý.
     - Nếu không thấy (hết hạn / bị xóa) → yêu cầu login lại.

3. Thêm/bớt web node:
   - Không ảnh hưởng session, vì tất cả đều dùng chung Redis.
   - Web servers **stateless**, dễ scale in/out.
```
