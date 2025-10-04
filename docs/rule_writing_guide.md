# Rule Writing Guide

This guide explains how to create and structure the YAML rule files used for processing and classifying medical imaging data.

## 1. File Structure Overview

Each rule file is a YAML document with the following main sections:

- **Metadata:** Basic information about the rule file.
- **`column_mapping`:** Defines how to map columns from the input data.
- **`categories`:** Defines the classification hierarchy.
- **`classification_rules`:** Contains the logic for assigning data to categories.
- **`default_classification`:** A fallback category for data that doesn't match any rule.
- **`export_as_sheet`:** Specifies how to format the output data.

---

## 2. Metadata

This section contains general information about the rule file.

- `title`: A human-readable title for the rule set.
- `version`: The version of the rule file.
- `created_at` / `updated_at`: Timestamps for creation and last modification.
- `description`: A brief explanation of what the rule set does.
- `author`: Information about the author.
  - `name`: The author's name.
  - `email`: The author's email.

**Example:**
```yaml
title: "Example Hospital Rules"
version: "1.0"
created_at: "2023-01-01T00:00:00Z"
updated_at: "2023-01-01T00:00:00Z"
description: "Rules for processing imaging data from Example Hospital."
author:
  name: "John Doe"
  email: "john.doe@example.com"
```

---

## 3. Column Mapping (`column_mapping`)

This section maps the columns from the source data (e.g., a CSV file) to standardized fields used by the processing engine.

- **`required`**: A list of columns that must be present in the source data.
- **`optional`**: A list of columns that are not mandatory.
- **`default_order`**: The default column order for display or processing.

Each column is defined with:
- `name`: A human-readable name.
- `field`: The internal field name to map to.
- `type`: The data type (`string`, `number`, `date`).
- `aliases`: A list of possible column names from the source data. This allows for flexibility if the source data column names vary.
- `description`: An optional explanation of the column.

**Example:**
```yaml
column_mapping:
  required:
    - name: "고유번호"
      field: "id"
      type: "string"
      aliases: ["STUDY KEY", "STUDY_KEY"]
    - name: "검사일"
      field: "study_date"
      type: "date"
      aliases: ["STUDY DATE", "Exam Date(Time)"]
  optional:
    - name: "나이"
      field: "age"
      type: "number"
      aliases: ["AGE", "PATIENT_AGE"]
  default_order:
    - "study_date"
    - "id"
    - "age"
```

---

## 4. Categories (`categories`)

This section defines the hierarchical structure for classifying the data.

- `name`: The display name of the category.
- `id`: A unique identifier for the category.
- `target`: (Optional) A numerical target or goal for this category.
- `subcategories`: (Optional) A nested list of sub-categories, which follow the same structure.

**Example:**
```yaml
categories:
  - name: "CT"
    id: "ct"
    target: 3500
    subcategories:
      - name: "신경두경부"
        id: "ct_neu"
        target: 700
      - name: "흉부"
        id: "ct_cht"
        target: 800
```

---

## 5. Classification Rules (`classification_rules`)

This is the core of the rule file, where you define the logic to assign each data record to a category. It's an ordered list of rules, where the first matching rule is applied.

Each rule has:
- `category_id`: The `id` of the category to assign if the conditions are met.
- `parent_category_id`: (Optional) Specifies that this rule is for a sub-category. The record must already be classified into the parent category for this rule to be evaluated.
- `priority`: (Optional) A number to control the execution order of rules at the same level. Lower numbers have higher priority. This is useful for resolving ambiguities where a record could match multiple rules.
- `composed_by_subcategories`: (Optional) If `true`, this category is just a container for its sub-categories. It won't be directly assigned.
- `inherit_conditions_from`: (Optional) The `id` of another rule to inherit conditions from. This helps to avoid duplicating logic (DRY principle).
- `conditions`: A list of conditions that must be met.

### Conditions

Conditions can be simple or nested using logical operators.

- `operator`: The comparison operator to use.
- `field`: The internal field name to check (from `column_mapping`).
- `value`: The value to compare against.
- `case_sensitive`: (Optional) `true` or `false` for string comparisons.

**Available Operators:**
- `equals`, `not_equals`
- `contains`, `not_contains`
- `is_null`, `not_null`
- `lessThan`, `greaterThan`

### Logical Grouping

You can group conditions using `logic: AND` or `logic: OR`.

**Example Rules:**

**1. Simple Rule:** Assign to `ct` category if `modality` is "CT".
```yaml
- category_id: ct
  conditions:
    - operator: equals
      field: modality
      value: "CT"
```

**2. Rule with `AND` logic:**
```yaml
- category_id: ct_cardiac
  parent_category_id: ct
  conditions:
    - logic: AND
      conditions:
        - operator: contains
          field: exam_name
          value: "heart"
        - operator: not_contains
          field: exam_name
          value: "angiography"
```

**3. Rule with `OR` logic and `priority`:**
```yaml
- category_id: xray_kub
  parent_category_id: xray
  priority: 2
  conditions:
    - operator: contains
      field: exam_name
      value: "KUB"
      case_sensitive: true
```

**4. Inheriting Conditions:**
```yaml
- category_id: xray_ped_msk
  parent_category_id: xray_ped
  inherit_conditions_from: xray_msk # Inherits conditions from the 'xray_msk' rule
```

---

## 6. Default Classification (`default_classification`)

If a data record does not match any of the `classification_rules`, it will be assigned to the category specified here.

**Example:**
```yaml
default_classification:
  category_id: "unclassified"
```

---

## 7. Export Configuration (`export_as_sheet`)

This section defines how to generate the final output file (e.g., CSV or Excel).

- `default_format`: The default file format (e.g., "csv").
- `encoding`: The file encoding (e.g., "utf-8").
- `filename`:
  - `pattern`: A pattern for the output filename. You can use `{date}` and `{time}` placeholders.
  - `date_format`, `time_format`: Formats for the date and time placeholders.
- `templates`: A list of output templates.

### Templates

Each template defines a specific output format.

- `name`: A unique name for the template.
- `description`: A brief description of the template's purpose.
- `column_order`: The order of columns in the output file.
- `columns`: A list defining each column in the output.
- `template_column_mapping`: (Optional) Maps internal category names to specific values for the export.
- `ajax_template`: (Optional) A JavaScript snippet for making AJAX calls with the processed data.

### Column Definitions

Each column in the `columns` list can derive its value from various sources:

- **`from_field`**: Pulls the value directly from a data field.
- **`from_metadata`**: Pulls the value from the classification metadata (e.g., category name).
- **`static`**: A fixed, static value.
- **`auto_increment`**: A sequentially increasing number.
- **`conditional`**: A value based on a set of conditions.
- **`fallback_chain`**: Tries a list of sources in order and uses the first one that returns a value.

**Example: Export Configuration**
```yaml
export_as_sheet:
  filename:
    pattern: "logbook_export_{date}_{time}"
    date_format: "YYYY-MM-DD"
  templates:
    - name: "for_record2"
      description: "record2 사이트 업로드용"
      column_order: ["serial_number", "modality", "specialty"]
      columns:
        - serial_number:
            name: "일련번호"
            value:
              auto_increment:
                start_value: 1
        - modality:
            name: "검사종류"
            value:
              from_metadata:
                key: examination_type_name
            is_classifier: true
        - specialty:
            name: "세부전공"
            value:
              from_metadata:
                key: "specialty_name"
            is_classifier: true
      template_column_mapping:
        modality:
          "일반촬영":
            value: 1
            speciality:
              "흉부": 2
              "복부": 3
```