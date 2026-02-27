# Git快速指南 - 交付工作必读

**适用对象**：所有创作型Agent（Design, Echo, Dev等）  
**目的**：让你的工作成果被看见、被记录、被认可

---

## 为什么要用Git？

**简单版**：
> 你的文件改动 ≠ 交付证明  
> GitHub commit = 交付证明

**详细版**：
- 本地文件改动只有你能看见
- Git commit让整个团队看见
- 没有commit = 老大看不到 = 视为0产出

---

## 最简工作流（3步搞定）

### 场景：你刚完成一个文档/代码

```bash
# 1. 进入你的工作目录
cd /home/ubuntu/.openclaw/workspace-<你的名字>
# 例如：cd /home/ubuntu/.openclaw/workspace-echo

# 2. 添加所有改动并提交
git add .
git commit -m "完成XXX任务"

# 3. 推送到GitHub
git push
```

**就这么简单！** 3行命令，工作成果立即可见。

---

## 提交信息（commit message）怎么写？

**好的例子**：
```bash
git commit -m "完成世界观文档（850字）"
git commit -m "新增10个对话场景"
git commit -m "修复对话系统bug"
git commit -m "优化性格系统框架"
```

**不好的例子**：
```bash
git commit -m "update"  # ❌ 太模糊
git commit -m "fix"     # ❌ 修了什么？
git commit -m "做完了"   # ❌ 做完了什么？
```

**原则**：别人读了能知道你做了什么

---

## 常见问题

### Q1: 我不知道改了哪些文件，能检查吗？

```bash
git status
```

会显示：
- 红色 = 未添加的改动
- 绿色 = 已添加、待提交的改动

### Q2: 我想看具体改了什么内容？

```bash
git diff
```

显示文件的具体改动（+ 新增，- 删除）

### Q3: 我不小心提交错了，怎么办？

**方案1：修改上一次提交**（还没push的情况）
```bash
# 修改文件后
git add .
git commit --amend -m "新的提交信息"
```

**方案2：撤销上一次提交**（保留文件改动）
```bash
git reset --soft HEAD~1
# 然后重新提交
git add .
git commit -m "正确的提交信息"
```

### Q4: push失败了怎么办？

**可能原因1：需要先拉取**
```bash
git pull --rebase
git push
```

**可能原因2：权限问题**
找Kuro或老大检查Git配置

### Q5: 我有很多小改动，需要每次都commit吗？

**建议**：
- **小改动**：攒到一个完整功能再commit
- **大任务**：每完成一个模块就commit一次
- **紧急**：随时commit，别怕commit太多

**原则**：宁可多commit，别0 commit

---

## 高级技巧（可选）

### 1. 查看提交历史

```bash
git log --oneline
```

显示简洁的提交历史

### 2. 只提交部分文件

```bash
git add 文件1.md 文件2.md
git commit -m "只提交这两个文件"
```

### 3. 查看某个文件的改动历史

```bash
git log -p 文件名.md
```

---

## 工作模式集成

根据 **WORK-PROTOCOL.md**：

- **进入Work Mode** → 开始工作
- **完成工作** → `git add . && git commit -m "..." && git push`
- **汇报结果** → 附上GitHub commit链接

**commit链接格式**：
```
https://github.com/<仓库名>/commit/<commit-hash>
```

例如：
```
https://github.com/CrypticDriver/opc-workspace/commit/f739567
```

---

## 辅助脚本（更简单的方式）

如果你觉得命令太多，可以用：

```bash
bash /home/ubuntu/.openclaw/workspace/scripts/submit-work.sh "完成XXX任务"
```

**这个脚本会自动**：
1. 添加所有改动
2. 提交
3. 推送
4. 显示commit链接

---

## 最后的忠告

> **Git不会咬人，但0 commit会让你的工作白费**

3行命令，30秒操作，换来工作成果的永久记录。

**记住**：
- 文件改了 → `git add .`
- 提交说明 → `git commit -m "..."`
- 推送GitHub → `git push`

**就这么简单** 🚀

---

**有问题？**  
@Kuro 随时帮你
