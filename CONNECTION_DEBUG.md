# 🔍 观影室连接失败排查指南

## 问题现象

```
[WatchRoom] Unauthorized connection attempt from: https://tv.886200.xyz
```

这表示 MoonTVPlus 客户端尝试连接观影室服务器，但认证失败。

## 排查步骤

### 1. 检查 Vercel 环境变量

登录 Vercel 控制台，检查以下环境变量是否正确设置：

```env
WATCH_ROOM_ENABLED=true
WATCH_ROOM_SERVER_TYPE=external
WATCH_ROOM_EXTERNAL_SERVER_URL=https://your-watch-room-server.com
WATCH_ROOM_EXTERNAL_SERVER_AUTH=your-secret-auth-key
```

**重要提示：**
- `WATCH_ROOM_EXTERNAL_SERVER_AUTH` 必须与观影室服务器的 `AUTH_KEY` 完全一致
- URL 必须包含 `https://` 或 `http://`
- 设置后需要重新部署 Vercel 项目

### 2. 验证观影室服务器的 AUTH_KEY

检查观影室服务器的环境变量：

```bash
# 如果使用 Railway/Render
# 在控制台查看环境变量 AUTH_KEY

# 如果使用 Docker
docker exec watch-room-server env | grep AUTH_KEY

# 如果使用 PM2
pm2 env watch-room-server | grep AUTH_KEY
```

### 3. 测试配置 API

在浏览器访问：
```
https://tv.886200.xyz/api/server-config
```

检查返回的 JSON 中的 `WatchRoom` 配置：

```json
{
  "WatchRoom": {
    "enabled": true,
    "serverType": "external",
    "externalServerUrl": "https://your-server.com",
    "externalServerAuth": "your-key"
  }
}
```

**注意：** 出于安全考虑，`externalServerAuth` 可能不会在响应中显示。

### 4. 检查浏览器控制台

打开浏览器开发者工具（F12），查看 Console 和 Network 标签：

**Console 标签：**
```javascript
// 应该看到类似的日志
[WatchRoom] Connected to server
// 或错误信息
[WatchRoom] Connection error: ...
```

**Network 标签：**
- 查找 `socket.io` 相关的请求
- 检查请求头是否包含 `Authorization: Bearer your-key`
- 查看响应状态码（401 表示认证失败）

### 5. 手动测试连接

在浏览器控制台运行以下代码测试连接：

```javascript
// 替换为你的实际值
const serverUrl = 'https://your-watch-room-server.com';
const authKey = 'your-secret-auth-key';

const socket = io(serverUrl, {
  auth: {
    token: authKey
  },
  extraHeaders: {
    Authorization: `Bearer ${authKey}`
  }
});

socket.on('connect', () => {
  console.log('✅ 连接成功！');
});

socket.on('connect_error', (err) => {
  console.error('❌ 连接失败:', err.message);
});
```

## 常见问题和解决方案

### 问题 1: AUTH_KEY 不匹配

**症状：** 持续显示 "Unauthorized connection attempt"

**解决方案：**
1. 确认两边的 AUTH_KEY 完全一致（包括大小写、空格）
2. 重新生成一个新的 AUTH_KEY：
   ```bash
   # 生成随机密钥
   openssl rand -base64 32
   ```
3. 同时更新服务器和 Vercel 的环境变量
4. 重新部署两边的服务

### 问题 2: 环境变量未生效

**症状：** API 返回的配置不正确

**解决方案：**
1. 在 Vercel 设置环境变量后，必须重新部署
2. 确认环境变量应用到了正确的环境（Production/Preview/Development）
3. 清除浏览器缓存并刷新页面

### 问题 3: CORS 问题

**症状：** 浏览器控制台显示 CORS 错误

**解决方案：**
1. 检查观影室服务器的 `ALLOWED_ORIGINS` 环境变量
2. 确保包含了 MoonTVPlus 的域名：
   ```env
   ALLOWED_ORIGINS=https://tv.886200.xyz,https://www.tv.886200.xyz
   ```
3. 或者临时设置为 `*` 进行测试（不推荐用于生产环境）

### 问题 4: URL 格式错误

**症状：** 无法连接到服务器

**解决方案：**
确保 URL 格式正确：
- ✅ `https://your-server.com`
- ✅ `http://your-server.com:3001`
- ❌ `your-server.com` （缺少协议）
- ❌ `https://your-server.com/` （末尾不要斜杠）

## 快速修复步骤

### 步骤 1: 生成新的 AUTH_KEY

```bash
# 生成一个强密钥
openssl rand -base64 32
# 或者
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

### 步骤 2: 更新观影室服务器

```bash
# Railway/Render: 在控制台更新环境变量
AUTH_KEY=新生成的密钥

# Docker: 更新 .env 文件
echo "AUTH_KEY=新生成的密钥" > .env
docker-compose restart

# PM2: 更新环境变量并重启
export AUTH_KEY=新生成的密钥
pm2 restart watch-room-server
```

### 步骤 3: 更新 Vercel 环境变量

1. 登录 Vercel 控制台
2. 进入项目设置 -> Environment Variables
3. 添加或更新：
   ```
   WATCH_ROOM_ENABLED=true
   WATCH_ROOM_SERVER_TYPE=external
   WATCH_ROOM_EXTERNAL_SERVER_URL=https://your-server.com
   WATCH_ROOM_EXTERNAL_SERVER_AUTH=新生成的密钥
   ```
4. 点击 "Save"
5. 重新部署项目

### 步骤 4: 验证连接

1. 访问 `https://tv.886200.xyz/api/server-config`
2. 确认 `WatchRoom.enabled` 为 `true`
3. 确认 `WatchRoom.serverType` 为 `external`
4. 确认 `WatchRoom.externalServerUrl` 正确
5. 在 MoonTVPlus 创建观影室测试

## 调试技巧

### 查看详细日志

**观影室服务器：**
```bash
# Docker
docker-compose logs -f watch-room-server

# PM2
pm2 logs watch-room-server --lines 100

# Railway/Render
# 在控制台查看 Logs 标签
```

**MoonTVPlus（浏览器）：**
1. 打开开发者工具（F12）
2. Console 标签查看连接日志
3. Network 标签查看 WebSocket 连接

### 临时禁用认证测试

**仅用于调试，不要在生产环境使用！**

临时修改观影室服务器的 `src/index.ts`：

```typescript
// 注释掉认证检查
allowRequest: (req, callback) => {
  // const auth = req.headers.authorization;
  // if (auth === `Bearer ${AUTH_KEY}`) {
  callback(null, true);
  // } else {
  //   console.log('[WatchRoom] Unauthorized connection attempt from:', req.headers.origin);
  //   callback('Unauthorized', false);
  // }
},
```

如果这样可以连接，说明问题确实是认证配置。

## 联系支持

如果以上步骤都无法解决问题，请提供以下信息：

1. Vercel 环境变量截图（隐藏敏感信息）
2. 观影室服务器日志
3. 浏览器控制台错误信息
4. `/api/server-config` 的响应内容

---

**提示：** 最常见的问题是 AUTH_KEY 不匹配或环境变量未生效。确保两边的密钥完全一致，并且在修改后重新部署了服务。
