# 题目下发与提交判分协作机制分析

## 核心数据流

```
数据库 QuestionOption { ID, QuestionID, Content, IsCorrect }
        │
        ├─ GetQuizQuestions ──→ StudentOption { ID, Content }  (顺序打散，ID 不变)
        │        ↓
        │    学生选择某选项，提交 { QuestionID, OptionID }
        │        ↓
        └─ Submit ──→ 从数据库重新加载 QuestionOption，按 OptionID 匹配，检查 IsCorrect
```

---

## 问题 1：GetQuizQuestions 对选项顺序的打散，为什么不会导致 Submit 把本应正确的作答判成错误？

关键在于 **打散操作只改变了选项在切片中的排列顺序，而没有改变每个选项携带的 ID**。

具体来看 [GetQuizQuestions](file:///d:/Agsb/gsb3130/backend/internal/service/question_service.go#L163-L194) 的实现：

```go
opts := make([]StudentOption, 0, len(q.Options))
for _, opt := range q.Options {
    opts = append(opts, StudentOption{ID: opt.ID, Content: opt.Content})  // ← ID 来自数据库原值
}
r.Shuffle(len(opts), func(i, j int) {
    opts[i], opts[j] = opts[j], opts[i]  // ← 仅交换切片中的位置
})
```

`StudentOption` 结构体只暴露 `ID` 和 `Content` 两个字段，其中 `ID` 直接取自数据库中 `QuestionOption.ID`（主键）。`Shuffle` 做的事情仅仅是把切片中元素的位置互换——就像把一摞卡片洗牌，每张卡片上写的编号并没有被涂改。

学生端看到的是打散后的选项列表，选择后提交的是 `{ QuestionID, OptionID }`。而 [Submit](file:///d:/Agsb/gsb3130/backend/internal/service/attempt_service.go#L62-L131) 判分时，是从数据库重新加载完整的 `QuestionOption`（包含 `IsCorrect` 字段），然后按 `OptionID` 逐个比对：

```go
for _, opt := range question.Options {
    if opt.ID == answer.OptionID {   // ← 按 ID 精确匹配，与位置无关
        selectedValid = true
        if opt.IsCorrect {
            correct = true
        }
        break
    }
}
```

因此，无论选项在前端以何种顺序呈现，学生选中的 `OptionID` 始终指向数据库中同一个选项记录，`IsCorrect` 的判定结果不会因展示顺序的改变而产生偏差。

---

## 问题 2：Submit 的判分到底依赖选项的哪一项信息，与选项在列表中的位置是否有关？

Submit 的判分依赖选项的 **两项信息**，且均与位置无关：

| 依赖信息 | 用途 | 是否与位置相关 |
|---------|------|--------------|
| `opt.ID`（选项主键） | 定位学生选中的是哪一个选项，即 `opt.ID == answer.OptionID` | 否——ID 是数据库主键，是唯一标识，不随列表顺序变化 |
| `opt.IsCorrect`（正确标记） | 判定被选中的选项是否为正确答案 | 否——`IsCorrect` 是选项自身的布尔属性，与其在切片中的下标无关 |

Submit 的判分逻辑完全不使用选项的下标（index）。它遍历 `question.Options` 时，唯一关心的匹配条件是 `opt.ID == answer.OptionID`，匹配成功后再读取 `opt.IsCorrect`。即使选项在数据库中的加载顺序与前端展示顺序不同，也不影响结果，因为遍历是全量扫描直到 ID 命中为止。

简言之：**判分的唯一依据是"学生提交的 OptionID 对应的数据库记录中 IsCorrect 是否为 true"，与选项在任何列表中的排列位置毫无关系。**

---

## 问题 3：假设 GetQuizQuestions 在打散顺序的同时为每个选项重新生成 ID 再下发，Submit 的判分会出现什么后果？

**后果：所有提交都会被判定为非法提交（`ErrInvalidSubmission`），系统完全无法正常判分。**

原因如下：

1. **学生拿到的 ID 是伪造的**：假设 `GetQuizQuestions` 为选项重新生成了 ID（例如自增序号 1、2、3、4），学生选择后提交的 `OptionID` 就是这些伪造的值。

2. **Submit 用伪造的 ID 去数据库中查找，永远找不到匹配项**：Submit 从数据库加载的是原始的 `QuestionOption` 记录，其 `ID` 是数据库主键（例如 101、102、103、104）。当它执行 `opt.ID == answer.OptionID` 时，用 101 去比较 1，用 102 去比较 2……永远无法命中。

3. **`selectedValid` 始终为 false**：由于没有任何 `opt.ID` 能与 `answer.OptionID` 匹配，循环结束后 `selectedValid` 仍为 `false`，代码直接返回 `ErrInvalidSubmission`：

   ```go
   if !selectedValid {
       return nil, ErrInvalidSubmission  // ← 每一题都会走到这里
   }
   ```

4. **连锁效应**：由于 `Submit` 在遍历答案时一旦遇到任何一个无效的 `OptionID` 就立即返回错误，因此只要有一道题的选项 ID 是伪造的，整次提交就会被拒绝——即使其他题目作答正确也无法得分。

**根本原因**：`GetQuizQuestions` 和 `Submit` 之间的协作契约是"选项 ID 必须与数据库主键一致"。`GetQuizQuestions` 负责将数据库中的选项（含原始 ID）安全地传递给学生端，`Submit` 负责用同样的 ID 回数据库验证。一旦 `GetQuizQuestions` 篡改了 ID，这条信任链就断裂了——学生端持有的 ID 在服务端无法被识别，判分逻辑彻底失效。

---

## 总结

| 问题 | 结论 |
|------|------|
| 打散顺序是否影响判分 | **不影响**——打散只改变切片位置，选项 ID 保持不变，Submit 按 ID 匹配而非按位置匹配 |
| 判分依赖的信息 | 依赖 `OptionID`（定位选项）和 `IsCorrect`（判定对错），与位置完全无关 |
| 重新生成 ID 的后果 | **致命**——伪造 ID 无法在数据库中找到匹配，所有提交均被判定为非法，系统判分功能完全瘫痪 |

这套机制的设计精髓在于：**将"展示逻辑"（顺序随机化）与"判分逻辑"（基于持久化 ID 的身份验证）彻底解耦**。前端展示的随机性不会渗透到后端的确定性判分中，前提是作为两者桥梁的选项 ID 必须保持全局一致性。
