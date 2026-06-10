# 选项打散与判分协作关系分析

本文针对后端两个方法的协作关系给出分析结论：

- `backend/internal/service/question_service.go` 中的 `GetQuizQuestions`
- `backend/internal/service/attempt_service.go` 中的 `Submit`

## 一、关键代码事实

### GetQuizQuestions（下发题目）

- 从数据库加载题目及其 `Options`。
- 将每道题的选项映射为 `StudentOption{ID, Content}`，其中 `ID` 直接取自数据库中 `QuestionOption` 记录的真实主键 `opt.ID`。
- 通过 `r.Shuffle(len(opts), func(i, j int){ opts[i], opts[j] = opts[j], opts[i] })` 仅交换切片元素位置，不修改元素内部字段。
- 返回给学生的每个选项只携带 `ID` 和 `Content`，不携带 `IsCorrect`。

### Submit（提交判分）

- 按 `req.Answers` 中出现的 `QuestionID` 批量加载题目（含选项）。
- 对每个回答执行：
  1. 通过 `questionMap[answer.QuestionID]` 定位题目，找不到则整次提交按非法处理。
  2. 在该题的 `question.Options` 中遍历，比较 `opt.ID == answer.OptionID`。
  3. 若命中，读取该选项数据库中的 `IsCorrect` 决定是否计分；若未命中，整次提交按非法处理。

## 二、问题分析

### 1. 为什么 GetQuizQuestions 打散选项顺序不会导致 Submit 误判？

`Shuffle` 只是重排切片元素的位置，并没有改变任何选项的字段值。每个 `StudentOption.ID` 仍然是数据库里那条 `QuestionOption` 的真实主键，且这个 ID 与"该选项是否正确"在数据库里是一一绑定的。

学生提交回来的 `OptionID` 就是这个真实主键。Submit 在判分时是按 ID 在题目选项集合里定位的，定位之后再读取该选项自身的 `IsCorrect`。整条链路没有"按下标取第几个选项"的逻辑，因此前端展示顺序如何变化都不会影响判分结果。

### 2. Submit 的判分依赖选项的哪一项信息？与列表位置是否有关？

只依赖两项信息：

- **选项主键 `ID`**：用于在该题选项集合中精确定位。
- **该选项数据库中的 `IsCorrect`**：决定是否得分。

判分过程是"按 QuestionID 取题 → 按 OptionID 在选项集合中查找 → 读取该记录的 IsCorrect"，整个流程不依赖选项在前端列表中的下标或位置。无论正确选项被打散到第几位，只要 ID 不变，就能被正确命中并计分。

### 3. 如果 GetQuizQuestions 在打散的同时为每个选项重新生成 ID 再下发，会有什么后果？

会导致**所有作答都被 Submit 视为非法提交而整体拒绝**（返回 `ErrInvalidSubmission`）。原因：

- 重新生成的 ID 仅存在于此次响应中，数据库中没有对应记录。
- 学生提交回来的 `OptionID` 都是这些临时 ID。Submit 加载到的仍是数据库里的真实选项，二者主键不可能匹配。
- 遍历完选项后 `selectedValid` 始终为 `false`，触发 `return nil, ErrInvalidSubmission`，本次提交无法保存，也得不到任何分数。
- 即便发生 ID 偶然撞上另一条无关选项主键的极端情况，也会按那条无关选项的 `IsCorrect` 判分，结果是随机性错判。

**结论：判分正确性的前提是"下发的 OptionID 必须是数据库里该选项的真实主键"。重写 ID 会直接打破这个契约，使整个判分链路失效。**

## 三、设计要点小结

- 打散展示顺序与判分逻辑解耦：判分基于 ID + IsCorrect，与位置无关。
- 下发数据中刻意不包含 `IsCorrect`，避免泄露答案，但保留 ID 用作回传定位。
- 任何会改变"下发 ID 与数据库 ID 对应关系"的改动（如重写 ID、用前端下标代替 ID 等）都会破坏判分契约，必须避免。
