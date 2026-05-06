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

| Kiểu dữ liệu | Mô tả | Ứng dụng |
|-------------|-------|---------|
| **Strings** | Chuỗi văn bản, số | Lưu session, cache |
| **Lists** | Danh sách có thứ tự | Hàng đợi, timeline |
| **Sets** | Tập hợp không trùng lặp | Tags, followers |
| **Hashes** | Bản ghi key-value lồng nhau | User profile, metadata |
| **Sorted Sets** | Tập hợp có điểm số | Leaderboards, rankings |
| **Bitmaps** | Bit operations | Tracking, analytics |
| **HyperLogLogs** | Đếm phần tử duy nhất | Unique visitors |
| **Streams** | Luồng dữ liệu | Message queue, logging |

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
const redis = require('redis');
const client = redis.createClient();

// Cache kết quả truy vấn SQL
async function getUser(userId) {
  // Kiểm tra cache trước
  const cached = await client.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);
  
  // Nếu không có, query database
  const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);
  
  // Lưu vào cache 1 giờ (3600 giây)
  await client.setex(`user:${userId}`, 3600, JSON.stringify(user));
  
  return user;
}
```

### 🔹 Quản lý Session
**Mục đích:** Lưu trữ thông tin đăng nhập, giỏ hàng của người dùng

```javascript
// Lưu session người dùng
await client.setex(`session:${sessionId}`, 86400, JSON.stringify({
  userId: 123,
  email: 'user@example.com',
  cartItems: ['item1', 'item2'],
  loginTime: new Date()
}));

// Lấy session
const session = JSON.parse(await client.get(`session:${sessionId}`));
```

### 🔹 Bảng xếp hạng (Leaderboards)
**Mục đích:** Tạo và cập nhật bảng xếp hạng game/ứng dụng theo thời gian thực

```javascript
// Sử dụng Sorted Set để lưu điểm số
// Format: client.zadd(key, score, member)

// Cập nhật điểm khi người chơi thắng
await client.zadd('game_leaderboard', 100, 'player1');  // player1 có 100 điểm
await client.zadd('game_leaderboard', 250, 'player2');  // player2 có 250 điểm
await client.zadd('game_leaderboard', 150, 'player3');  // player3 có 150 điểm

// Lấy top 10 người chơi (xếp hạng cao nhất)
const top10 = await client.zrevrange('game_leaderboard', 0, 9, 'WITHSCORES');
console.log(top10);
// Output: ['player2', '250', 'player3', '150', 'player1', '100']

// Lấy hạng của một người chơi
const rank = await client.zrevrank('game_leaderboard', 'player1');
console.log(`player1 xếp hạng: ${rank + 1}`);  // Output: 3
```

### 🔹 Hàng đợi tin nhắn (Queues)
**Mục đích:** Điều phối các tác vụ nền

```javascript
// Producer: Thêm job vào queue
async function addJob(jobData) {
  await client.rpush('job_queue', JSON.stringify(jobData));
}

// Consumer: Lấy và xử lý job
async function processQueue() {
  while (true) {
    // BLPOP chặn đợi cho đến khi có job (timeout 0 = chờ mãi)
    const job = await client.blpop('job_queue', 0);
    if (job) {
      const jobData = JSON.parse(job[1]);
      console.log('Processing job:', jobData);
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
  
  subscriber.on('message', (ch, message) => {
    console.log(`Nhận tin từ ${ch}: ${message}`);
  });
  
  await subscriber.subscribe(channel);
}

// Sử dụng
publishMessage('chat_room_1', { user: 'Alice', text: 'Hello!' });
subscribeToChannel('chat_room_1');
```

---

## 3. 📝 Các lệnh Redis cơ bản

| Lệnh | Mô tả | Ví dụ |
|------|-------|-------|
| `SET` | Đặt giá trị | `SET key value` |
| `GET` | Lấy giá trị | `GET key` |
| `DEL` | Xóa key | `DEL key` |
| `EXPIRE` | Đặt thời gian sống | `EXPIRE key 3600` |
| `TTL` | Kiểm tra thời gian còn lại | `TTL key` |
| `INCR` | Tăng số lên 1 | `INCR counter` |
| `LPUSH` | Thêm vào đầu list | `LPUSH list value` |
| `RPOP` | Lấy từ cuối list | `RPOP list` |
| `SADD` | Thêm vào set | `SADD set member` |
| `ZADD` | Thêm vào sorted set | `ZADD zset score member` |
| `HSET` | Đặt field trong hash | `HSET hash field value` |

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
const redis = require('redis');
const client = redis.createClient({
  host: 'localhost',
  port: 6379
});

client.on('connect', () => {
  console.log('Connected to Redis!');
});
```

---

## 5. 📊 So sánh Redis với các lựa chọn khác

| Tiêu chí | Redis | Memcached | Database |
|----------|-------|-----------|----------|
| **Tốc độ** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **Kiểu dữ liệu** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Persistence** | ✅ | ❌ | ✅ |
| **Pub/Sub** | ✅ | ❌ | ❌ |
| **TTL** | ✅ | ✅ | ❌ |
| **Cluster** | ✅ | ✅ | ✅ |

**Kết luận:** Redis phù hợp cho cache, session, real-time; Database phù hợp cho dữ liệu lâu dài
