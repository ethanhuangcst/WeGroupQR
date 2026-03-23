# 微信群永久二维码系统技术方案

## 一、核心问题分析

**痛点**：微信群二维码 7 天失效，无法长期使用

**解决思路**：
- 生成一个永久的"入口二维码"
- 定期更新微信群二维码（建议每 6 天更新一次）
- 用户扫描入口二维码时，自动跳转到最新的微信群二维码

---

## 二、技术架构方案

### 方案 A：轻量级静态方案（推荐快速启动）

**技术栈**：
- **前端**：纯静态 HTML 页面 + QRCode.js
- **后端**：Serverless Functions（Vercel/Netlify/Cloudflare Workers）
- **存储**：JSON 文件 + GitHub 作为数据库
- **定时任务**：GitHub Actions / Vercel Cron

**优势**：
- 成本极低（免费）
- 部署简单
- 无需服务器维护

**架构流程**：
```
用户扫描永久二维码
    ↓
访问固定 URL（your-domain.com/group）
    ↓
Serverless Function 读取最新群二维码
    ↓
返回最新的群二维码图片
```

---

### 方案 B：完整应用方案（推荐生产环境）

**技术栈**：
- **前端**：Next.js 14 + TypeScript + Tailwind CSS
- **后端**：Next.js API Routes
- **数据库**：PostgreSQL（Supabase/PlanetScale）
- **文件存储**：Cloudflare R2 / AWS S3
- **定时任务**：Vercel Cron Jobs / node-cron
- **部署**：Vercel / Railway

**数据库设计**：
```sql
CREATE TABLE group_qrcodes (
  id SERIAL PRIMARY KEY,
  group_name VARCHAR(255) NOT NULL,
  permanent_qrcode_url VARCHAR(500) UNIQUE,
  current_qrcode_url VARCHAR(500),
  current_qrcode_expire_at TIMESTAMP,
  last_updated_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE qrcode_history (
  id SERIAL PRIMARY KEY,
  group_id INTEGER REFERENCES group_qrcodes(id),
  qrcode_url VARCHAR(500),
  valid_from TIMESTAMP,
  valid_until TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 三、核心功能模块

### 1. 永久二维码生成模块

```typescript
// 生成永久入口二维码
function generatePermanentQRCode(groupId: string): string {
  const permanentUrl = `https://your-domain.com/group/${groupId}`;
  // 使用 QRCode 库生成二维码图片
  return QRCode.toDataURL(permanentUrl);
}
```

### 2. 微信群二维码更新模块

```typescript
// 定时更新微信群二维码
async function updateWeChatGroupQRCode(groupId: string) {
  // 1. 调用微信 API 获取新的群二维码
  const newQRCode = await getWeChatGroupQRCode(groupId);
  
  // 2. 上传到存储服务
  const qrcodeUrl = await uploadToStorage(newQRCode);
  
  // 3. 更新数据库
  await updateDatabase(groupId, qrcodeUrl);
  
  // 4. 记录历史
  await saveHistory(groupId, qrcodeUrl);
}
```

### 3. 智能重定向模块

```typescript
// API: 获取最新群二维码
app.get('/group/:groupId', async (req, res) => {
  const { groupId } = req.params;
  
  // 从数据库获取最新的群二维码
  const group = await getGroupQRCode(groupId);
  
  // 检查是否即将过期（提前 1 天更新）
  if (isExpiringSoon(group.current_qrcode_expire_at)) {
    await triggerUpdate(groupId);
  }
  
  // 返回二维码图片或重定向
  res.redirect(group.current_qrcode_url);
});
```

### 4. 定时任务调度

```typescript
// 使用 node-cron 或 Vercel Cron
import cron from 'node-cron';

// 每 6 天凌晨 2 点执行
cron.schedule('0 2 */6 * *', async () => {
  const groups = await getAllGroups();
  
  for (const group of groups) {
    if (needsUpdate(group)) {
      await updateWeChatGroupQRCode(group.id);
    }
  }
});
```

---

## 四、关键技术要点

### 1. 微信群二维码获取方式

**方式 A：企业微信（推荐）**
- 使用企业微信 API
- 可以生成永久群二维码
- 需要企业微信认证

**方式 B：手动上传**
- 管理员手动上传群二维码
- 系统定时提醒更新
- 适合小型项目

**方式 C：微信机器人**
- 使用 WeChaty 等框架
- 自动获取群二维码
- 存在封号风险

### 2. 二维码过期检测

```typescript
function isExpiringSoon(expireTime: Date): boolean {
  const now = new Date();
  const hoursLeft = (expireTime.getTime() - now.getTime()) / (1000 * 60 * 60);
  return hoursLeft < 24; // 剩余时间小于 24 小时
}
```

### 3. 高可用性保障

- **缓存策略**：使用 Redis 缓存最新二维码
- **降级方案**：数据库不可用时返回缓存的二维码
- **监控告警**：二维码即将过期时发送通知

---

## 五、实施步骤

### Phase 1: MVP（1-2 周）
1. 搭建基础项目框架（Next.js）
2. 实现手动上传群二维码功能
3. 实现永久二维码生成
4. 部署到 Vercel

### Phase 2: 自动化（2-3 周）
1. 集成数据库（Supabase）
2. 实现定时任务
3. 添加过期提醒
4. 实现自动更新（如果可行）

### Phase 3: 优化（1-2 周）
1. 添加管理后台
2. 支持多个群
3. 添加访问统计
4. 优化用户体验

---

## 六、成本估算

### 方案 A（静态方案）
- 域名：¥50/年
- 其他：免费
- **总计：¥50/年**

### 方案 B（完整应用）
- 域名：¥50/年
- Vercel Pro：$20/月
- Supabase：免费（小规模）
- **总计：¥1550/年**

---

## 七、推荐方案

**建议采用渐进式开发**：

1. **第一阶段**：使用方案 A 快速验证想法
2. **第二阶段**：如果验证成功，迁移到方案 B

这样可以：
- 快速上线验证
- 降低初期成本
- 逐步完善功能

---

## 八、技术选型建议

### 前端框架
- **Next.js 14**：React 框架，支持 SSR 和 API Routes
- **TypeScript**：类型安全
- **Tailwind CSS**：快速样式开发
- **QRCode.js**：二维码生成库

### 后端服务
- **Vercel**：部署平台，支持 Serverless Functions
- **Supabase**：PostgreSQL 数据库 + 认证 + 存储
- **Cloudflare R2**：文件存储（可选）

### 定时任务
- **Vercel Cron Jobs**：简单的定时任务
- **node-cron**：更灵活的定时任务（需要持久化进程）

### 监控告警
- **Sentry**：错误监控
- **Uptime Robot**：服务监控
- **企业微信机器人**：告警通知

---

## 九、安全考虑

### 1. 访问控制
- 管理后台需要认证
- API 接口限流
- 防止恶意扫描

### 2. 数据安全
- 敏感信息加密存储
- 定期备份数据库
- HTTPS 强制加密

### 3. 隐私保护
- 不收集用户个人信息
- 访问日志脱敏处理
- 符合相关法律法规

---

## 十、扩展性设计

### 1. 多群管理
- 支持创建多个群二维码
- 每个群独立管理
- 批量操作支持

### 2. 数据分析
- 扫码次数统计
- 地域分布分析
- 时间段分析

### 3. API 开放
- 提供 RESTful API
- 支持第三方集成
- Webhook 通知

---

## 十一、风险与应对

### 风险 1：微信 API 限制
**应对**：使用手动上传方式作为备选方案

### 风险 2：服务不可用
**应对**：多地域部署 + CDN 加速

### 风险 3：二维码过期未更新
**应对**：多重提醒机制 + 自动降级

### 风险 4：成本超支
**应对**：使用免费方案 + 监控资源使用

---

## 十二、总结

本技术方案提供了从简单到完整的渐进式解决方案，建议：

1. **快速验证**：先使用方案 A 验证需求
2. **逐步迭代**：根据实际使用情况优化
3. **持续改进**：收集用户反馈，不断完善

技术选型遵循以下原则：
- **简单优先**：能用简单方案就不用复杂方案
- **成本可控**：优先使用免费或低成本方案
- **可扩展**：架构设计考虑未来扩展需求
- **易维护**：选择成熟稳定的技术栈
