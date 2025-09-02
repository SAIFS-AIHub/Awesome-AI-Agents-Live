# 并发文件访问解决方案

## 问题描述

当同时运行多个进程（如get cite和analyze）时，会出现文件保存冲突的问题，因为多个进程可能同时读写同一个文件（如`papers.json`、`analyses.json`等）。

## 解决方案

我们实现了一个基于文件锁的解决方案，确保同一时间只有一个进程能够访问特定的文件。

### 核心组件

1. **FileLock类** (`utils/file_lock.py`)
   - 使用`fcntl`实现文件锁
   - 支持超时机制
   - 自动释放锁

2. **安全文件操作函数**
   - `safe_json_load()`: 安全读取JSON文件
   - `safe_json_dump()`: 安全写入JSON文件
   - `safe_file_operation()`: 通用文件操作上下文管理器

### 使用方法

#### 基本用法
```python
from utils.file_lock import safe_json_load, safe_json_dump

# 安全读取
data = safe_json_load('data/papers.json')

# 安全写入
safe_json_dump(data, 'data/papers.json', indent=2, ensure_ascii=False)
```

#### 高级用法
```python
from utils.file_lock import safe_file_operation

# 自定义文件操作
with safe_file_operation('data/papers.json', 'w') as f:
    f.write("custom content")
```

### 并发运行示例

现在可以安全地同时运行多个进程：

```bash
# 终端1：运行分析
python scripts/run.py --analyze-only --model gpt-4

# 终端2：同时运行引用数获取
python agent/fetch_citation_counts.py

# 终端3：同时运行状态同步
python scripts/sync_paper_status.py
```

### 锁文件

每个被保护的文件都会有一个对应的`.lock`文件：
- `data/papers.json` → `data/papers.json.lock`
- `data/analyses.json` → `data/analyses.json.lock`

### 超时机制

默认超时时间为300秒（5分钟），可以通过参数调整：

```python
# 设置60秒超时
data = safe_json_load('data/papers.json', timeout=60)
```

### 错误处理

- 如果无法获取锁，会等待并重试
- 超时后会抛出`TimeoutError`
- 锁会在进程结束时自动释放

## 优势

1. **数据一致性**: 避免文件损坏和数据丢失
2. **并发安全**: 支持多个进程同时运行
3. **自动管理**: 锁的获取和释放都是自动的
4. **向后兼容**: 现有代码无需大幅修改
5. **性能优化**: 只在必要时等待锁

## 注意事项

1. 确保所有文件操作都使用安全的函数
2. 避免长时间持有锁
3. 合理设置超时时间
4. 在Linux/Unix系统上运行（依赖fcntl） 