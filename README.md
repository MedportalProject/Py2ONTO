<p align="center">
  <a href="https://github.com/MedportalProject/Py2ONTO">
    <img src="https://raw.githubusercontent.com/MedportalProject/Py2ONTO/main/logo/logo2.png"
         alt="Py2ONTO Logo"
         width="550">
  </a>
</p>

# Py2ONTO ontology development tool (`py2onto.py`) --- User Manual (English)

> This manual covers both the **command-line interface (CLI)** and the
> **Python API**.

------------------------------------------------------------------------

## Table of Contents

1.  [Overview](#1-overview)
2.  [Requirements and installation](#2-requirements-and-installation)
3.  [Quick start in three steps](#3-quick-start-in-three-steps)
4.  [CLI arguments](#4-cli-arguments)
5.  [CSV template specifications](#5-csv-template-specifications)
    -   [5.1 Common rules](#51-common-rules)
    -   [5.2 Class template](#52-class-template-templateclasscsv)
    -   [5.3 Annotation-property
        template](#53-annotation-property-template-templateapcsv)
    -   [5.4 Object-property
        template](#54-object-property-template-templateopcsv)
    -   [5.5 Individual
        template](#55-individual-template-templateindividualcsv)
    -   [5.6 Custom columns](#56-custom-columns)
6.  [CLI workflow](#6-cli-workflow)
7.  [Python API](#7-python-api)
8.  [External terminology search](#8-external-terminology-search)
9.  [Output and verification](#9-output-and-verification)
10. [FAQ](#10-faq)
11. [Known caveats](#11-known-caveats)
12. [Citation](#citation)

------------------------------------------------------------------------

## 1. Overview

Py2ONTO is an OWL ontology builder based on
[owlready2](https://github.com/pypa/owlready2). It assembles **class
hierarchies, annotation properties, object properties and individuals**
from CSV templates or Python calls, and saves a RDF/XML `.owl` file.

Two design points:

-   **CSV as audit trail**: the same CSV input yields the same OWL
    output, supporting reproducibility and version control.
-   **External IRI reuse**: the `IRI` column of the class template can
    hold canonical IRIs from GO / MONDO / NCIt, avoiding duplicate
    entities with different IRIs.

The script itself **does not call any LLM**; an API Key is only needed
for `-s` external term search.

------------------------------------------------------------------------

## 2. Requirements and installation

  Item           Requirement
  -------------- --------------------------------------
  Python         \>= 3.10 (validated locally on 3.13)
  Dependencies   `pandas`, `owlready2`

``` bash
pip install pandas owlready2
```

------------------------------------------------------------------------

## 3. Quick start in three steps

``` bash
# 1. Create four empty CSV templates (the path MUST end with a directory separator; see §11)
python py2onto.py -i "http://example.org/demo.owl#" -c "./templates/"

# 2. Fill in ./templates/templateCLASS.csv (example in §5.2)
# 3. Build the OWL file
python py2onto.py -i "http://example.org/demo.owl#" \
  -n "./templates/templateCLASS.csv" \
  -p "./demo.owl"
```

> `-c` only creates templates and exits immediately; it does **not**
> produce an `.owl` file.

------------------------------------------------------------------------

## 4. CLI arguments

  -----------------------------------------------------------------------
  Flag                                Description
  ----------------------------------- -----------------------------------
  `-i`, `--base_iri`                  **Required.** Base ontology IRI;
                                      must start with `http://` or
                                      `https://`. A trailing `#` is
                                      appended if the IRI does not end
                                      with `#` or `/`.

  `-c`,                               Write four empty templates into the
  `--save_generate_templates_path`    directory and exit.

  `-a`, `--annotation_property`       Annotation-property CSV template.

  `-o`, `--object_property`           Object-property CSV template.

  `-n`, `--init`                      **Class-hierarchy CSV**; the OWL
                                      file is saved only when this is
                                      given.

  `-b`, `--build`                     Individuals CSV; effective only
                                      together with `-n`.

  `-s`, `--standard`                  Terminology portal: `bioportal`
                                      (case-insensitive) selects
                                      BioPortal; any other value selects
                                      MedPortal. Use with `-q`.

  `-q`, `--ontology`                  Target ontology short name,
                                      e.g. `go`, `hpo`, `ncit`.

  `-p`, `--save_path`                 Output path; default
                                      `./new_onto.owl`.

  `-h`, `--help`                      Show help.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 5. CSV template specifications

### 5.1 Common rules

-   Column names and order **must match exactly**; the `*` in
    `*definition` is a literal character --- do not write `definition`.
-   Encoding fallback order: `utf-8-sig` → `gbk` → `gb18030`. Prefer
    **UTF-8 with BOM** for non-ASCII content.
-   Multiple values are separated by `&`, e.g. `Diarrhea&Fatigue`;
    targets of `$` relation columns must exist in the final
    class/individual tables, while AP/OP `domain`/`range` may be forward
    references in the CLI (see §5.3).
-   Class row order is **irrelevant**: the engine retries parent
    resolution across passes; a misspelled parent ID causes
    `unresolved parent class`.
-   Columns after the 6th may be custom: prefix `*` = annotation
    property, prefix `$` = object property (see §5.6).

### 5.2 Class template `templateCLASS.csv`

Two accepted headers:

``` csv
# New (recommended, with IRI)
Parent_Class,ID,label,IRI,comment,*definition

# Old (backward compatible)
Parent_Class,ID,label,comment,*definition
```

> Note: `-c` generates only the new header with `IRI`; the old header is
> accepted for input compatibility only.

  ------------------------------------------------------------------------
  Column                              Meaning
  ----------------------------------- ------------------------------------
  `Parent_Class`                      Parent class ID; top-level classes
                                      **must** use `Thing`.

  `ID`                                Unique class identifier,
                                      e.g. `InflammatoryBowelDisease`.

  `label`                             Human-readable label,
                                      e.g. `Inflammatory Bowel Disease`.

  `IRI`                               Leave blank for a local class
                                      (auto-derived as `base_iri + ID`);
                                      fill a canonical IRI to reuse an
                                      external term (preserved).

  `comment`                           Description text.

  `*definition`                       Definition text (stored as
                                      annotation property `definition`).

  `source` (optional)                 Provenance label; defaults to
                                      `local`; may be `OLS`, `MONDO`, etc.
  ------------------------------------------------------------------------

Minimal example:

``` csv
Parent_Class,ID,label,IRI,comment,*definition
Thing,Animal,Animal,,A living organism.,
Animal,Dog,Dog,,A domestic canine.,
Animal,Cat,Cat,,A domestic feline.,
```

### 5.3 Annotation-property template `templateAP.csv`

``` csv
ID,label,comment,domain,range,*definition
```

-   `domain` / `range` reference class IDs from the class table; use `&`
    for multiple values (e.g. `Drug&Device`). The CLI supports forward
    references.
-   The CLI reads this template before building classes and applies
    domain/range afterwards, so forward references are allowed.

### 5.4 Object-property template `templateOP.csv`

Exactly 17 columns:

``` csv
ID,label,comment,FunctionalProperty,InverseFunctionalProperty,TransitiveProperty,SymmetricProperty,AsymmetricProperty,ReflexiveProperty,IrreflexiveProperty,equivalent_to,subproperty_of,inverse_of,domain,range,disjoint_with,*definition
```

Any non-empty value in these seven columns enables the corresponding
characteristic:

  Column                        OWL characteristic
  ----------------------------- --------------------
  `FunctionalProperty`          functional
  `InverseFunctionalProperty`   inverse functional
  `TransitiveProperty`          transitive
  `SymmetricProperty`           symmetric
  `AsymmetricProperty`          asymmetric
  `ReflexiveProperty`           reflexive
  `IrreflexiveProperty`         irreflexive

`equivalent_to` / `subproperty_of` / `inverse_of` / `disjoint_with` hold
property IDs; use `&` for multiple values.

### 5.5 Individual template `templateINDIVIDUAL.csv`

``` csv
Types,relation,ID,label,comment,*definition
```

  -----------------------------------------------------------------------
  Column                              Meaning
  ----------------------------------- -----------------------------------
  `Types`                             Class ID; must already exist in the
                                      class table.

  `relation`                          **Must** be `has_individual`; empty
                                      or other values raise an error per
                                      row.

  `ID`                                Unique individual ID; must differ
                                      from its class name.

  `label`                             Human-readable label.

  `comment`                           Description.

  `*definition`                       Definition text.
  -----------------------------------------------------------------------

### 5.6 Custom columns

  -----------------------------------------------------------------------
  Prefix                  Semantics               Example
  ----------------------- ----------------------- -----------------------
  `*<name>`               Annotation property;    `*hasSynonym` →
                          the cell value becomes  `Crohn disease`
                          its value.              

  `$<property>`           Object property; the    `$hasSymptom` →
                          cell value is the       `Diarrhea&Fatigue`
                          target entity ID.       

  `$<property>.<hint>`    Same as above; the      `$treats.Disease` →
                          dotted suffix is only a `Drug_A`
                          naming hint, the base   
                          property name is used.  
  -----------------------------------------------------------------------

Target IDs must exist in the corresponding class or individual table,
otherwise an assertion error is raised.

------------------------------------------------------------------------

## 6. CLI workflow

`Process.run_parser()` follows a fixed order:

``` text
Py2ONTO(base_iri)
  ├─ -c              -> create templates and return (no OWL saved)
  └─ -a  annotation properties
      -o  object properties
      -n  classes
          ├─ apply deferred AP/OP domain & range
          ├─ -b  individuals
          ├─ -s + -q  external term search
          └─ save( -p ) -> RDF/XML + console summary
```

-   **Without `-n`**: an error is printed and **no file** is written.
-   The standard cross-reference annotation property `hasDbXref`
    (oboInOwl) is added automatically at construction.
-   The save format is always **RDF/XML**.

Complete example using the bundled IBD data:

``` bash
python py2onto.py \
  -i "http://bmicc.cn/IBD_py2ontove.owl#" \
  -o "py2onto-oss/ibd_example/object_properties.csv" \
  -n "py2onto-oss/ibd_example/classes.csv" \
  -p "./IBD_demo.owl"
```

A successful run prints:

``` text
Step 1: Creating empty ontology...
Processing object property template...
Step 2: Loading class hierarchy from ...
Ontology saved to ./IBD_demo.owl
  Summary: N classes, M object properties, K annotation properties
```

`K` is at least 2: the auto-added `hasDbXref` plus `definition` created
from the `*definition` column.

> The companion `.txt` metadata report is generated by the Flask editor
> (`py2onto-oss/app.py`), **not by this script**.

------------------------------------------------------------------------

## 7. Python API

``` python
from owlready2 import Thing
from py2onto import Py2ONTO

o = Py2ONTO("https://example.org/demo.owl#")

# classes
o.add_class("Disease", "Disease", Thing, comment_value="A pathological condition")
o.add_class("Symptom", "Symptom", Thing)
o.add_class("IBD", "Inflammatory bowel disease", o._classes["Disease"])

# reuse an external canonical IRI (recommended)
o.build_reused_class(
    "MONDO_0000001", "disease", Thing,
    iri="http://purl.obolibrary.org/obo/MONDO_0000001",
    source="MONDO",
)

# annotation property
o.create_ap("hasSynonym", "has synonym")

# object property: domain/range must reference existing class IDs
o.create_op("hasSymptom", "has symptom", domain="IBD", range="Symptom")

# individual
o.add_individual("IBD", "ibd_case_001", "IBD case 1")

# relationship (arguments are owlready2 entities)
ibd = o.onto.IBD
symptom = o.onto.Symptom
o.add_one_objectProperty(ibd, "hasSymptom", symptom)

o.save("./demo_api.owl")
```

Main public methods:

  ------------------------------------------------------------------------------------------------------
  Method                                                     Purpose
  ---------------------------------------------------------- -------------------------------------------
  `Py2ONTO(base_iri)`                                        Create an empty ontology; `#` is appended
                                                             if needed.

  `create_template_file(path)`                               Create four CSV templates; `path` must end
                                                             with `/` or `\`.

  `init(class_csv)` / `build(ind_csv)`                       Build classes / individuals from CSV.

  `add_class(name, label, super_class, ...)`                 Create or extend a class.

  `build_reused_class(name, label, super_class, iri, ...)`   Create a class that reuses an external IRI.

  `create_ap(ap_name, label, **kwargs)`                      Create an annotation property; `kwargs`
                                                             supports `domain`, `range` (`&`-separated)
                                                             and other annotation properties.

  `create_op(name, label, **kwargs)`                         Create an object property; supports
                                                             `function_list` (seven 0/1 flags),
                                                             `equ`/`sub`/`inv`/`dis`/`domain`/`range`.

  `add_individual(class_name, name, label, ...)`             Create an individual.

  `add_one_objectProperty(a, prop, b)` /                     Append one relation (`prop` may be a string
  `add_object_property(a, prop_obj, values)`                 or property object) / replace the relation
                                                             list (`prop_obj` must be a property
                                                             object).

  `search_class_by_name(label)`                              Find a class by label.

  `save(path)`                                               Save as RDF/XML.
  ------------------------------------------------------------------------------------------------------

`function_list` order:
`[Functional, InverseFunctional, Transitive, Symmetric, Asymmetric, Reflexive, Irreflexive]`,
e.g. `[1,0,0,0,0,0,0]`.

------------------------------------------------------------------------

## 8. External terminology search

Place a `config.json` next to `py2onto.py` (or in the `visual_py2onto/`
subdirectory):

``` json
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

``` bash
python py2onto.py -i "http://example.org/onto.owl#" \
  -n "./classes.csv" \
  -s bioportal -q go \
  -p "./onto_with_xrefs.owl"
```

Behaviour:

-   Only candidates whose `prefLabel` is an **exact case-insensitive
    match** and whose ontology short name matches are accepted; their
    `@id` is written to `hasDbXref`.
-   The ontology short name is the last URL segment of `links.ontology`,
    lowercased.
-   Network access and a valid API Key are required.
-   **Note**: source comments mention `MEDPORTAL_API_KEY` /
    `BIOPORTAL_API_KEY` environment variables, but the current
    implementation **only reads `config.json`**, not the environment.

------------------------------------------------------------------------

## 9. Output and verification

-   Output: a single `.owl` file in RDF/XML, openable in Protégé /
    WebProtégé.
-   Console summary: class count (excluding `Thing`), object properties,
    annotation properties; the individual count is shown only when
    greater than zero.
-   Quick check:

``` python
import owlready2
onto = owlready2.get_ontology("./demo.owl").load()
print(len(list(onto.classes())), len(list(onto.individuals())),
      len(list(onto.object_properties())), len(list(onto.annotation_properties())))
```

------------------------------------------------------------------------

## 10. FAQ

  ----------------------------------------------------------------------------------------
  Problem                                  Cause                   Fix
  ---------------------------------------- ----------------------- -----------------------
  `unresolved parent class`                Misspelled or empty     Ensure the parent ID
                                           `Parent_Class`; top     exactly matches an `ID`
                                           level must be `Thing`.  in the class table.

  `CSV header mismatch`                    Header mismatch or      Regenerate templates
                                           missing `*` in          with `-c`.
                                           `*definition`.          

  `Domain class ... is not defined`        Property references a   Use `-a`/`-o` in the
                                           class that does not     CLI, or create classes
                                           exist yet.              first in the API.

  `individual ... has an empty relation`   `relation` is not       Fill in
                                           `has_individual`.       `has_individual`.

  Non-ASCII text garbled                   File read with the      Save as UTF-8 with BOM;
                                           wrong encoding.         fallback is gbk /
                                                                   gb18030.

  No `.owl` produced                       `-n` missing.           Provide the class CSV
                                                                   via `-n`.

  Template names concatenated after `-c`   Path lacks a trailing   Use
                                           separator.              `-c "./templates/"`.

  API Key from environment ignored         Env vars are not read.  Put the key in
                                                                   `config.json`.
  ----------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 11. Known caveats

1.  **Trailing separator required for `-c`**: file names are
    concatenated as strings; `./templates` produces
    `./templatestemplateAP.csv`.
2.  **No `-n`, no output**: `-a`/`-o`/`-b` alone never produce an OWL
    file.
3.  **`-b` and `-s` are processed only when `-n` is present**.
4.  **RDF/XML only**; no SHACL validation or alternative serialisations
    in this script.
5.  **Class and individual labels are stored with `lang="en"`** even
    when the label is non-English.
6.  **The dotted suffix in `$prop.Class` is only a hint**; the base
    property name is used and targets come from cell values.
7.  **Any `-s` value other than `bioportal` routes to MedPortal**.
8.  **Parent resolution retries 50/100 passes** (100 for \>100 rows)
    before failing on orphan classes.

------------------------------------------------------------------------

## Citation

If you use Py2ONTO, please cite our paper:

**Chinese reference:**

``` text
王哲, 杨晟, 朱彦, 等. 本体构建工具 Py2ONTO 的设计与实现[J]. 中华医学图书情报杂志, 2023, 31(9): 42–50.
```

**English reference:**

``` text
Wang Z, Yang S, Zhu Y, et al. Py2ONTO: a python-based tool for ontology development[J]. Chinese Journal of Medical Library and Information Science. 2023;31(9):42–50. (in Chinese).
```
