# Py2ONTO 本体构建工具（`py2onto.py`）使用手册 · 中文版
> 本手册对应覆盖**命令行（CLI）**和 **Python API** 两种用法。


---

## 1. 简介

Py2ONTO 是一个基于 [owlready2](https://github.com/pypa/owlready2) 的 OWL 本体构建器。它把**类层级、注解属性、对象属性、个体（实例）**以 CSV 模板或 Python 调用的方式组装成 RDF/XML 格式的 `.owl` 文件。

两个设计要点：

- **CSV 即审计链**：同一份 CSV 输入 → 同一份 OWL 输出，便于复现与版本管理。
- **外部 IRI 复用**：类模板的 `IRI` 列可填入 GO / MONDO / NCIt 等标准术语的规范 IRI，避免“同义不同 IRI”。

脚本本身**不调用大语言模型**；只有在使用 `-s` 外部术语检索时才需要 API Key。

---

## 2. 环境要求与安装

| 项目 | 要求 |
|---|---|
| Python | ≥ 3.10（本机验证环境：3.13） |
| 依赖 | `pandas`、`owlready2`|

```bash
pip install pandas owlready2
```

---

## 3. 三步快速开始

```bash
# ① 生成 4 个空 CSV 模板（路径末尾必须带目录分隔符，见第 11 节）
python py2onto.py -i "http://example.org/demo.owl#" -c "./templates/"

# ② 填写 ./templates/templateCLASS.csv（示例见 5.2）
# ③ 构建 OWL
python py2onto.py -i "http://example.org/demo.owl#" \
  -n "./templates/templateCLASS.csv" \
  -p "./demo.owl"
```

> `-c` 只生成模板并立即退出，**不会**生成 `.owl` 文件。

---

## 4. 命令行参数

| 参数 | 说明 |
|---|---|
| `-i`, `--base_iri` | **必填**。本体基础 IRI，必须是 `http://` 或 `https://` 开头；若结尾不是 `#` 或 `/`，脚本自动补 `#`。 |
| `-c`, `--save_generate_templates_path` | 生成 4 个空模板到该目录并退出。 |
| `-a`, `--annotation_property` | 注解属性 CSV 模板。 |
| `-o`, `--object_property` | 对象属性 CSV 模板。 |
| `-n`, `--init` | **类层级 CSV**；只有提供它才会保存 OWL。 |
| `-b`, `--build` | 个体（实例）CSV；仅在同时给出 `-n` 时生效。 |
| `-s`, `--standard` | 外部术语源：`bioportal`（不区分大小写）走 BioPortal，其他任意值走 MedPortal；需与 `-q` 同用。 |
| `-q`, `--ontology` | 目标本体短名，如 `go`、`hpo`、`ncit`。 |
| `-p`, `--save_path` | 输出路径，默认 `./new_onto.owl`。 |
| `-h`, `--help` | 显示帮助。 |

---

## 5. CSV 模板规范

### 5.1 通用规则

- 列名和列顺序**必须严格匹配**表头；`*definition` 中的星号是字面字符，不能写成 `definition`。
- 编码读取顺序：`utf-8-sig` → `gbk` → `gb18030`。推荐用 **UTF-8 with BOM** 保存中文 CSV。
- 多个值用 `&` 分隔，例如 `Diarrhea&Fatigue`；`$` 关系列的目标必须存在于最终的类/个体表中，而 AP/OP 的 `domain`/`range` 在命令行下允许前向引用（见 5.3）。
- 类行的**先后顺序无关**：引擎会多轮重试解析父类；但类名拼错会导致 `unresolved parent class`。
- 模板第 6 列之后可追加自定义列：`*` 前缀 = 注解属性，`$` 前缀 = 对象属性（见 5.6）。

### 5.2 类模板 `templateCLASS.csv`

支持两种表头：

```csv
# 新版（推荐，含 IRI 列）
Parent_Class,ID,label,IRI,comment,*definition

# 旧版（兼容）
Parent_Class,ID,label,comment,*definition
```

> 注意：`-c` 生成的模板只有含 `IRI` 的新版表头；旧版仅作为历史文件输入兼容。

| 列 | 说明 |
|---|---|
| `Parent_Class` | 父类 ID；顶层类必须填 `Thing`，留空会报错。 |
| `ID` | 类唯一标识，如 `InflammatoryBowelDisease`。 |
| `label` | 可读标签，如 `Inflammatory Bowel Disease`。 |
| `IRI` | 本地类留空 → 自动派生 `base_iri + ID`；复用外部术语时填完整规范 IRI，且不会被基础 IRI 覆盖。 |
| `comment` | 描述文本。 |
| `*definition` | 定义文本（作为注解属性 `definition` 写入）。 |
| `source`（可选） | 来源标签，默认 `local`；可写 `OLS`、`MONDO` 等。 |

最小示例：

```csv
Parent_Class,ID,label,IRI,comment,*definition
Thing,Animal,Animal,,A living organism.,
Animal,Dog,Dog,,A domestic canine.,
Animal,Cat,Cat,,A domestic feline.,
```

### 5.3 注解属性模板 `templateAP.csv`

```csv
ID,label,comment,domain,range,*definition
```

- `domain` / `range` 填类表中的类名，多个用 `&` 分隔（如 `Drug&Device`）；命令行支持前向引用。
- 命令行会先读本模板、再建类，最后统一补上 domain/range，因此模板里引用后建的类也可以。

### 5.4 对象属性模板 `templateOP.csv`

表头共 17 列：

```csv
ID,label,comment,FunctionalProperty,InverseFunctionalProperty,TransitiveProperty,SymmetricProperty,AsymmetricProperty,ReflexiveProperty,IrreflexiveProperty,equivalent_to,subproperty_of,inverse_of,domain,range,disjoint_with,*definition
```

7 个特性列中任意非空值即启用对应特性：

| 列 | OWL 特性 |
|---|---|
| `FunctionalProperty` | 函数性 |
| `InverseFunctionalProperty` | 逆函数性 |
| `TransitiveProperty` | 传递性 |
| `SymmetricProperty` | 对称性 |
| `AsymmetricProperty` | 非对称性 |
| `ReflexiveProperty` | 自反性 |
| `IrreflexiveProperty` | 非自反性 |

`equivalent_to` / `subproperty_of` / `inverse_of` / `disjoint_with` 填属性 ID，多个用 `&` 分隔。

### 5.5 个体模板 `templateINDIVIDUAL.csv`

```csv
Types,relation,ID,label,comment,*definition
```

| 列 | 说明 |
|---|---|
| `Types` | 所属类的 ID，必须已在类表中定义。 |
| `relation` | **必须**填 `has_individual`；空值或其他值会逐行报错。 |
| `ID` | 个体唯一标识，不能与类名相同。 |
| `label` | 可读标签。 |
| `comment` | 描述。 |
| `*definition` | 定义文本。 |

### 5.6 自定义列

| 前缀 | 语义 | 示例 |
|---|---|---|
| `*<名称>` | 注解属性；单元格值写入该属性。 | `*hasSynonym` → `Crohn disease` |
| `$<属性名>` | 对象属性；单元格值是目标实体 ID。 | `$hasSymptom` → `Diarrhea&Fatigue` |
| `$<属性名>.<提示类名>` | 同上，点号后是命名提示；实际使用的属性名是点号前的部分。 | `$treats.Disease` → `Drug_A` |

目标 ID 必须能在对应的类表或个体表中找到，否则断言失败。

---

## 6. 命令行工作流

`Process.run_parser()` 的执行顺序是固定的：

```text
Py2ONTO(base_iri)
  ├─ -c              → 生成模板并 return（不保存 OWL）
  └─ -a  注解属性模板
      -o  对象属性模板
      -n  类层级
          ├─ 补齐 AP/OP 的 domain & range
          ├─ -b  个体
          ├─ -s + -q  外部术语检索
          └─ save( -p ) → RDF/XML + 控制台统计
```

- **没有 `-n`**：只打印错误提示，**不生成文件**。
- 构造时会自动加入标准交叉引用注解属性 `hasDbXref`（来自 oboInOwl）。
- 保存格式固定为 **RDF/XML**。

完整示例（仓库自带 IBD 数据）：

```bash
python py2onto.py \
  -i "http://bmicc.cn/IBD_py2ontove.owl#" \
  -o "py2onto-oss/ibd_example/object_properties.csv" \
  -n "py2onto-oss/ibd_example/classes.csv" \
  -p "./IBD_demo.owl"
```

成功时控制台输出：

```text
Step 1: Creating empty ontology...
Processing object property template...
Step 2: Loading class hierarchy from ...
Ontology saved to ./IBD_demo.owl
  Summary: N classes, M object properties, K annotation properties
```

其中 `K` 至少为 2：自动加入的 `hasDbXref` 加上从 `*definition` 列生成的 `definition`。

> 元数据 `.txt` 报告由 Flask 编辑器 `py2onto-oss/app.py` 生成，**本脚本不生成**。

---

## 7. Python API（编程方式）

```python
from owlready2 import Thing
from py2onto import Py2ONTO

o = Py2ONTO("https://example.org/demo.owl#")

# 类
o.add_class("Disease", "Disease", Thing, comment_value="A pathological condition")
o.add_class("Symptom", "Symptom", Thing)
o.add_class("IBD", "Inflammatory bowel disease", o._classes["Disease"])

# 复用外部规范 IRI（推荐）
o.build_reused_class(
    "MONDO_0000001", "disease", Thing,
    iri="http://purl.obolibrary.org/obo/MONDO_0000001",
    source="MONDO",
)

# 注解属性
o.create_ap("hasSynonym", "has synonym")

# 对象属性：domain/range 必须是已存在的类 ID
o.create_op("hasSymptom", "has symptom", domain="IBD", range="Symptom")

# 个体
o.add_individual("IBD", "ibd_case_001", "IBD case 1")

# 关系（首参为 owlready2 实体）
ibd = o.onto.IBD
symptom = o.onto.Symptom
o.add_one_objectProperty(ibd, "hasSymptom", symptom)

o.save("./demo_api.owl")
```

主要公开方法：

| 方法 | 用途 |
|---|---|
| `Py2ONTO(base_iri)` | 创建空本体；IRI 自动补 `#`。 |
| `create_template_file(path)` | 生成 4 个 CSV 模板；`path` 需以 `/` 或 `\` 结尾。 |
| `init(class_csv)` / `build(ind_csv)` | 从 CSV 建类 / 建个体。 |
| `add_class(name, label, super_class, ...)` | 新建或扩展类。 |
| `build_reused_class(name, label, super_class, iri, ...)` | 复用外部 IRI 的类。 |
| `create_ap(ap_name, label, **kwargs)` | 建注解属性；`kwargs` 支持 `domain`、`range`（`&` 分隔）及其他注解属性。 |
| `create_op(name, label, **kwargs)` | 建对象属性；支持 `function_list`（7 个 0/1 标志）、`equ`/`sub`/`inv`/`dis`/`domain`/`range`。 |
| `add_individual(class_name, name, label, ...)` | 建个体。 |
| `add_one_objectProperty(a, prop, b)` / `add_object_property(a, prop_obj, values)` | 追加单条关系（`prop` 可为字符串或属性对象）/ 整体替换关系列表（`prop_obj` 必须是属性对象）。 |
| `search_class_by_name(label)` | 按 label 查找类。 |
| `save(path)` | 保存为 RDF/XML。 |

`function_list` 顺序：`[Functional, InverseFunctional, Transitive, Symmetric, Asymmetric, Reflexive, Irreflexive]`，例如 `[1,0,0,0,0,0,0]`。

---

## 8. 外部术语检索

在 `py2onto.py` 同目录（或 `visual_py2onto/` 子目录）放一个 `config.json`：

```json
{
  "medportal": {
    "url": "http://medportal.bmicc.cn:8080",
    "api_key": "your-key"
  },
  "bioportal": {
    "url": "http://data.bioontology.org",
    "api_key": "your-key"
  }
}
```

```bash
python py2onto.py -i "http://example.org/onto.owl#" \
  -n "./classes.csv" \
  -s bioportal -q go \
  -p "./onto_with_xrefs.owl"
```

行为说明：

- 只接受 `prefLabel` **完全一致**（不区分大小写）且本体短名匹配的候选；命中的 `@id` 写入 `hasDbXref`。
- 本体短名取 `links.ontology` URL 的最后一段并转小写。
- 需要网络可达和有效 API Key。
- **注意**：源码注释提到环境变量 `MEDPORTAL_API_KEY` / `BIOPORTAL_API_KEY`，但当前实现**只读取 config.json**，不读取环境变量。

---

## 9. 输出与验证

- 输出：单个 `.owl` 文件，RDF/XML 格式，可用 Protégé / WebProtégé 打开。
- 控制台统计：类数（不含 `Thing`）、对象属性数、注解属性数；个体数仅在大于 0 时显示。
- 快速复核：

```python
import owlready2
onto = owlready2.get_ontology("./demo.owl").load()
print(len(list(onto.classes())), len(list(onto.individuals())),
      len(list(onto.object_properties())), len(list(onto.annotation_properties())))
```

---

## 10. 常见问题

| 问题 | 原因 | 解决 |
|---|---|---|
| `unresolved parent class` | `Parent_Class` 拼错或留空；顶层必须写 `Thing`。 | 核对父类 ID 与类表中的 `ID` 完全一致。 |
| `CSV header mismatch` | 表头列名/顺序不符，或 `definition` 少了 `*`。 | 用 `-c` 重新生成模板再填。 |
| `Domain class ... is not defined` | 属性引用了尚未创建的类。 | 命令行用 `-a`/`-o` 让引擎延后处理；API 调用先建类再建属性。 |
| `individual ... has an empty relation` | `relation` 列不是 `has_individual`。 | 固定填 `has_individual`。 |
| 中文乱码 | 文件被按错误编码读取。 | 存为 UTF-8 with BOM；引擎依次回退 gbk / gb18030。 |
| 运行后没有 `.owl` | 没给 `-n`。 | 必须提供类 CSV。 |
| `-c` 后模板文件名拼在一起 | 路径没以分隔符结尾。 | 用 `-c "./templates/"`。 |
| 环境变量设了 Key 但不生效 | 脚本不读环境变量。 | 写入 `config.json`。 |

---

## 11. 已知注意事项

1. **`-c` 路径末尾需带 `/`**：`create_template_file()` 用字符串拼接文件名，传入 `./templates` 会生成 `./templatestemplateAP.csv`。
2. **无 `-n` 不保存**：即使给了 `-a`/`-o`/`-b`，没有类模板也不会输出 OWL。
3. **`-b` 与 `-s` 仅在 `-n` 存在时被处理**。
4. **仅 RDF/XML 导出**；本脚本不含 SHACL 校验或多格式导出。
5. **类与个体的 `label` 以 `lang="en"` 存储**（即使标签是中文）。
6. **`$属性名.类名` 的点号后部分只是提示**；实际属性名取点号前部分，目标由单元格值决定。
7. **`-s` 其他值一律走 MedPortal**；只有 `bioportal` 选择 BioPortal。
8. **父类解析最多重试 50/100 轮**（>100 行时 100 轮）；仍有孤儿类即断言失败。

---

