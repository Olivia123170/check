# CSV Check 技术文档

适用版本：1.0.0、1.1.0
文档状态：基于实际运行结果编写，非推测
最后核对：2026-10-28

---

## 1. 功能定位

CSV Check 是一个**只读的列名校验工具**，用于在把学生成绩表交给导入系统之前，确认必需的列名是否存在。

### 1.1 它做什么

| 能力 | 说明 |
|---|---|
| 表头读取 | 读 CSV 第一行作为列名 |
| 表头合法性检查 | 列名不得为空、不得重复 |
| 行完整性检查 | 每条记录的字段数必须与表头列数一致 |
| 必需列比对 | 检查指定列名是否都在表头中（区分大小写） |
| 报告输出 | 固定七行文本，含行数、列名、缺失项、结论 |
| 退出码 | 用 0 / 1 / 2 区分三种结果，便于脚本判断 |

### 1.2 它不做什么（重要边界）

以下事项**不在**校验范围内，文档中不应暗示它能保证：

- **不修改文件**。全程只读，不写回、不修复、不重排。
- **不校验数据内容**。`score` 列里写 `abc`、写空值、写负数，都不影响结果。
- **不计算任何统计量**。没有平均分、没有及格率。
- **不检查数据行是否重复**、不检查学号是否重复。
- **不保证导入成功**。列名齐全只是导入的前提条件之一，格式、类型、业务规则仍可能让导入失败。
- **不自动修复**。发现问题只报告，不给改法。

结论：`PASS` 的准确含义是「必需列名齐全且文件结构自洽」，**不是**「这张表合格」。

---

## 2. 依赖

### 2.1 运行时依赖

| 项目 | 要求 |
|---|---|
| Python | 建议 3.8 及以上；**已在 3.12.14 与 3.13.12 实测通过** |
| 第三方包 | **无** |

源码只用到 `f-string`、`pathlib`、`keyword-only` 参数等老特性，没有 3.9+ 的新语法，理论上 3.6 以上均可运行；但 3.8 以下未做实测，不写入保证范围。

工具只用 Python 标准库的四个模块：

```
argparse   # 命令行解析
csv        # CSV 解析
json       # 仅在 python -m csvcheck 时输出 JSON
pathlib    # 文件路径
```

因此**不需要 pip install，不需要虚拟环境，不需要联网**。有解释器即可运行。

### 2.2 文件依赖

三者必须保持同一层目录，缺一不可：

```
01_写作练习/
├── check.py              # 命令行入口
├── csvcheck/             # 实现包
│   ├── __init__.py       # inspect_csv / format_report / CsvCheckError / __version__
│   └── __main__.py       # 支持 python -m csvcheck
└── samples/              # 示例数据
    ├── students.csv
    ├── missing-name.csv
    ├── empty.csv
    └── students.tsv
```

### 2.3 操作系统差异

代码是纯 Python，跨平台。唯一差异是**解释器命令名**：

| 系统 | 命令 |
|---|---|
| macOS / Linux | `python3` |
| Windows | `py`（或 `python`） |

---

## 3. 命令行接口

### 3.1 完整帮助

```text
usage: check.py [-h] [--require REQUIRE [REQUIRE ...]] file

Check a student table without changing it.

positional arguments:
  file                  input CSV path

options:
  -h, --help            show this help message and exit
  --require REQUIRE [REQUIRE ...]
                        required column names (default: id name score)
```

### 3.2 参数说明

| 参数 | 是否必填 | 默认值 | 说明 |
|---|---|---|---|
| `file` | 必填 | — | 待检查的 CSV 路径，相对或绝对均可 |
| `--require` | 可选 | `id name score` | 自定义必需列名，可给多个，空格分隔 |
| `--delimiter` | 可选 | `,` | **仅 1.1.0 提供**。指定分隔符，制表符写作 `\t` |
| `-h` / `--help` | 可选 | — | 打印帮助并退出 |

### 3.3 参数位置陷阱（必须注意）

`--require` 使用 `nargs="+"`，会**连续吃掉后面所有非选项词**。因此它必须放在文件名**之后**：

```sh
# 正确
python3 check.py samples/students.csv --require id name

# 错误：students.csv 被 --require 吃掉，报 "file is required"
python3 check.py --require id samples/students.csv
```

报错文本：

```text
check.py: error: the following arguments are required: file
```

这个限制**只针对 `--require`**。`--delimiter` 是单值选项，不会吞掉文件名，放在文件名前后都成立（已实测）。

规避写法：把文件名放最前，或把 `--require` 放最后。

### 3.4 支持三种调用方式

```sh
# 方式一：直接运行脚本（课程使用）
python3 check.py samples/students.csv

# 方式二：作为模块运行，输出 JSON
python3 -m csvcheck samples/students.csv --require id name

# 方式三：作为库调用
```

方式二输出的是格式化 JSON，字段为 `columns` / `row_count` / `missing_columns` / `valid`。

方式三示例：

```python
from csvcheck import inspect_csv
report = inspect_csv("samples/students.csv", ["id", "name"])
# {'columns': ['id','name','score'], 'row_count': 3,
#  'missing_columns': [], 'valid': True}
```

#### 两个入口的默认值不一致（易踩）

| 入口 | `--require` 默认值 |
|---|---|
| `check.py` | `id name score` |
| `python -m csvcheck` | **空**（不要求任何列） |

后果很直接：`python -m csvcheck samples/missing-name.csv` 会返回 `valid: true`、退出码 0，因为根本没要求检查 `name`。实测：

```text
$ python3 -m csvcheck samples/missing-name.csv
{
  "columns": ["id", "score"],
  "missing_columns": [],
  "row_count": 1,
  "valid": true
}
```

同一文件用 `check.py` 检查则是 `Result: FAIL`、退出码 1。**用模块方式时必须显式给 `--require`**，否则检查形同虚设。

同理，库调用时 `required_columns` 传空列表 `[]` 表示不要求任何列，`valid` 恒为 True；它不接受 `None`。

---

## 4. 输出格式

成功读到表头后，固定输出七行：

```text
CSV Check 1.0.0
File: samples/students.csv
Data rows: 3
Columns: id, name, score
Missing columns: none
Result: PASS
PASS checks column names, not score values.
```

| 行 | 含义 | 易错点 |
|---|---|---|
| `CSV Check <版本>` | 工具版本号 | 排查问题时先确认版本 |
| `File: <路径>` | 回显传入路径 | 原样回显，不展开为绝对路径 |
| `Data rows: <n>` | 数据记录条数 | **不含表头**。4 行文件显示 3 |
| `Columns: <列名>` | 文件中实际读到的列名 | 是文件里的，不是要求的 |
| `Missing columns: <列名或 none>` | 缺失的必需列 | 顺序与 `--require` 一致 |
| `Result: PASS/FAIL` | 总判定 | 只看列名 |
| 固定提示句 | 工具自带的边界声明 | 是原始输出的一部分，不应删除 |

**`Result` 判定规则**：`missing_columns` 为空 → PASS；非空 → FAIL。

**第 3、4 行的对照价值**：当 `Columns` 里出现带空格或制表符的怪名字时，说明分隔符或导出格式有问题，而不是数据有问题。

---

## 5. 退出码

| 退出码 | 触发条件 | 含义 | 典型样例 |
|---:|---|---|---|
| `0` | 必需列齐全 | 校验通过 | `students.csv` |
| `1` | 文件可读，但有必需列缺失 | 内容不合格 | `missing-name.csv` |
| `2` | 文件无法解析（无表头、编码错、结构错） | 输入不合格 | `empty.csv` |

补充说明：

- `argparse` 自身的参数错误（缺 `file`、未知选项）**同样返回 2**，与文件解析失败共用退出码。脚本判断时若要区分，需看 stderr 内容。
- **退出码 1 不代表只有「缺列」一种原因**。`students.tsv` 在 1.0.0 下也返回 1，但根因是整行被当作单列、三列全缺，属分隔符问题。因此不能把 1 直接等同于「表头少写了某列」。

---

## 6. 输入要求

工具对输入有严格假设，不满足即拒绝或误判：

| 要求 | 说明 | 不满足的后果 |
|---|---|---|
| UTF-8 编码 | 允许带 BOM（按 `utf-8-sig` 读取） | 报 `Cannot read a UTF-8 CSV file`，退出码 2 |
| 必须有表头行 | 第一行视为列名 | 报 `File has no header.`，退出码 2 |
| 列名非空 | 单个列名不能只有空白 | 报 `Header names must not be blank.`，退出码 2 |
| 列名唯一 | 不能有两列同名 | 报 `Header names must be unique.`，退出码 2 |
| 字段数一致 | 每行字段数 = 表头列数 | 报 `Row N has X fields; expected Y.`，退出码 2 |
| 默认逗号分隔 | 1.0.0 硬编码逗号 | TSV 等文件需 1.1.0 的 `--delimiter` |

### 6.1 校验顺序

报错只报**第一个**命中的问题，顺序为：

```
打开文件 → 读表头 → 表头空值 → 表头重复 → 逐行字段数 → 比对必需列
```

关键推论：**如果某一行字段数不对，工具直接报 ERROR，不会告诉你缺哪列**。因此「补表头」和「改分隔符」是两类不同的修法，不能混。

### 6.2 表头空格不会自动清理

表头写成 `id, name,score`（`name` 前有空格）时：

```text
Columns: id,  name, score
Missing columns: name
Result: FAIL
```

因为非空检查用了 `strip()`，比对却用原字符串。从 Excel 导出的表常带这种空格，需要手工清掉。

### 6.3 引号内逗号可正确处理

```csv
id,name,score
1,"Ada, Jr",95
```

`Data rows: 1`、PASS，符合 CSV 标准。

---

## 7. 操作步骤

### 7.1 场景 A：检查标准成绩表

```sh
cd 01_写作练习
python3 check.py samples/students.csv
```

期望 `Data rows: 3`、`Missing columns: none`、`Result: PASS`，退出码 0。

### 7.2 场景 B：只要求部分列

```sh
python3 check.py samples/students.csv --require id name
```

`score` 不在要求内，因此即使缺失也不影响 PASS。

### 7.3 场景 C：遇到空文件后恢复

第一步，复现：

```sh
python3 check.py samples/empty.csv
```

```text
CSV Check 1.0.0
ERROR: File has no header.
Next step: use a readable UTF-8 file with a header row.
```

退出码 2。含义：文件为空或没有表头行，工具读不到任何列名。

第二步，换用已知正常的文件重跑：

```sh
python3 check.py samples/students.csv
```

确认 `Result: PASS` 即为恢复。若是自己的数据，则回源补上表头行再试，而不是重装工具。

### 7.4 场景 D：检查制表符分隔的文件（需 1.1.0）

```sh
python3 check.py samples/students.tsv --delimiter '\t'
```

1.0.0 会报 `unrecognized arguments: --delimiter`。升级方式见第 8 节。

### 7.5 场景 E：在脚本中调用

```sh
python3 check.py data.csv
case $? in
  0) echo "通过" ;;
  1) echo "缺列，需要改表头" ;;
  2) echo "文件读不了，检查编码或分隔符" ;;
esac
```

---

## 8. 版本差异

| 项目 | 1.0.0 | 1.1.0 |
|---|---|---|
| `--delimiter` | 不支持 | 支持，默认 `,`，`\t` 表示制表符 |
| `inspect_csv` 签名 | `(path, required_columns=())` | `(path, required_columns=(), *, delimiter=",")` |
| 分隔符校验 | — | 非单字符或为 CR/LF 时报错 |
| TSV 支持 | 误判为单列，FAIL | 加 `--delimiter '\t'` 后正常 PASS |

**升级方法**：把 `updates/v1.1/` 下的 `check.py`、`csvcheck/__init__.py`、`csvcheck/__main__.py` 复制覆盖到工具目录。`apply_update.py` 封装了这个动作，且不会动 `docs/` 里的文档。

1.1.0 实测：

```text
$ python3 check.py samples/students.tsv --delimiter '\t'
CSV Check 1.1.0
File: samples/students.tsv
Data rows: 2
Columns: id, name, score
Missing columns: none
Result: PASS
```

---

## 9. 错误速查

| 报错文本 | 原因 | 处理 |
|---|---|---|
| `ERROR: File has no header.` | 文件为空，或首行是空行 | 补表头行；或确认选对了文件 |
| `ERROR: Header names must not be blank.` | 某列名为空或纯空白 | 补列名，如 `id,,score` → `id,name,score` |
| `ERROR: Header names must be unique.` | 两列同名 | 改名，通常是复制粘贴重复 |
| `ERROR: Row N has X fields; expected Y.` | 第 N 行字段数与表头不符 | 检查该行是否漏写或多写了分隔符 |
| `ERROR: Cannot read a UTF-8 CSV file: ...` | 编码非 UTF-8，或路径不存在，或传入的是目录 | 转存为 UTF-8；核对路径 |
| `ERROR: Delimiter must be one character...` | `--delimiter` 传了多字符或换行 | 传单个字符，制表符写 `\t` |
| `check.py: error: the following arguments are required: file` | `--require` 放到了文件名前面 | 把文件名移到最前 |
| `unrecognized arguments: --delimiter` | 用的是 1.0.0 | 升级到 1.1.0 |

**通用提示句的局限**：所有 `ERROR` 分支都会打印同一句 `Next step: use a readable UTF-8 file with a header row.`。这句对编码类错误贴切，对「字段数不符」「分隔符非法」并不精准。排查时应以 `ERROR:` 那一行为准，把 `Next step` 当作兜底建议。

---

## 10. 已知限制

1. **空数据表会 PASS**。只有表头、没有数据行的文件，`Data rows: 0` 且 `Result: PASS`。如果业务上空表应当拒绝，需调用方额外判断 `row_count > 0`。
2. **表头空格不清理**，见 6.2。
3. **只认一个分隔符**。混合分隔、分号分隔都在 1.1.0 里靠 `--delimiter` 逐个指定。
4. **不做字段类型转换**。全部按字符串处理，`95` 与 `95.0` 无区别（反正不校验）。
5. **`--require` 无法表达「可选」**。一次调用只有一组必需列。
6. **两个入口默认检查项不同**，见 3.4，模块方式不传 `--require` 等于不检查。
7. **无 `--quiet` / `--json` 开关**。脚本要 JSON 只能走 `python -m csvcheck`，两者输出格式不同。

---

## 附：本文档的事实来源

所有输出片段均在本机实际执行所得，非按源码推测：

- 版本 1.0.0 与 1.1.0 分别运行 4 个官方样例与 8 个自建边界样例
- 退出码由 `$?` 逐一读取确认
- 边界样例含：字段数不齐、表头重复、表头空值、仅表头无数据、带 BOM、非 UTF-8 字节、纯空行、表头带空格、引号内逗号、传入目录、参数位置错误
