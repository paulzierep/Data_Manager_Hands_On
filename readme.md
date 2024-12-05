# Introduction

## Abstraction Logic

<img src="https://docs.galaxyproject.org/en/latest/_images/data_managers_schematic_overview.png" width="800" alt="Data Managers Schematic Overview">

## Folder Structure of a Data Manager (DM)

```plaintext
├── data_manager [like a tool]
│   ├── data_manager_fetch_mapseq_db.py
│   ├── macros.xml
│   ├── mapseq_db_fetcher.xml
│   ├── readme.md
│   └── .shed.yml
├── data_manager_conf.xml [DM specific configuration]
├── test-data
│   ├── mgnify_v5_lsu
│   │   ├── lsu2_trimmed.otu
│   │   ├── LSU.fasta_trimmed.mscluster
│   │   ├── LSU_trimmed.fasta
│   │   └── slv_lsu_filtered2_trimmed.txt
│   └── mgnify_v6_rp2
│       ├── PR2.fasta_trimmed.mscluster
│       ├── PR2-tax_trimmed.txt
│       ├── PR2_trimmed.fasta
│       ├── PR2_trimmed.otu
│       └── VERSION.txt
├── tool-data
│   └── mapseq_db.loc.sample
└── tool_data_table_conf.xml.sample [DM specific configuration]
```

---

## Step-by-Step Guide

Most Data Managers (DMs) follow a standard logic. You can copy an existing DM (e.g., mapseq DM) and adapt the following files to your requirements:

### 1. `data_manager_conf.xml`

This file tells Galaxy:

- **Purpose**:
  - Specifies columns written to the data table.
  - Defines where and how to store fetched data.
  - Links the DM "tool" to the data table.
- **Key Parameters to Adapt**:
  - `columns`
  - `data_table` name
  - `id`
  - `data_manager` tool_file
  - `target base`
  - `value_translation`

**Example:**

```xml
<?xml version="1.0"?>
<data_managers>
    <data_manager tool_file="data_manager/mapseq_db_fetcher.xml" id="mapseq_db_fetcher">
        <data_table name="mapseq_db">
            <output>
                <column name="value" />
                <column name="name" />
                <column name="version" />
                <column name="path" output_ref="out_file">
                    <move type="directory" relativize_symlinks="True">
                        <source>${path}</source>
                        <target base="${GALAXY_DATA_MANAGER_DATA_PATH}">mapseq/${value}</target>
                    </move>
                    <value_translation>${GALAXY_DATA_MANAGER_DATA_PATH}/mapseq/${value}</value_translation>
                    <value_translation type="function">abspath</value_translation>
                </column>
            </output>
        </data_table>
    </data_manager>
</data_managers>
```

---

### 2. `tool_data_table_conf.xml.sample`

This file specifies the structure and storage location of the data table.

- **Purpose**:
  - Defines the format of the data table.
- **Key Parameters to Adapt**:
  - `name`
  - `columns`
  - `file path`

**Example:**

```xml
<tables>
    <table name="mapseq_db" comment_char="#">
        <columns>value, name, version, path</columns>
        <file path="tool-data/mapseq_db.loc" />
    </table>
</tables>
```

---

### 3. `<tool_DB>.loc.sample`

- **Purpose**:
  - Provides a sample data table for admins.
  - Sample files must use tabs (`\t`) as separators.

**Example (DM sample):**

```plaintext
# Example format for mapseq DB:
<value>    <name>    <version>    <path>
```

**Example (Tool test folder):**

```plaintext
test_mapseq_db    MGnify LSU Reference Database 0.1    0.1    ${__HERE__}/mapseq_db
```

---

### 4. `data_manager/tool_db_fetcher.xml`

🚨 This file is similar to a tool wrapper but must output `data_manager_json`. 

- **Purpose**:
  - Specifies what to write in the data table.
  - Defines how data is fetched or processed.
- **Key Parameters to Adapt**:
  - Add database options (`<param>` tags).
  - Forward arguments to the DM Python script.

**Example JSON output:**

```json
{
    "data_tables": {
        "<table name>": {
            "value": <db_value>,
            "name": <name>,
            "version": <version>,
            "path": <db_path>
        }
    }
}
```

**Example XML output:**

```xml
<outputs>
    <data format="data_manager_json" name="out_file" />
</outputs>
```

**Adding Options:**

```xml
<param name="database_type" type="select" multiple="false" label="Database Type">
    <option value="mgnify_v6_lsu">MGnify LSU (v6.0)</option>
    <option value="mgnify_v6_ssu">MGnify SSU (v6.0)</option>
    ...
</param>
```

---

### 5. `data_manager/data_manager_tool.py`

This Python script fetches, processes, and organizes the data. It must output a JSON file (`data_manager_json`) that is passed to the wrapper.

#### Where to store the data ?

You first need a temporary location to store the file and then the DM will move it 
to `<target base="${GALAXY_DATA_MANAGER_DATA_PATH}">mapseq/${value}</target>` of the path.

Usually the temp. location is the `extra_files_path` of the tools
param, that can be accessed via the '${out_file}' of the DM wrapper.

```python
with open(args.output) as fh:
    params = json.load(fh)

workdir = params["output_data"][0]["extra_files_path"]
```

#### How to write the output json ?

```python
data_manager_entry = {
    "data_tables": {
        "mapseq_db": {
            "value": db_value,
            "name": f"{db_name} downloaded at {time}",
            "version": args.version,
            "path": db_path,
        }
    }
}

with open(os.path.join(args.output), "w+") as fh:
    json.dump(data_manager_entry, fh, sort_keys=True)
```

#### How to test a DM ?

The wrapper XML can only test weather the json is correct. This does not check the actual DB. 
In many cases the real DB cannot be tested due to the size, but some logics can be tested in the 
python script, such as checking if the url exists or if a test DB is correctly downloaded.



---

## 

## Run DM locally

### Install Planemo

```bash
mamba create -n planemo python=3.11
mamba activate planemo
pip install -U planemo
```

### Serve Galaxy with a Cached Instance

```bash
planemo serve <tool.xml> --biocontainers --galaxy_root ~/git/galaxy
```

---

## Running Tool and DM Together

**🚨 The order of the tool and DM in the planemo command matters, if the tool is used as first argument, only the .loc file in 
the test-data of the tool is used and even the DM will write in that .loc file !**

**General Command:**

```bash
planemo serve <DM_PATH>/data_manager/DM.xml <TOOL_PATH>/tool.xml --biocontainers --galaxy_root ~/git/galaxy
```

**Mapseq Example:**

```bash
cd ~/git/galaxyproject/tools-iuc
planemo serve data_managers/data_manager_mapseq/data_manager/mapseq_db_fetcher.xml tools/mapseq/mapseq.xml --biocontainers --galaxy_root ~/git/galaxy
```

---

## Verifying DM Integration in Galaxy

1. Navigate to **Admin** > **Local Data**.
2. Verify that the DM appears under **Installed Data Managers**.
3. Run the DM tool to populate data tables.
4. Go to **Admin** > **Data Tables** to verify new table entries.
5. Use the tool to confirm the new data is accessible.

