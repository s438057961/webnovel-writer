# Genre Taxonomy Convergence Plan

日期：2026-06-04

## 目标

把题材体系收敛到 CSV 已经采用的 15 个 `canonical_genre`，同时保留 37 个中文题材模板作为初始化阶段的可叠加预设。

一句话原则：

> CSV canonical 是主干；模板是预设；套路、形式、平台细分全部标签化。

## 当前判断

当前运行链路里，`templates/genres/*.md` 只被 `/webnovel-init` 使用：

- `webnovel-init/SKILL.md` 要求用户选题材后按需读取 `templates/genres/`。
- `init_project.py` 按用户传入的 `genre` 精确读取 `templates/genres/{genre}.md`，并写入新项目的 `设定集/世界观.md`。
- `plan/write/query/review` 不直接读取这 37 个模板。

CSV 层已经相对收敛：

- 9 张 CSV 的 `适用题材` 只允许 15 个 canonical 或 `全部`。
- `validate_csv.py` 当前可做到 0 errors / 0 warnings。

主要漂移点：

- `templates/genres/` 的 37 个文件名混合了 canonical、平台细分、legacy alias、套路标签和形式标签。
- `reference_search.py` 不能解析所有模板名。
- `webnovel-init/SKILL.md` 的题材集合仍是混合列表，容易把非 canonical 写进 `state.json.project.genre`。
- `题材与调性推理.csv` 的 `题材/流派` 实际更像 `route_tag`，不全是 platform tag。

## 目标模型

### 1. 硬题材枚举

唯一硬枚举继续使用 CSV 方案：

```text
都市 玄幻 仙侠 奇幻 科幻
历史 悬疑 游戏 古言 现言
幻言 年代 种田 快穿 衍生
```

这些值用于：

- CSV `适用题材`
- `裁决规则.csv` 的 `题材`
- Story System 的 `canonical_genre`
- `reference_search.py --genre`
- 新项目 `state.json.project.genre`

### 2. 模板预设

37 个 `templates/genres/*.md` 不再被视为硬题材，而是 template preset。

每个 preset 必须声明：

- `template_name`
- `canonical_genre`
- `template_type`
- `route_tags`
- `trope_tags`
- `format_tags`
- `aliases`

例子：

```csv
template_name,canonical_genre,template_type,route_tags,trope_tags,format_tags,aliases
都市异能,都市,route,都市异能,,,"异能|现代异能"
规则怪谈,悬疑,route,规则怪谈,,,"规则动物园|规则类"
系统流,玄幻,trope,,系统流,,"系统|系统文"
知乎短篇,现言,format,,,知乎短篇,"知乎体|盐选|小程序短篇"
西幻,奇幻,route,西幻,,,"西方奇幻"
```

### 3. 用户输入解析结果

用户输入不直接等于硬题材。解析结果应该是结构化对象：

```json
{
  "user_genre_label": "知乎短篇风的规则怪谈",
  "canonical_genre": "悬疑",
  "route_tags": ["规则怪谈"],
  "trope_tags": [],
  "format_tags": ["知乎短篇"],
  "template_presets": ["规则怪谈", "知乎短篇"]
}
```

## 改动范围

### 必改

- `webnovel-writer/templates/genres/*.md`
  - 标题中文化。
  - 不第一阶段移动文件，先降低风险。
- `webnovel-writer/templates/genres/index.csv`
  - 新增模板索引，覆盖 37 个模板。
- `webnovel-writer/scripts/reference_search.py`
  - 接入模板名/别名到 canonical 的解析。
- `webnovel-writer/scripts/init_project.py`
  - init 时先 resolve genre。
  - `state.json.project.genre` 写 canonical。
  - 额外保存用户原始题材与 tags。
  - 读取模板时按 index 加载 preset，而不是只按原始 `genre` 精确找文件。
- `webnovel-writer/skills/webnovel-init/SKILL.md`
  - 题材集合改为 15 个 canonical。
  - 说明可输入模板预设/套路/形式，但必须映射到 canonical。
- `webnovel-writer/scripts/validate_csv.py`
  - 增加模板索引覆盖校验。
  - 校验所有 template preset 都能映射到 canonical。

### 应改

- `webnovel-writer/references/csv/genre-canonical.md`
  - 明确 `题材与调性推理.csv` 的 `题材/流派` 是 `route_tag`，不是纯 platform tag。
- `webnovel-writer/references/csv/README.md`
  - 补充 template preset 与 canonical 的关系。
- `webnovel-writer/references/index/reference-loading-map.md`
  - 更新 init 阶段题材模板加载规则。
- 相关 tests
  - `reference_search` genre resolve 测试。
  - `init_project` 模板加载测试。
  - template index 覆盖测试。

### 暂不改

- 不大规模重写 9 张 CSV 内容。
- 不删除 37 个模板。
- 不立即把 `templates/genres/` 拆成 `canonical/` 和 `presets/` 子目录。
- 不改变老项目读取逻辑的兼容路径。
- 不把 `genre-profiles.md` 作为主真源，只保留 fallback 定位。

## 分阶段计划

### Phase 1: 建立模板索引和校验

范围：

- 新增 `templates/genres/index.csv`。
- 覆盖现有 37 个模板。
- 所有模板 H1 中文化，去掉英文括号。
- 新增校验：每个 `templates/genres/*.md` 必须在 index 里有一行。
- 新增校验：每行 `canonical_genre` 必须属于 15 个 canonical。

不改运行逻辑。

验证：

```powershell
python -X utf8 webnovel-writer\scripts\validate_csv.py
$env:PYTHONUTF8='1'; python -m pytest webnovel-writer\scripts\data_modules\tests\test_prompt_integrity.py -q --no-cov
```

### Phase 2: 接入 genre resolve

范围：

- 在 `reference_search.py` 中增加模板索引加载。
- `resolve_genre()` 支持：
  - canonical
  - platform tag
  - legacy value
  - template name
  - aliases
- 未识别输入保持原行为，但返回 warning 或可诊断状态。

验证：

- 增加测试覆盖以下输入：
  - `都市异能 -> 都市`
  - `规则怪谈 -> 悬疑`
  - `知乎短篇 -> 现言` 或按 index 配置值
  - `西幻 -> 奇幻`
  - `系统流 -> 玄幻` 默认映射

### Phase 3: 改 init 写入与模板加载

范围：

- `init_project.py` 接收用户原始题材。
- 解析为 `canonical_genre` 与 tags。
- `state.json.project.genre` 写 canonical。
- 新增兼容字段：

```json
{
  "project": {
    "genre": "悬疑",
    "genre_label": "知乎短篇风的规则怪谈",
    "genre_tags": {
      "route": ["规则怪谈"],
      "trope": [],
      "format": ["知乎短篇"],
      "templates": ["规则怪谈", "知乎短篇"]
    }
  }
}
```

- 生成世界观时加载：
  - canonical 对应模板（如存在）
  - template preset 对应模板

兼容策略：

- 老项目如果 `project.genre` 是非 canonical，运行时通过 resolver 兼容。
- 新项目只写 canonical。

验证：

- init 项目测试。
- Story System route 测试。
- prompt integrity。
- 全量 pytest。

### Phase 4: 更新 skill 与文档

范围：

- `webnovel-init/SKILL.md`
  - 主体题材只展示 15 个 canonical。
  - 说明模板预设/套路/形式的输入会被映射。
- `references/csv/genre-canonical.md`
  - 改清楚 route_tag / trope_tag / format_tag。
- `references/csv/README.md`
  - 写明 CSV 只接受 canonical，模板 index 负责用户输入层。

验证：

- prompt integrity。
- 文档关键字断链搜索。

### Phase 5: 可选目录重构

只有前四阶段稳定后再做。

目标结构：

```text
templates/genres/
  index.csv
  canonical/
    都市.md
    玄幻.md
    ...
  presets/
    都市异能.md
    规则怪谈.md
    知乎短篇.md
    系统流.md
```

这一步改动范围大，建议单独 PR/commit。

## 建议提交拆分

1. `docs(genres): document taxonomy convergence plan`
2. `chore(genres): add template index and normalize headings`
3. `feat(genres): resolve template presets to canonical genres`
4. `feat(init): persist canonical genre and genre tags`
5. `docs(genres): update init and csv taxonomy guidance`
6. 可选：`refactor(genres): split canonical and preset templates`

## 风险与控制

- 风险：`state.json.project.genre` 从细分名改为 canonical 后，用户肉眼看到的信息变少。  
  控制：保留 `genre_label` 和 `genre_tags`。

- 风险：`系统流`、`知乎短篇` 等默认映射可能有争议。  
  控制：index 中标注 `template_type`，并允许 init 交互时让用户确认 canonical。

- 风险：一次性移动模板会造成路径断裂。  
  控制：第一阶段只加 index，不移动文件。

- 风险：CSV route 表的 `题材/流派` 命名与语义不一致。  
  控制：先文档改名为 route_tag 语义，代码字段兼容不动，后续再考虑列名迁移。

## 完成标准

- 37 个模板全部有 index 映射。
- 所有模板标题纯中文。
- `validate_csv.py` 同时校验 CSV 与模板 index。
- `reference_search.py` 能解析所有模板名到 canonical。
- 新 init 项目写入 canonical `project.genre`。
- 旧项目非 canonical genre 不崩，能 fallback resolve。
- prompt integrity 与全量 pytest 通过。
