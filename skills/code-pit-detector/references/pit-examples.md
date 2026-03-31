# 常见代码坑示例

本文档提供详细的代码问题示例，帮助识别潜在的"坑"。这些示例来自真实场景。

## 1. 隐式行为不一致

### 示例：参数传入导致排序方向不同

```python
# 坑：start_time 传入时按最新返回，不传时按最老返回
if start_time:
    messages = sorted(messages, key=lambda x: x.time, reverse=True)
else:
    messages = sorted(messages, key=lambda x: x.time)  # 隐式升序
```

**问题**：调用者可能期望行为一致，但实际上传不传参数会导致完全不同的排序结果。

**修复**：显式控制排序方向，或添加明确的 `latest_first` 参数。

### 示例：Optional 参数的隐式默认值

```python
def get_users(role: str | None = None):
    if role:
        return db.query(User).filter(User.role == role).all()
    else:
        return db.query(User).filter(User.is_active == True).all()  # 隐式过滤！
```

**问题**：不传 role 时，额外添加了 `is_active` 过滤，调用者可能不知道。

---

## 2. 调试代码泄露

### 示例：异常的魔法数字

```python
# 坑：明显的调试值
history_max_minutes = 5000000000  # 应该是 30 分钟
timeout = 999999  # 测试时设的超大值
retry_count = 1  # 原本应该是 3，调试时改成 1
```

### 示例：硬编码测试数据

```python
# 坑：测试数据未清理
user_id = "test_user_123"
email = "test@example.com"
api_url = "http://localhost:8080/api"
```

### 示例：注释掉的调试代码

```python
def process_order(order):
    # print(f"DEBUG: order = {order}")  # 应该删除
    # import pdb; pdb.set_trace()  # 应该删除
    return order.save()
```

---

## 3. 边界条件缺失

### 示例：时间戳可能为负

```python
# 坑：当 history_max_minutes 很大时，start_time 可能为负
start_time = (time.time() - config.history_max_minutes * 60) * 1000

# 修复：
start_time = max(0, (time.time() - config.history_max_minutes * 60) * 1000)
```

### 示例：空列表访问

```python
# 坑：如果 messages 为空会抛异常
latest_message = messages[-1]

# 修复：
latest_message = messages[-1] if messages else None
```

**注意**：不要过度防御。如果业务逻辑保证 messages 不会为空，则不需要添加检查。

---

## 4. 变量修改未同步

### 示例：常量修改后逻辑未同步

```python
# 坑：MAX_RETRY 从 3 改成 5，但超时计算还是基于 3
MAX_RETRY = 5  # 原来是 3
timeout = 3 * RETRY_INTERVAL  # 应该用 MAX_RETRY * RETRY_INTERVAL
```

### 示例：字段重命名后未全部更新

```python
# 坑：字段从 user_id 改成 member_id，但有的地方还用旧名
class Order:
    member_id: int  # 原来是 user_id

# 某处代码
order.user_id  # 应该改成 order.member_id
```

---

## 5. 其他常见坑

以上只是常见类型，还可能遇到：

- **竞态条件**：并发访问共享资源
- **资源泄露**：文件/连接未关闭
- **类型不匹配**：字符串和数字混用
- **时区问题**：时间处理未考虑时区
- **编码问题**：字符编码处理不当

发现问题时应运用判断力，不局限于以上类别。
