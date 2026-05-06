# 🔴 Redis - Hướng Dẫn Chi Tiết

## Redis là gì?

**Redis** (REmote DIctionary Server) là một hệ thống lưu trữ dữ liệu mã nguồn mở, hoạt động hoàn toàn trong bộ nhớ (in-memory) dưới dạng cấu trúc dữ liệu. Nó được biết đến với vai trò là:

- 🗄️ Cơ sở dữ liệu NoSQL
- ⚡ Bộ nhớ đệm (Cache)
- 📨 Bộ môi giới tin nhắn (Message Broker)

---

## 1. 🌟 Đặc điểm nổi bật của Redis

### ⚡ Tốc độ cực nhanh

- Lưu trữ dữ liệu trên RAM thay vì ổ cứng
- Thực hiện hàng triệu yêu cầu mỗi giây
- Độ trễ cực thấp (vài micro giây)

**Ví dụ:** Truy vấn từ Redis nhanh hơn 100-1000x so với cơ sở dữ liệu truyền thống

### 📊 Cấu trúc dữ liệu đa dạng

Redis không phải chỉ là key-value đơn giản, mà hỗ trợ:

| Kiểu dữ liệu     | Mô tả                       | Ứng dụng               |
| ---------------- | --------------------------- | ---------------------- |
| **Strings**      | Chuỗi văn bản, số           | Lưu session, cache     |
| **Lists**        | Danh sách có thứ tự         | Hàng đợi, timeline     |
| **Sets**         | Tập hợp không trùng lặp     | Tags, followers        |
| **Hashes**       | Bản ghi key-value lồng nhau | User profile, metadata |
| **Sorted Sets**  | Tập hợp có điểm số          | Leaderboards, rankings |
| **Bitmaps**      | Bit operations              | Tracking, analytics    |
| **HyperLogLogs** | Đếm phần tử duy nhất        | Unique visitors        |
| **Streams**      | Luồng dữ liệu               | Message queue, logging |

### 💾 Tính bền vững (Persistence)

Mặc dù hoạt động trên RAM, Redis vẫn có thể:

- Sao lưu dữ liệu xuống đĩa cứng định kỳ
- Phục hồi dữ liệu khi hệ thống gặp sự cố

### 📡 Hỗ trợ Pub/Sub

Redis cho phép xây dựng các hệ thống giao tiếp thời gian thực thông qua cơ chế Publish/Subscribe

---

## 2. 💡 Các ứng dụng phổ biến

### 🔹 Caching - Lưu trữ cache

**Mục đích:** Lưu trữ dữ liệu thường xuyên truy cập để giảm tải DB chính

```javascript
// Node.js with redis client
const redis = require("redis");
const client = redis.createClient();

// Cache kết quả truy vấn SQL
async function getUser(userId) {
  // Kiểm tra cache trước
  const cached = await client.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);

  // Nếu không có, query database
  const user = await db.query("SELECT * FROM users WHERE id = ?", [userId]);

  // Lưu vào cache 1 giờ (3600 giây)
  await client.setex(`user:${userId}`, 3600, JSON.stringify(user));

  return user;
}
```

### 🔹 Quản lý Session

**Mục đích:** Lưu trữ thông tin đăng nhập, giỏ hàng của người dùng

```javascript
// Lưu session người dùng
await client.setex(
  `session:${sessionId}`,
  86400,
  JSON.stringify({
    userId: 123,
    email: "user@example.com",
    cartItems: ["item1", "item2"],
    loginTime: new Date(),
  }),
);

// Lấy session
const session = JSON.parse(await client.get(`session:${sessionId}`));
```

### 🔹 Bảng xếp hạng (Leaderboards)

**Mục đích:** Tạo và cập nhật bảng xếp hạng game/ứng dụng theo thời gian thực

```javascript
// Sử dụng Sorted Set để lưu điểm số
// Format: client.zadd(key, score, member)

// Cập nhật điểm khi người chơi thắng
await client.zadd("game_leaderboard", 100, "player1"); // player1 có 100 điểm
await client.zadd("game_leaderboard", 250, "player2"); // player2 có 250 điểm
await client.zadd("game_leaderboard", 150, "player3"); // player3 có 150 điểm

// Lấy top 10 người chơi (xếp hạng cao nhất)
const top10 = await client.zrevrange("game_leaderboard", 0, 9, "WITHSCORES");
console.log(top10);
// Output: ['player2', '250', 'player3', '150', 'player1', '100']

// Lấy hạng của một người chơi
const rank = await client.zrevrank("game_leaderboard", "player1");
console.log(`player1 xếp hạng: ${rank + 1}`); // Output: 3
```

### 🔹 Hàng đợi tin nhắn (Queues)

**Mục đích:** Điều phối các tác vụ nền

```javascript
// Producer: Thêm job vào queue
async function addJob(jobData) {
  await client.rpush("job_queue", JSON.stringify(jobData));
}

// Consumer: Lấy và xử lý job
async function processQueue() {
  while (true) {
    // BLPOP chặn đợi cho đến khi có job (timeout 0 = chờ mãi)
    const job = await client.blpop("job_queue", 0);
    if (job) {
      const jobData = JSON.parse(job[1]);
      console.log("Processing job:", jobData);
      // Xử lý job...
    }
  }
}
```

### 🔹 Pub/Sub - Giao tiếp thời gian thực

**Mục đích:** Xây dựng hệ thống real-time messaging

```javascript
// Publisher: Gửi tin nhắn
async function publishMessage(channel, message) {
  await client.publish(channel, JSON.stringify(message));
}

// Subscriber: Nhận tin nhắn
async function subscribeToChannel(channel) {
  const subscriber = client.duplicate();

  subscriber.on("message", (ch, message) => {
    console.log(`Nhận tin từ ${ch}: ${message}`);
  });

  await subscriber.subscribe(channel);
}

// Sử dụng
publishMessage("chat_room_1", { user: "Alice", text: "Hello!" });
subscribeToChannel("chat_room_1");
```

---

## 3. 📝 Các lệnh Redis cơ bản

| Lệnh     | Mô tả                      | Ví dụ                    |
| -------- | -------------------------- | ------------------------ |
| `SET`    | Đặt giá trị                | `SET key value`          |
| `GET`    | Lấy giá trị                | `GET key`                |
| `DEL`    | Xóa key                    | `DEL key`                |
| `EXPIRE` | Đặt thời gian sống         | `EXPIRE key 3600`        |
| `TTL`    | Kiểm tra thời gian còn lại | `TTL key`                |
| `INCR`   | Tăng số lên 1              | `INCR counter`           |
| `LPUSH`  | Thêm vào đầu list          | `LPUSH list value`       |
| `RPOP`   | Lấy từ cuối list           | `RPOP list`              |
| `SADD`   | Thêm vào set               | `SADD set member`        |
| `ZADD`   | Thêm vào sorted set        | `ZADD zset score member` |
| `HSET`   | Đặt field trong hash       | `HSET hash field value`  |

---

## 4. 🚀 Cài đặt Redis

### macOS

```bash
brew install redis
brew services start redis
```

### Docker

```bash
docker run -d -p 6379:6379 redis:latest
```

### Node.js Client

```bash
npm install redis
```

```javascript
const redis = require("redis");
const client = redis.createClient({
  host: "localhost",
  port: 6379,
});

client.on("connect", () => {
  console.log("Connected to Redis!");
});
```

---

## 5. 📊 So sánh Redis với các lựa chọn khác

| Tiêu chí         | Redis      | Memcached | Database   |
| ---------------- | ---------- | --------- | ---------- |
| **Tốc độ**       | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐  | ⭐⭐       |
| **Kiểu dữ liệu** | ⭐⭐⭐⭐⭐ | ⭐⭐      | ⭐⭐⭐⭐⭐ |
| **Persistence**  | ✅         | ❌        | ✅         |
| **Pub/Sub**      | ✅         | ❌        | ❌         |
| **TTL**          | ✅         | ✅        | ❌         |
| **Cluster**      | ✅         | ✅        | ✅         |

**Kết luận:** Redis phù hợp cho cache, session, real-time; Database phù hợp cho dữ liệu lâu dài

---

## 6. 🔥 Tại Sao Redis Lại Nhanh Khủng Khiếp?

### 📚 Bản chất của sự nhanh: Làm việc trực tiếp trên RAM

Cái "hay" của Redis chính là: **Nó lưu trữ và làm việc trực tiếp trên RAM ngay từ đầu**, không phải đọc từ ổ cứng mỗi lần cần dữ liệu.

---

### 🔄 Luồng dữ liệu: "Ngược" so với Database truyền thống

#### Database truyền thống (MySQL, PostgreSQL)

```
Request
  ↓
Ổ cứng (HDD/SSD) ← NỐN THẮT CỔ CHAI
  ↓
Đưa dữ liệu lên RAM
  ↓
Xử lý
  ↓
Trả về kết quả
```

**Vấn đề:** Ổ cứng cực chậm (milliseconds), dù SSD cũng chậm hơn RAM hàng trăm lần

#### Redis (Ngược lại)

```
Request
  ↓
Dữ liệu đã có sẵn ở RAM
  ↓
Xử lý ngay lập tức (microseconds)
  ↓
Trả về kết quả
  ↓
Ghi xuống ổ cứng (backup, không ảnh hưởng performance)
```

**Ưu điểm:** Phản hồi tức thì, không bao giờ chờ ổ cứng

---

### ⏱️ So sánh tốc độ

| Thao tác       | Tốc độ             | Ví dụ            |
| -------------- | ------------------ | ---------------- |
| Truy cập RAM   | 1 microsecond (μs) | ⚡ Redis         |
| SSD Read/Write | 100 microseconds   | 100x chậm hơn    |
| HDD Read/Write | 10+ milliseconds   | 10,000x chậm hơn |

```
Redis:    ████ (1μs)
SSD:      ██████████████████████████ (100μs)
HDD:      ████████████████████████████████████████ (10,000μs)
```

---

### 🎯 Tại Sao Redis Lại "Hay" Hơn Là Lưu RAM Bình Thường?

Nếu chỉ là lưu trên RAM thì biến (variable) trong code cũng làm được. Redis hay ở ba điểm này:

#### 1️⃣ **Vừa nhanh vừa bền (Fast + Durable)**

Redis không chỉ nhanh, mà còn **không mất dữ liệu** khi server sập!

**Cơ chế:**

- **RDB (Redis Database)**: Định kỳ (mỗi 15 phút) snapshot toàn bộ dữ liệu xuống đĩa
- **AOF (Append Only File)**: Ghi lại mọi lệnh thay đổi, giống như "undo/redo history"

```javascript
// Config Redis persistence
// redis.conf
save 900 1        // Nếu có 1 key thay đổi trong 900 giây, save
save 300 10       // Nếu có 10 key thay đổi trong 300 giây, save
save 60 10000     // Nếu có 10000 key thay đổi trong 60 giây, save

appendonly yes    // Bật AOF (Append Only File)
```

**Sơ đồ:**

```
Thời điểm 1
Redis (RAM)  ← [user:1, user:2, user:3] ← Application
  ↓ (RDB/AOF)
Ổ cứng       ← Backup [user:1, user:2, user:3]

Server crash!

Khởi động lại
Ổ cứng [user:1, user:2, user:3]
  ↓ Load
Redis (RAM) ← Dữ liệu được phục hồi ✅
```

---

#### 2️⃣ **Cấu trúc dữ liệu thông minh**

Redis không chỉ lưu chuỗi văn bản, mà **hiểu được các cấu trúc phức tạp** và xử lý chúng cực nhanh trên RAM.

**Ví dụ: Lấy Top 5 người có điểm cao nhất**

❌ **Cách truyền thống (Database):**

```sql
-- Query: Quét qua HÀNG TRIỆU dòng ở ổ cứng
SELECT name, score FROM leaderboard
ORDER BY score DESC
LIMIT 5;
-- Kết quả: CHẬM (phải đọc từ ổ cứng)
```

✅ **Cách Redis (Smart Data Structure):**

```javascript
// Redis xử lý ngay trên RAM
const top5 = await client.zrevrange("leaderboard", 0, 4, "WITHSCORES");
// Kết quả: NHANH (dữ liệu đã sắp xếp sẵn trong Sorted Set)
```

**So sánh:**

```
Database:  Phải quét 1,000,000 dòng → sắp xếp → lấy 5 cái
Redis:     Sorted Set đã sắp xếp sẵn → lấy 5 cái ngay
```

---

#### 3️⃣ **Chia sẻ dữ liệu giữa nhiều server**

Nếu bạn có 10 server web chạy cùng lúc, mỗi server không thể dùng RAM cục bộ của mình (vì dữ liệu khác nhau). **Redis giải quyết bằng cách tập trung dữ liệu vào 1 server**:

```
┌─────────────┐
│  Redis      │ ← Central data store
│  Server     │
└────────┬────┘
    ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑
    │ │ │ │ │ │ │ │ │ │
┌───┴┬┴┬┴┬┴┬┴┬┴┬┴┬┴┬┴┬┴───┐
Web1 Web2 Web3 Web4 Web5 ... Web10
(Tất cả đều dùng chung dữ liệu từ Redis)
```

**Ví dụ thực tế:**

```javascript
// Web Server 1: Đăng nhập
await redis.setex(
  `session:user123`,
  3600,
  JSON.stringify({
    userId: 123,
    username: "alice",
  }),
);

// Web Server 2: Kiểm tra session
const session = await redis.get(`session:user123`);
// ✅ Web Server 2 cũng thấy session của user từ Web Server 1
```

**Lợi ích:**

- ✅ Tất cả server thấy cùng dữ liệu (Session, Cart, Cache)
- ✅ Không phải query database 10 lần
- ✅ Dữ liệu luôn sync

---

### 📊 Bảng So Sánh: Biến RAM vs Redis vs Database

| Đặc tính                    | Biến RAM   | Redis             | Database |
| --------------------------- | ---------- | ----------------- | -------- |
| Tốc độ                      | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐        | ⭐⭐     |
| Lưu dữ liệu khi crash       | ❌         | ✅                | ✅       |
| Chia sẻ giữa các process    | ❌         | ✅                | ✅       |
| Cấu trúc dữ liệu thông minh | ❌         | ✅                | ✅       |
| Dung lượng lớn              | ❌         | ⚠️ (Giới hạn RAM) | ✅       |
| Độ bền                      | ❌         | ✅                | ✅       |

---

## 7. 💡 Khi Nào Dùng Redis, Khi Nào Dùng Database?

| Tình huống                   | Giải pháp |
| ---------------------------- | --------- |
| Lưu session người dùng       | Redis     |
| Cache kết quả query          | Redis     |
| Leaderboard game             | Redis     |
| Real-time notifications      | Redis     |
| Dữ liệu lâu dài (năm, tháng) | Database  |
| Dữ liệu báo cáo, analytics   | Database  |
| Dữ liệu lịch sử giao dịch    | Database  |
| Dữ liệu >100GB               | Database  |

**💡 Tip:** Kết hợp cả hai! Database + Redis cache = Perfect 🚀

---

## 8. 🎯 Cache Strategies - 6 Chiến Lược Quản Lý Cache

Trong kỹ thuật hệ thống, để quản lý cache hiệu quả, có 6 chiến lược phổ biến. Mỗi chiến lược ưu tiên khác nhau giữa **tốc độ đọc**, **tốc độ ghi**, và **tính nhất quán dữ liệu**.

---

### 1️⃣ **Cache-Aside (Lazy Loading)** ⭐ Phổ biến nhất

**Mô tả:** Ứng dụng chủ động kiểm tra cache. Nếu không có, lấy từ DB rồi lưu vào cache.

**Luồng dữ liệu:**

```
Application
  ↓
Kiểm tra cache
  ↓ (Cache Hit)     ↓ (Cache Miss)
 Trả ngay          Query DB
                     ↓
                  Lưu vào cache
                     ↓
                  Trả về user
```

**Code Example:**

```javascript
async function getUser(userId) {
  // Bước 1: Kiểm tra cache trước
  let user = await redis.get(`user:${userId}`);
  if (user) {
    console.log("✅ Cache Hit");
    return JSON.parse(user);
  }

  // Bước 2: Cache miss - query database
  console.log("❌ Cache Miss - Query DB");
  user = await db.query("SELECT * FROM users WHERE id = ?", [userId]);

  // Bước 3: Lưu vào cache để lần sau nhanh hơn
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));

  return user;
}
```

**Ưu điểm:**

- ✅ Cache chỉ chứa dữ liệu thực sự cần thiết
- ✅ Không phí cache vào dữ liệu không cần
- ✅ Dễ implement

**Nhược điểm:**

- ❌ Lần đầu truy cập sẽ chậm (miss)
- ❌ Có thời gian data stale (dữ liệu cũ)

**Khi nào dùng:** Ứng dụng web thông thường, blog, e-commerce

---

### 2️⃣ **Read-Through Cache**

**Mô tả:** Giống Cache-Aside nhưng ứng dụng không tự lấy DB. Thay vào đó, cache library tự động xử lý.

**Luồng dữ liệu:**

```
Application
  ↓
Yêu cầu Cache Library
  ↓
Cache Library kiểm tra
  ↓ (Hit)          ↓ (Miss)
Trả ngay      Tự query DB + Lưu cache
  ↑              ↓
└──────────────────┘ Trả về
```

**Code Example:**

```javascript
// Sử dụng library như node-cache hoặc Redis wrapper
const cacheManager = require("cache-manager");
const redisStore = require("cache-manager-redis-store");

const cache = cacheManager.caching({
  store: redisStore,
  host: "localhost",
  port: 6379,
  ttl: 3600, // 1 giờ
});

async function getUser(userId) {
  // Cache library tự động kiểm tra & query
  return await cache.wrap(`user:${userId}`, async () => {
    return await db.query("SELECT * FROM users WHERE id = ?", [userId]);
  });
}
```

**Ưu điểm:**

- ✅ Code ứng dụng sạch hơn
- ✅ Tách biệt cache logic

**Nhược điểm:**

- ❌ Phụ thuộc library/ORM hỗ trợ
- ❌ Lần đầu vẫn chậm

**Khi nào dùng:** Ứng dụng sử dụng ORM/framework hỗ trợ

---

### 3️⃣ **Write-Through Cache**

**Mô tả:** Khi ghi dữ liệu, phải ghi vào **cả Cache AND Database** cùng lúc.

**Luồng dữ liệu:**

```
Application
  ↓
Ghi vào Cache
  ↓ (Đợi hoàn tất)
Ghi vào Database
  ↓ (Cả hai xong)
Báo "Success" cho user
```

**Code Example:**

```javascript
async function updateUser(userId, userData) {
  // Ghi vào cache
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(userData));

  // Ghi vào database (phải đợi cả hai hoàn tất)
  await db.query("UPDATE users SET ? WHERE id = ?", [userData, userId]);

  return { success: true };
}
```

**Ưu điểm:**

- ✅ Cache luôn mới nhất, khớp với DB
- ✅ Không bao giờ bị data mismatch
- ✅ An toàn dữ liệu

**Nhược điểm:**

- ❌ Tốc độ ghi chậm (phải đợi cả 2)
- ❌ Nếu cache fail, phải rollback

**Khi nào dùng:** Ứng dụng cần tính nhất quán cao (Banking, Payment)

---

### 4️⃣ **Write-Around (Write-Behind)**

**Mô tả:** Ghi dữ liệu trực tiếp vào DB, **bỏ qua cache**. Cache chỉ được cập nhật khi có ai đó đọc.

**Luồng dữ liệu:**

```
Write Request
  ↓
Ghi trực tiếp DB (không update cache)
  ↓
Báo "Success" (nhanh)

Read Request
  ↓
Cache miss → Query DB → Update cache
```

**Code Example:**

```javascript
async function updateUser(userId, userData) {
  // Chỉ ghi vào DB, BỎ QUA cache
  await db.query("UPDATE users SET ? WHERE id = ?", [userData, userId]);

  // (Không ghi vào cache)
  // Lần tới khi ai đó đọc, sẽ cache-aside tự động

  return { success: true };
}

async function getUser(userId) {
  // Cache-aside: nếu miss thì update
  let user = await redis.get(`user:${userId}`);
  if (!user) {
    user = await db.query("SELECT * FROM users WHERE id = ?", [userId]);
    await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));
  }
  return JSON.parse(user);
}
```

**Ưu điểm:**

- ✅ Ghi rất nhanh (không đợi cache)
- ✅ Tránh cache bị đầy bằng dữ liệu ít dùng
- ✅ Giảm tải cache

**Nhược điểm:**

- ❌ Một thời gian Cache sẽ outdated
- ❌ Lần đầu đọc sẽ miss

**Khi nào dùng:** Ứng dụng có lượt ghi lớn, ít đọc

---

### 5️⃣ **Write-Back (Write-Behind)** ⚠️ Rủi ro cao

**Mô tả:** Ghi vào cache rồi **báo "xong" ngay**. Sau đó, cache tự động batch ghi vào DB theo định kỳ.

**Luồng dữ liệu:**

```
Write Request
  ↓
Ghi vào cache + Báo "xong" (cực nhanh)
  ↓ (Background - định kỳ)
Cache gom dữ liệu → Batch ghi DB
```

**Code Example:**

```javascript
// Cấu hình cache batch write
const batchWriteInterval = 5000; // Mỗi 5 giây
const writeBuffer = [];

async function updateUser(userId, userData) {
  // Chỉ ghi vào cache, báo xong ngay
  await redis.set(`user:${userId}`, JSON.stringify(userData));

  // Thêm vào batch queue
  writeBuffer.push({ userId, userData });

  return { success: true }; // ← Báo xong ngay (NHANH!)
}

// Background worker: Batch write to DB
setInterval(async () => {
  if (writeBuffer.length === 0) return;

  const batch = writeBuffer.splice(0); // Lấy tất cả

  try {
    await db.query("INSERT INTO user_updates VALUES ?", [batch]);
    console.log(`✅ Batch wrote ${batch.length} updates`);
  } catch (err) {
    console.error("❌ Batch write failed!", err);
    // ⚠️ Dữ liệu có thể bị mất nếu cache sập!
  }
}, batchWriteInterval);
```

**Ưu điểm:**

- ✅ Tốc độ ghi cực nhanh (write immediately to cache)
- ✅ Chịu được tải ghi lớn
- ✅ Giảm I/O database

**Nhược điểm:**

- ❌ ⚠️ **RẤT RỦI RO** - Nếu cache sập trước khi batch ghi DB, dữ liệu **MẤT VĩNH VIỄN**
- ❌ Khó debug issues
- ❌ Yêu cầu monitoring chặt chẽ

**Khi nào dùng:** Analytics, logs (dữ liệu không critical), hoặc cache có backup

---

### 6️⃣ **Refresh-Ahead**

**Mô tả:** Cache tự động dự đoán và **làm mới dữ liệu sắp hết hạn** trước khi user yêu cầu.

**Luồng dữ liệu:**

```
Dữ liệu trong cache
  ↓ (Sắp hết hạn? TTL < 30%)
Cache tự động query DB + Update
  ↓
User yêu cầu
  ↓
Dữ liệu mới nhất + Nhanh ✅
```

**Code Example:**

```javascript
const REFRESH_THRESHOLD = 0.3; // Refresh khi còn 30% TTL

async function refreshAhead(key, ttl, queryFn) {
  const value = await redis.get(key);
  const remaining = await redis.ttl(key);

  // Nếu còn < 30% TTL, refresh trước
  if (remaining > 0 && remaining / ttl < REFRESH_THRESHOLD) {
    console.log(`🔄 Refreshing ${key} (TTL: ${remaining}s)`);

    const newValue = await queryFn();
    await redis.setex(key, ttl, JSON.stringify(newValue));
  }

  return JSON.parse(value);
}

// Sử dụng
async function getProduct(productId) {
  return await refreshAhead(
    `product:${productId}`,
    3600, // TTL 1 giờ
    async () => {
      console.log("📍 Query database for fresh product data");
      return await db.query("SELECT * FROM products WHERE id = ?", [productId]);
    },
  );
}
```

**Ưu điểm:**

- ✅ User luôn có dữ liệu mới + nhanh
- ✅ Không bao giờ cache miss
- ✅ Giảm độ trễ

**Nhược điểm:**

- ❌ Phức tạp implement
- ❌ Tiêu tốn tài nguyên refresh không cần thiết
- ❌ Khó dự đoán pattern user

**Khi nào dùng:** Dữ liệu hot (được truy cập nhiều), dashboard real-time

---

## 9. 📊 So Sánh 6 Chiến Lược Cache

| Chiến lược        | Tốc độ Đọc | Tốc độ Ghi | Nhất quán   | Rủi ro   | Complexity |
| ----------------- | ---------- | ---------- | ----------- | -------- | ---------- |
| **Cache-Aside**   | ⭐⭐⭐     | ⭐⭐⭐⭐⭐ | ⚠️ Thấp     | 🟢 Thấp  | 🟢 Dễ      |
| **Read-Through**  | ⭐⭐⭐     | ⭐⭐⭐⭐⭐ | ⚠️ Thấp     | 🟢 Thấp  | 🟡 Trung   |
| **Write-Through** | ⭐⭐⭐⭐   | ⭐⭐       | 🟢 Cao      | 🟢 Thấp  | 🟡 Trung   |
| **Write-Around**  | ⭐⭐       | ⭐⭐⭐⭐⭐ | ⚠️ Thấp     | 🟢 Thấp  | 🟢 Dễ      |
| **Write-Back**    | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 🔴 Rất thấp | 🔴 CAO   | 🔴 Khó     |
| **Refresh-Ahead** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐     | 🟡 Trung    | 🟡 Trung | 🔴 Khó     |

---

## 10. 🎯 Hướng Dẫn Chọn Chiến Lược

```
Có cần dữ liệu luôn mới nhất?
  ├─ YES → Write-Through (safe nhất)
  └─ NO ↓

Có lượt ghi rất lớn (>10k/s)?
  ├─ YES → Write-Back (nhanh nhất) ⚠️ nhưng rủi ro
  │        hoặc Write-Around (an toàn hơn)
  └─ NO ↓

Dữ liệu thường xuyên truy cập?
  ├─ YES → Refresh-Ahead (best UX)
  └─ NO → Cache-Aside (default, phổ biến nhất)
```

**Khuyến nghị theo loại ứng dụng:**

| Loại ứng dụng                 | Chiến lược                  |
| ----------------------------- | --------------------------- |
| **Web App, Blog, E-commerce** | Cache-Aside                 |
| **Real-time Dashboard**       | Refresh-Ahead               |
| **Payment, Banking**          | Write-Through               |
| **High-traffic Analytics**    | Write-Back + Backup cache   |
| **Social Media Feed**         | Cache-Aside + Refresh-Ahead |
| **Search Engine**             | Read-Through + Write-Around |

---

## 11. 💡 Best Practice: Hybrid Strategy

**Thực tế, ứng dụng lớn thường kết hợp 2-3 chiến lược:**

```javascript
// Hybrid: Cache-Aside + Refresh-Ahead + Write-Through
class HybridCacheManager {
  async getUser(userId) {
    // 1. Cache-Aside: Kiểm tra cache
    let user = await redis.get(`user:${userId}`);

    if (!user) {
      // 2. Cache Miss - Query database
      user = await db.query("SELECT * FROM users WHERE id = ?", [userId]);
      await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));
    } else {
      // 3. Refresh-Ahead: Nếu sắp hết hạn, update background
      const ttl = await redis.ttl(`user:${userId}`);
      if (ttl < 300) {
        // Còn < 5 phút
        this.refreshUserInBackground(userId);
      }
    }

    return JSON.parse(user);
  }

  async updateUser(userId, data) {
    // Write-Through: Ghi cả 2 cùng lúc
    await redis.set(`user:${userId}`, JSON.stringify(data));
    await db.query("UPDATE users SET ? WHERE id = ?", [data, userId]);
  }

  async refreshUserInBackground(userId) {
    const user = await db.query("SELECT * FROM users WHERE id = ?", [userId]);
    await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));
  }
}
```

**Kết quả:**

- ✅ Đọc nhanh (Cache-Aside)
- ✅ Luôn có data fresh (Refresh-Ahead)
- ✅ Data nhất quán (Write-Through)
