# Agent Evaluation Guide

This documentation provides instructions for AI agents to evaluate whether a given Python repository contains instrument driver or data processing module code that is compatible with **IvoryOS**.

IvoryOS auto-introspects Python modules and populates them into a drag-and-drop user interface. Because IvoryOS is a web application, it has specific constraints on the input arguments and return types it supports.

When evaluating a repository, follow this three-step process:

## Step 1: Repository Value Assessment

Determine if the repository is a valid candidate for IvoryOS.
- **Criteria**: Is the repository an instrument controller (e.g., hardware driver) or a data processing module?
- **Format**: Is it written using Python classes?
- **Action**: If it meets these criteria, proceed to Step 2. If it is irrelevant (e.g., a generic web framework, unstructured scripts without classes, or non-Python code), mark it as incompatible and halt the evaluation.

## Step 2: Compatibility Evaluation

Evaluate whether the Python classes and their public methods are compatible with IvoryOS input constraints. Compare the module's method signatures against the supported types and UI controls detailed below. 

### Input types and UI controls

IvoryOS builds UI controls from Python method signatures. While good type hints make direct control, workflow design, CSV configuration, generated proxy clients, and optimizer setup more predictable, **type hints are not strictly required**. Repositories lacking type hints can still be compatible and should not be marked as incompatible for this reason alone.

#### Basic pattern

Expose a normal Python object or function, then launch IvoryOS from the script where it is initialized.

```python
class Pump:
    def dispense(self, volume_ml: float, rate_ml_min: float = 1.0) -> float:
        return self.driver.dispense(volume_ml=volume_ml, rate_ml_min=rate_ml_min)
```

IvoryOS sees `pump.dispense(...)`, builds form fields for `volume_ml` and `rate_ml_min`, and converts submitted UI values using the annotations.

#### Primitive types

| Annotation | UI input | Converted value |
| --- | --- | --- |
| `int` | `42` | `42` |
| `float` | `3.14` | `3.14` |
| `str` | `hello` | `"hello"` |
| `bool` | `True` / `False` | `True` / `False` |

Use primitive annotations for most scalar hardware settings such as volumes, temperatures, durations, speeds, and names.

#### Container types

| Annotation | UI input | Converted value |
| --- | --- | --- |
| `list` | `[1, 2, 3]` | `[1, 2, 3]` |
| `tuple` | `(1, "a")` | `(1, "a")` |
| `set` | `{1, 2, 3}` | `{1, 2, 3}` |

Container inputs use Python literal syntax. Keep them for compact configuration values; if users need to edit many rows, prefer a CSV parameter table.

#### Optional and Union types

`Optional[...]` allows blank or `None` values. `Union[...]` lets IvoryOS try more than one supported conversion.

#### Enum dropdowns

Use Python `Enum` classes when a parameter should be selected from a fixed list. IvoryOS turns Enum annotations into dropdown-style choices in the UI. 

*Note: The UI shows the Enum member names as choices. Convert it back inside the method if the rest of your code expects an Enum instance.*

#### Configurable workflow parameters

In the workflow designer, prefix an argument with `#` to make it configurable at run time (e.g., `#wash_solvent`). Configurable parameters appear during run preparation.

#### Return values

- **Scalar numerical outputs** can be used for optimizer-based runs.
- **Tuple outputs** (e.g., `tuple[float, float]`) are used when one method returns multiple values that users may want to save separately.
- **Dict outputs** are used when the result is best handled as a structured object.

#### Practical rules

- While highly recommended, type hints are **not a strict requirement**. If a module lacks type hints, do not mark it as incompatible; IvoryOS can often handle untyped inputs.
- If present, use type hints for every public method you want users to call from IvoryOS.
- Use `Enum` for settings with a fixed set of valid choices.
- Keep hardware-only driver objects inside your wrapper class; expose small, stable methods to IvoryOS.

**Action**: If the public methods use unsupported types that cannot be serialized/deserialized easily by IvoryOS, evaluate the effort required to fix it:
- If the module requires a lot of refactoring, **do not spend time refactoring**. Simply add a mark saying `incompatible` in your evaluation report.
- If it is just a quick, easy fix, perform a quick evaluation and add a mark saying `fixable - easy`.
Document any incompatibilities clearly in your evaluation report.

## Step 3: Data Extraction and Schema

If you find compatible drivers or modules, extract their metadata and write everything into a structured CSV format. 

### Output CSV Schema Requirements

Generate a CSV table with the following columns (derived from the `public.modules` SQL schema):

| Column Name | Type | Description |
|---|---|---|
| `name` | text | Display name of the module |
| `icon_emoji` | string | An emoji representing the module |
| `pip_name` | text | The package name on PyPI (if applicable) |
| `module_name` | text | The importable Python module name |
| `module_path` | text | Path to the module within the repository |
| `difficulty` | text | E.g., 'beginner', 'intermediate', 'advanced' |
| `download_count` | integer | (Leave empty or 0 if unknown) |
| `contributor_id` | uuid | (Leave empty if unknown) |
| `status` | string | Current status (e.g., 'active', 'testing') |
| `os` | text[] | Supported operating systems (e.g., `linux,macos,windows`) |
| `device_id` | integer | (Leave empty if unknown) |
| `python_versions` | text[] | Supported Python versions (e.g., `3.9,3.10`) |
| `is_tested_with_ivoryos`| boolean | `true` if tested with IvoryOS, otherwise `false` |
| `is_unlisted` | boolean | Typically `false` for public registry |
| `init_args` | jsonb | Default initialization arguments (e.g., `[]`) |
| `is_original_developer` | boolean | `true` or `false` |
| `connection` | text[] | Connection types (e.g., `USB,Ethernet`) |
| `start_command` | text | Command to start the module |
| `description` | text | Brief description of the module |
| `python_command` | text | The Python command to run it |

**Final Output**: Present your evaluation report detailing your findings for Step 1 and Step 2. If the repository is compatible, conclude with the Step 3 CSV block containing the extracted schema data.

### Example CSV Output

```csv
name,icon_emoji,pip_name,module_name,module_path,difficulty,download_count,contributor_id,status,os,device_id,python_versions,is_tested_with_ivoryos,is_unlisted,init_args,is_original_developer,connection,start_command,description,python_command
"Ika Thermoshaker",,ika,Thermoshaker,ika.thermoshaker,,9,009e37d3-608d-403f-bc4b-4fbee4ccd4fc,,Windows,6,,true,false,"[{""name"": ""port"", ""type"": ""str""}, {""name"": ""dummy"", ""type"": ""bool""}]",false,"usb,serial",,,
```
