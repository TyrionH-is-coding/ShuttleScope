# ShuttleScope 参数计算方法

## 数据来源

所有数据来自羽毛球赛事数据供应商，包含：
- 比赛元数据（日期、赛事、轮次、选手）
- 逐分得分序列（`pts` 数组：`[[home_score, away_score], ...]`）
- 逐分耗时序列（`dt` 数组：`[interval_seconds, ...]`，点与点之间的时间间隔）
- 比赛总耗时（`dur`，分钟）

## 核心函数：`computeFullStats(matches, pid)`

对某位球员的所有比赛逐局扫描，累加统计量。

### 基础统计

| 字段 | 计算方式 |
|------|----------|
| `matchTotal` / `matchWon` | 该球员参与的比赛总数/获胜数 |
| `totalSets` / `setWon` | 总局数/赢下的局数 |
| `thirdSet` / `thirdSetWon` | 三局场次/赢下的第三局数 |
| `clutchPoints` / `clutchWon` | 比分 ≥20 后的总得分/得分次数 |

### 1. 韧性与稳定性

#### 翻盘能力 (`cb[d]`)
- `d = {4,5,6,7}` 对应落后分差阈值
- `cb[d].d` = 该球员在该局中最大落后分差 ≥ d 的局数
- `cb[d].w` = 上述局中最终赢下的局数
- 翻盘率 = `cb[d].w / cb[d].d`

#### 领先稳定性
- `lead3SetTotal` / `lead3SetLoss`：该局中曾领先 ≥3 分 / 领先 3+ 分但输掉的局数
- 领先稳定性 = 1 - `lead3SetLoss / lead3SetTotal`
- `lead5SetTotal` / `lead5SetLoss` / `lead7SetTotal` / `lead7SetLoss` 同理

#### 落后翻盘率
- `neverTrailed` = 从头领先到尾的局数
- `trailWon` = 曾落后但赢下的局数
- 翻盘率 = `trailWon / (totalSets - neverTrailed)`
- `maxComeback` = 单局最大逆转分差

#### 领先交替
- `leadChangesSets` = 发生了领先交替的局数
- `totalLeadChanges` = 总领先交替次数

#### 止损效率
- `mcTotal` / `mcSum`：连失 2+ 分后，记录需要多少分才能得下一分
- 止损效率 = `mcSum / mcTotal`

### 2. 压制力

#### 连续得分频率
- `streak3pFreq` / `streak4pFreq` / `streak5pFreq` = 3连/4连/5连分的总次数（每局内）
- `streak3pSets` = 出现 3+ 连分的局数
- 频率 = `streak3pFreq / streak3pSets`

#### 得分自相关 r
- 将得分序列转为 `+1`（得分）/ `-1`（失分）信号序列
- 计算 Lag-1 自相关系数：
  ```
  r = Σ(sig[i]-μ)(sig[i-1]-μ) / √(Σ(sig[i]-μ)²) / √(Σ(sig[i-1]-μ)²)
  ```
- `r > 0.15` → 波动型（有连续得分/失分倾向）
- `r < -0.1` → 反转型（交替得分）
- 其他 → 稳定型

### 3. 战术弹性

#### ΔA（剩余能量）
- 找到每局中第一个达到 11 分的点（`breakIdx`）
- **断前** = `breakIdx` 之前所有分的得分率（含第 11 分那拍）
- **断后** = `breakIdx` 之后 5 分的得分率
- ΔA = 断后得分率 - 断前得分率
- **正值** = 间歇后调整有效，打得更好
- **负值** = 间歇后发挥下降，可能体能或战术原因

#### G1 → G2 进化
- G1 的 ΔA = `set1Post5W/set1Post5Pt - set1PreAllW/set1PreAllPt`
- G2 的 ΔA = `set2Post5W/set2Post5Pt - set2PreAllW/set2PreAllPt`
- 进化值 = G2 的 ΔA - G1 的 ΔA

#### 分段得分率
- **seg1**（1-11 分）：该比分区间内，得分 / 总拍数
- **seg2**（12-18 分）：同上
- **seg3**（19-21 分）：同上
- **斜率** = seg3 得分率 - seg1 得分率（正值=越打越好）

#### 消耗战得分率
- 若上一个回合耗时 >25s（`prevDt > 25`），记录当前拍
- 消耗战得分率 = `rallyStaminaWon / rallyStaminaTotal`

#### 战术休息比
- 若上一个回合耗时 >25s，记录当前拍的耗时
- 战术休息比 = `tacticalRestSum / tacticalRestN`

### 4. 关键分表现

#### 关键分得分率
- 定义：单分中 home 或 away 得分 ≥20 以后的每一拍
- 关键分得分率 = `clutchWon / clutchPoints`

#### Clutch C（关键分坚韧值）
- 失分后，连续失分几次才能得下一分
- `clutchLossDistance` / `clutchLossN`：关键分（≥18-18）时的数据
- `normalLossDistance` / `normalLossN`：普通分时的数据
- Clutch C = 普通分恢复距离 - 关键分恢复距离（值越大=关键分越坚韧）

#### Clutch Δt
- 失分后下一拍的耗时（反映心理调整时间）
- `clutchLossDtSum / clutchLossDtN`：关键分场景
- `normalLossDtSum / normalLossDtN`：普通分场景

#### 末段顽强值
- 在落后 ≥5 分且处于最后 5 拍的场景中
- `antiSurrenderDtSum / antiSurrenderDtN` = 平均出球耗时
- 值越大 = 不轻易放弃

### 5. 体能与节奏

#### 体力战表现
- `lateSets`：比赛总耗时 ≥60 分钟且 <150 分钟的局数
- `lateStreak3pFreq`：体力战中的 3+ 连分频率
- `lateMaxStreak`：体力战中的最长连胜

#### 节奏分布
- 从 `dt` 数组提取所有逐分耗时
- 按 5 秒分桶（0-50, 50-55, 55-60, ..., ≥90）
- 绘制直方图展示球员的比赛节奏特征

## 排名算法

### 七维排名

7 个维度 × 3 个项目（男单/女单/男双）独立排序：

1. **胜率** = `matchWon / matchTotal`
2. **关键分得分率** = `clutchWon / clutchPoints`
3. **翻盘率** = `trailWon / (totalSets - neverTrailed)`
4. **第三局胜率** = `thirdSetWon / thirdSet`
5. **领先稳定性** = `1 - lead3SetLoss / lead3SetTotal`
6. **剩余能量 δA** = 断后得分率 - 断前得分率
7. **体力战得分率** = `rallyStaminaWon / rallyStaminaTotal`

### 过滤条件
- 全项目统一 ≥60 场比赛
- 每维度 TOP 10

## 样本可信度

用 🟢🟡🔴 小圆点表示：

| 指标 | 阈值 | 充足条件 |
|------|------|----------|
| 比赛 | 20 场 | ≥20 场 🟢 |
| 局 | 100 局 | ≥100 局 🟢 |
| 关键分 | 50 拍 | ≥50 拍 🟢 |
| 翻盘 4分 | 30 局 | ≥30 局 🟢 |
| 翻盘 5分 | 25 局 | ≥25 局 🟢 |
| 连失 | 40 局 | ≥40 局 🟢 |
| 消耗战 | 30 拍 | ≥30 拍 🟢 |
| 连胜频率 | 30 局 | ≥30 局 🟢 |
| H2H | 10 场 | ≥10 场 🟢 |

可信度 = `min(1, n/threshold)`，三色：
- ≥0.6 🟢（充足）
- ≥0.3 🟡（一般）
- < 0.3 🔴（不足）
