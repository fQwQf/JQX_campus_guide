### 校园导游服务平台 API 文档整理

---

## 一、导游管理

### 导游资质审核
```http
POST /api/campus/guides/verify
```
**多级审核流程**：
```json
{
  "student_id": "2020302112345",
  "id_card_front": "base64_img",
  "tour_guide_cert": "pdf_url"  // OCR识别关键信息
}

// 审核状态机
待初审 → 院系审核 → 保卫处备案 → 生效
```

### 导游档案管理
```http
PUT /api/campus/guides/{guide_id}/profile
```
**多媒体校验规则**：
```javascript
const validateMedia = (files) => {
  return files.every(file => 
    (file.type === 'video' && file.duration <= 30) || 
    (file.type === 'image' && file.ratio === 4/3)
  );
};
```

---

## 二、时间管理

### 时间槽批量设置
```http
POST /api/guide/schedule
```
**冲突检测算法**：
```python
def detect_conflict(existing, new_slot):
    return any((new_start < exist_end) and (new_end > exist_start) 
               for (exist_start, exist_end) in existing)
```

### 临时关闭时间段
```http
PATCH /api/campus/timeslots/block
```
**请求示例**：
```json
{
  "start": "2023-12-24T14:00:00",
  "end": "2023-12-24T16:00:00",
  "reason": "期末考试"
}
```

---

## 三、订单管理

### 多模式订单创建
```http
POST /api/campus/orders
```
**模式参数**：
```json
{
  "mode_type": "抢单|派单|指定",  // 枚举值
  "requirements": {
    "language": "en",         // 派单模式必填
    "preferred_guides": ["G1001"]  // 指定模式必填
  }
}
```

### 智能派单引擎
```http
POST /api/campus/orders/dispatch
```
**推荐算法**：
```python
sorted(guides, 
       key=lambda x: x['rating']*0.6 + x['response_rate']*0.4, 
       reverse=True)[:3]
```

---

## 四、评价管理

### 差评审核
```http
POST /api/campus/reviews/appeal
```
**申诉流程跟踪**：
```json
{
  "appeal_id": "AP20231128001",
  "current_stage": "校方复核",
  "deadline": "2023-12-05T23:59:59"
}
```

---

## 五、财务管理

### 收益提现
```http
POST /api/campus/finance/withdraw
```
**风控规则**：
```java
if (withdrawHistory.stream()
    .filter(w -> w.getCreateTime().isAfter(now.minusDays(1)))
    .count() >= 3) {
    throw new BizException("单日提现不得超过3次");
}
```

### 分账回调
```http
POST /api/campus/finance/notify
```
**幂等性处理**：
```sql
CREATE TABLE idempotent_keys (
    key VARCHAR(64) PRIMARY KEY,
    created_at TIMESTAMP
);
```

---

## 六、认证体系

### 用户注册
```http
POST /api/v1/auth/register
```
**密码策略**：
```javascript
/^(?=.*[A-Z])(?=.*[a-z])(?=.*\d)(?=.*[!@#$%^&*]).{8,20}$/
```

### 登录
```http
POST /api/v1/auth/login
```
**OAuth2.0流程**：
```
用户 → 服务端换取令牌 → 返回Token
```

---

## 七、Token管理

### Token刷新
```http
POST /api/v1/auth/refresh
```
```json
{
  "refresh_token": "eyJhbGciOiJIUz...",
  "device_id": "a1b2c3d4" 
}
```

### 强制登出
```http
DELETE /api/v1/auth/logout
```
**黑名单机制**：
```go
redis.Setex("jwt_blacklist:"+token, "1", expiry)
```

---

## 八、密码管理

### 密码重置
```http
POST /api/v1/auth/password/reset-request
```
**多通道支持**：
```json
{
  "identity": "13812345678",
  "channel": "sms|email"
}
```

---

**响应状态码规范**  
| 状态码 | 说明                  |
|--------|----------------------|
| 200    | 请求成功             |  
| 201    | 资源创建成功         |
| 400    | 客户端参数错误       |
| 409    | 资源冲突             |
| 500    | 服务器内部错误       |

**注意事项**  
1. 所有时间参数需使用ISO 8601格式  
2. 文件上传需在前端进行格式校验  
3. 敏感操作需进行身份验证（Bearer Token）