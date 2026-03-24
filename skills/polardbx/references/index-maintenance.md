# 资料索引维护指南

本文档说明如何维护 PolarDB-X 资料索引，包括链接验证、重构和新增链接的方法。

## 目录

1. [链接验证方法](#链接验证方法)
2. [并发重构流程](#并发重构流程)
3. [新增链接规范](#新增链接规范)
4. [失效链接处理](#失效链接处理)

---

## 链接验证方法

### 手动验证

逐个访问链接检查可访问性：

```bash
# 使用 curl 检查链接状态
curl -I -s -o /dev/null -w "%{http_code}" "https://help.aliyun.com/..."

# HTTP 200 表示正常
# HTTP 404/403 表示失效或需要更新
```

### 批量验证脚本

创建 `scripts/validate-links.py`：

```python
#!/usr/bin/env python3
"""
批量验证资料索引中的链接有效性
"""

import re
import requests
import concurrent.futures
from pathlib import Path

def extract_links_from_md(file_path):
    """从 markdown 文件中提取所有链接"""
    content = Path(file_path).read_text(encoding='utf-8')
    # 匹配 markdown 链接: [text](url)
    pattern = r'\[([^\]]+)\]\((https?://[^\)]+)\)'
    matches = re.findall(pattern, content)
    return [(text, url) for text, url in matches]

def check_link(url, timeout=10):
    """检查单个链接状态"""
    try:
        response = requests.head(url, timeout=timeout, allow_redirects=True)
        return url, response.status_code, response.status_code == 200
    except Exception as e:
        return url, str(e), False

def validate_links_concurrent(links, max_workers=10):
    """并发验证多个链接"""
    results = []
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        future_to_url = {executor.submit(check_link, url): (text, url)
                        for text, url in links}
        for future in concurrent.futures.as_completed(future_to_url):
            text, url = future_to_url[future]
            try:
                _, status, is_valid = future.result()
                results.append({
                    'text': text,
                    'url': url,
                    'status': status,
                    'valid': is_valid
                })
            except Exception as e:
                results.append({
                    'text': text,
                    'url': url,
                    'status': str(e),
                    'valid': False
                })
    return results

def main():
    # 验证所有 references/ 下的 markdown 文件
    ref_dir = Path(__file__).parent.parent / 'references'
    md_files = list(ref_dir.glob('*.md'))

    all_links = []
    for md_file in md_files:
        links = extract_links_from_md(md_file)
        all_links.extend([(md_file.name, text, url) for text, url in links])

    print(f"共发现 {len(all_links)} 个链接，开始验证...")

    # 并发验证
    unique_links = list(set([(text, url) for _, text, url in all_links]))
    results = validate_links_concurrent(unique_links, max_workers=20)

    # 输出结果
    invalid = [r for r in results if not r['valid']]
    print(f"\n验证完成: {len(results)} 个链接")
    print(f"有效: {len(results) - len(invalid)} 个")
    print(f"失效: {len(invalid)} 个")

    if invalid:
        print("\n失效链接列表:")
        for r in invalid:
            print(f"  - [{r['text']}] {r['url']} (状态: {r['status']})")

if __name__ == '__main__':
    main()
```

运行验证：
```bash
cd ~/.qoderwork/skills/polardbx
python scripts/validate-links.py
```

---

## 并发重构流程

### 1. 分片处理

将大规模链接验证/更新任务分片并行处理：

```python
# 将链接列表分片
def chunk_list(lst, chunk_size):
    """将列表分片"""
    for i in range(0, len(lst), chunk_size):
        yield lst[i:i + chunk_size]

# 示例: 700+ 链接分 10 片，每片约 70 个
all_links = extract_all_links()  # 700+ 链接
chunks = list(chunk_list(all_links, 70))  # 10 片

# 并发处理每片
with concurrent.futures.ProcessPoolExecutor() as executor:
    results = executor.map(process_chunk, chunks)
```

### 2. 增量更新策略

| 操作类型 | 处理方式 | 标记 |
|----------|----------|------|
| 新增链接 | 追加到对应分类末尾 | 无标记 |
| 链接更新 | 保留原记录，添加新链接 | `[更新]` |
| 链接失效 | 保留记录，标记失效 | `[已失效]` |
| 删除链接 | 软删除，不物理删除 | `[已删除]` |

### 3. 重构检查清单

重构索引前确认：
- [ ] 备份原始文件
- [ ] 确定重构范围（全量/增量）
- [ ] 准备验证脚本
- [ ] 分片并发执行
- [ ] 验证结果汇总
- [ ] 更新文档版本号

---

## 新增链接规范

### 表格格式

```markdown
| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| N | [文档标题] | [完整URL] | [50字以内总结] | [关键词1、关键词2、关键词3] |
```

### 字段规范

| 字段 | 要求 | 示例 |
|------|------|------|
| 序号 | 连续数字，从1开始 | 1, 2, 3... |
| 标题 | 使用文档原始标题 | "什么是云原生数据库PolarDB分布式版" |
| 链接 | HTTPS 完整链接 | https://help.aliyun.com/zh/polardb/... |
| 一句话总结 | 准确概括核心内容 | PolarDB-X产品介绍、核心特性与适用场景 |
| 关键词 | 3-5个，顿号分隔 | 产品简介、核心特性、架构 |

### 分类归属

新增链接时选择正确分类：

**aliyun-docs.md 分类**:
- 产品简介 | 产品计费 | 动态与公告
- 快速入门 | 用户指南 | 开发指南
- SQL调优指南 | 技术白皮书 | 性能白皮书
- 最佳实践 | 常见问题 | 安全合规
- API参考 | 相关协议 | PolarDB-X 1.0 | 轻量版

**zhihu-articles.md 分类**:
- 列存索引与HTAP | 存储引擎核心技术 | 分布式事务
- 最佳实践系列 | 高可用与容灾 | 源码解读系列
- 版本发布与更新 | 产品动态与解读 | 客户案例与解决方案

**open-source-dev-guide.md 分类**:
- 官方文档站点 | GitHub仓库文档
- 社区与交流资源 | 相关组件仓库

---

## 失效链接处理

### 处理优先级

```
发现失效链接
    │
    ├─► 1. 远程更新 Skill ──► qoder skill update polardbx
    │                         或重新拉取最新版本
    │
    ├─► 2. 主站重新检索 ──► 阿里云文档中心搜索
    │                      知乎专栏搜索
    │                      GitHub 仓库查找
    │
    └─► 3. 报告问题 ──► 钉钉群 32432897
                        GitHub Issues
```

### 检索方法

**阿里云文档**:
1. 访问 https://help.aliyun.com/zh/polardb/polardb-for-xscale/
2. 使用顶部搜索框，输入原文章标题关键词
3. 或按分类导航查找

**知乎文章**:
1. 访问 https://www.zhihu.com/org/polardb-x/posts
2. 使用知乎搜索，输入文章标题
3. 按专栏分类浏览

**GitHub 文档**:
1. 访问 https://github.com/polardb/polardbx-sql/tree/main/docs
2. 使用 GitHub 文件搜索
3. 查看 README 中的文档链接

### 失效链接标记示例

```markdown
| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 1 | [已失效] 原文章标题 | ~~https://old-url~~ | 文档已迁移，请搜索新链接 | - |
| 2 | [更新] 新文章标题 | https://new-url | 更新后的文档内容 | 关键词 |
```

---

## 维护工具

### 快速命令

```bash
# 进入 skill 目录
cd ~/.qoderwork/skills/polardbx

# 统计链接数量
grep -o 'https://[^)]*' references/*.md | wc -l

# 查找特定关键词的链接
grep -n "关键词" references/aliyun-docs.md

# 按分类统计
awk '/^## /{print $2}' references/aliyun-docs.md
```

### 版本管理

维护索引时更新版本信息：

```markdown
> 来源: https://help.aliyun.com/zh/polardb/polardb-for-xscale/
> 更新时间: 2026-03-24
> 文档总数: 282篇
> 版本: v1.2.0
```

---

## 报告渠道

发现链接问题或需要更新索引：

| 渠道 | 适用场景 | 联系方式 |
|------|----------|----------|
| 钉钉群 | 快速咨询、问题反馈 | 群号: 32432897 |
| GitHub Issues | 正式问题报告、功能建议 | https://github.com/polardb/polardbx-skills/issues |
| 邮件 | 详细问题描述 | - |
