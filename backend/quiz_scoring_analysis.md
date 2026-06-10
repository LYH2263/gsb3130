# 题目下发与提交判分协作机制分析

## 一、相关代码定位

- **题目下发**：[GetQuizQuestions](file:///d:/Agsb/gsb3130/backend/internal/service/question_service.go#L163-L194)
- **提交判分**：[Submit](file:///d:/Agsb/gsb3130/backend/internal/service/attempt_service.go#L62-L131)

---

## 二、问题分析

### 1. 为什么选项顺序打散不会导致正确作答被判错？

核心原因在于：**打散操作只改变选项在切片中的排列顺序，而不改变每个选项的 `ID`**。

在 [GetQuizQuestions](file:///d:/Agsb/gsb3130/backend/internal/service/question_service.go#L179-L185) 中：

```go
opts := make([]StudentOption, 0, len(q.Options))
for _, opt := range q.Options {
    opts = append(opts, StudentOption{ID: opt.ID, Content: opt.Content})
}
r.Shuffle(len(opts), func(i, j int) {
    opts[i], opts[j] = opts[j], opts[i]
})
```

可以清楚地看到：
- 先将数据库中每个选项的原始 `ID` 和 `Content` 复制到 `StudentOption`；
- 然后调用 `r.Shuffle` 交换的是切片元素的**位置**，每个 `StudentOption` 结构体内部的 `ID` 字段从未被修改。

学生在前端看到的选项顺序虽然被打乱了，但他选中某个选项后提交回来的 `OptionID`，仍然是该选项在数据库中持久化存储的原始主键 ID。

而在 [Submit](file:///d:/Agsb/gsb3130/backend/internal/service/attempt_service.go#L92-L100) 中判分时，是**重新从数据库加载题目和选项**，然后通过 `opt.ID == answer.OptionID` 做精确匹配，和选项在列表里的顺序没有任何关系。因此，打散顺序不会对判分结果产生任何影响。

---

### 2. Submit 的判分依赖什么信息，与位置是否有关？

Submit 的判分**完全依赖选项的 `ID` 标识，与选项在列表中的位置毫无关系**。具体依赖链如下：

1. **定位题目**：用学生提交的 `answer.QuestionID` 作为 key，在 `questionMap`（[第 76-79 行](file:///d:/Agsb/gsb3130/backend/internal/service/attempt_service.go#L76-L79)）中查找对应题目；
2. **校验选项合法性**：遍历该题的所有选项，用 `opt.ID == answer.OptionID` 比较（[第 93 行](file:///d:/Agsb/gsb3130/backend/internal/service/attempt_service.go#L93)）。只有 ID 完全相等才认为该 `OptionID` 属于本题；如果遍历完都找不到，`selectedValid` 保持 `false`，整次提交按非法处理（返回 `ErrInvalidSubmission`）；
3. **判定对错**：一旦 ID 匹配成功，检查该选项的 `opt.IsCorrect` 字段（[第 95 行](file:///d:/Agsb/gsb3130/backend/internal/service/attempt_service.go#L95)），若为 `true` 则计分。

判分逻辑既不关心下发给学生时选项的排列顺序，也不关心学生提交答案时选项在前端列表中的位置。它关心的只有两件事：
- 提交的 `OptionID` 是否存在于本题的选项集合中；
- 这个 ID 对应的选项的 `IsCorrect` 是否为 `true`。

因此，选项在列表中的位置与判分结果无关。

---

### 3. 如果打散时重新生成选项 ID，会有什么后果？

**后果是：学生的所有提交都会被判定为非法提交（`ErrInvalidSubmission`），整次提交直接被拒绝，无法完成答题。**

原因分析：

假设修改后 `GetQuizQuestions` 在打散顺序的同时，为每个选项生成一个全新的临时 ID 再下发给学生，那么：

1. 学生看到的题目选项携带的是**新生成的临时 ID**，而非数据库中的原始主键；
2. 学生提交答案时，前端把这个临时 ID 作为 `OptionID` 传回后端；
3. Submit 处理时，会重新从数据库查询该题目的所有选项，这些选项的 ID 仍然是数据库中原始的持久化 ID；
4. 在 [第 92-100 行](file:///d:/Agsb/gsb3130/backend/internal/service/attempt_service.go#L92-L100) 的循环中，`opt.ID == answer.OptionID` 永远不会成立（因为一个是数据库原始 ID，一个是临时生成的 ID，二者对不上）；
5. 循环结束后 `selectedValid` 仍为 `false`，代码进入 [第 101-103 行](file:///d:/Agsb/gsb3130/backend/internal/service/attempt_service.go#L101-L103)，直接返回 `ErrInvalidSubmission`。

即使巧合情况下新生成的 ID 与数据库中某条记录的 ID 撞车，也会导致严重的判分错误——学生选的是 A 选项，却可能被匹配成 B 选项，从而出现"选对了但判错"或"选错了但判对"的严重数据错乱问题。

**总结**：选项 `ID` 是连接"下发给学生的选项"与"数据库中标准答案"的唯一桥梁，一旦这座桥梁被替换，判分逻辑就彻底失效。`ID` 的稳定性是整个判分机制正确运行的前提。
