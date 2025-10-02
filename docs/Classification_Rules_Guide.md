# 분류 규칙 작성 가이드

이 문서는 `rules.yaml` 파일 내의 새로운 **계층적 분류 시스템**을 작성하고 이해하는 데 필요한 지침을 제공합니다. 새로운 시스템은 모든 로그북 데이터를 **검사종류**, **세부전공**, **비고(옵션)**의 계층으로 분류합니다.

## 1. 새로운 계층적 분류 시스템 개요

- **검사종류 (Examination Type)**: 일반촬영, CT, MRI, 초음파, 시술, 핵의학 등
- **세부전공 (Specialty)**: 각 검사종류별로 정의된 전문 분야
- **비고 (Remarks, 옵션)**: 세부전공 하위의 추가 구분 레벨

### 비고 레벨을 가지는 세부전공

- 일부 세부전공은 하위에 **비고** 레벨을 가질 수 있음
- 비고 레벨을 가지는 세부전공의 경우, 추가적인 세분화 분류가 가능
- 예: `일반촬영-소아-신경두경부`, `CT-소아-복부` 등 (소아 세부전공의 비고 레벨)

이를 통해 `일반촬영-복부`, `일반촬영-소아-복부` 등의 세분화된 분류가 가능합니다.

## 2. 계층적 분류 구조 정의

### 2.1. 검사종류별 세부전공 매핑 (`examination_types`)

각 검사종류마다 해당하는 세부전공이 다르며, 일부 세부전공은 비고 레벨을 가질 수 있습니다:

```yaml
examination_types:
  xray:
    name: '일반촬영'
    id: 'xray'
    target: 10000
    specialties:
      neu: '신경두경부'
      cht: '흉부'
      abd: '복부'
      kub: 'KUB'
      spine: '척추'
      msk: '관절및사지'
      ped: # 비고 레벨을 가지는 세부전공 예시
        name: '소아'
        target: 400
        remarks: # 세부전공 하위의 비고 레벨
          neu: '신경두경부'
          cht: '흉부'
          abd: '복부'
          spine: '척추'
          msk: '관절및사지'
          etc: '기타'

  ct:
    name: 'CT'
    id: 'ct'
    target: 3500
    specialties:
      neu: '신경두경부'
      cht: '흉부'
      cardiac: '심장'
      vas: '혈관'
      abd: '복부'
      gu: '비뇨생식기'
      spine: '척추'
      msk: '관절및사지'
      ped: # 비고 레벨을 가지는 세부전공 예시
        name: '소아'
        target: 100
        remarks:
          neu: '신경두경부'
          cht: '흉부'
          abd: '복부'
          gu: '비뇨생식기'
          spine: '척추'
          msk: '관절및사지'
          etc: '기타'

  mri:
    name: 'MRI'
    id: 'mri'
    target: 1200
    specialties:
      neu: '신경두경부'
      cardiac: '심장'
      abd: '복부'
      gu: '비뇨생식기'
      spine: '척추'
      msk: '관절및사지'
      breast: '유방'
      ped: # 비고 레벨을 가지는 세부전공 예시
        name: '소아'
        target: 30
        remarks:
          neu: '신경두경부'
          abd: '복부'
          spine: '척추'
          msk: '관절및사지'
          etc: '기타'
```

### 2.2. 최종 분류 결과 구조

새로운 계층적 분류 시스템에서 최종 분류 결과는 다음과 같은 형태가 됩니다:

#### 일반 세부전공 (비고 없음)

```
{examination_type}_{specialty}
```

#### 비고 레벨이 있는 세부전공

```
{examination_type}_{specialty}_{remarks}
```

**예시:**

- `xray_abd`: 일반촬영-복부
- `xray_ped`: 일반촬영-소아 (비고 없음)
- `xray_ped_abd`: 일반촬영-소아-복부 (비고 있음)
- `ct_neu`: CT-신경두경부
- `ct_ped_neu`: CT-소아-신경두경부
- `mri_spine`: MRI-척추
- `mri_ped_spine`: MRI-소아-척추

## 3. 분류 규칙 구조 (`classification_rules`)

새로운 계층적 분류 시스템에서는 다음과 같이 분류가 이루어집니다:

### 3.1. 1단계: 검사종류 분류

각 검사종류 분류 규칙은 다음 속성을 가집니다:

- `category_id`: (`string`, 필수) 검사종류 식별자 (예: `xray`, `ct`, `mri`)
- `name`: (`string`, 필수) 분류 규칙의 사람이 읽을 수 있는 이름
- `priority`: (`number`, 필수) 규칙의 우선순위 (낮은 숫자가 높은 우선순위)
- `conditions`: (`array`, 필수) 조건 목록
- `logic`: (`string`, 선택 사항, 기본값: `AND`) 조건 평가 방법 (`AND` 또는 `OR`)
- `composed_by_subcategories`: (`boolean`, 선택, 기본값: `false`) 하위 카테고리로만 구성되는지 여부

#### `composed_by_subcategories` 옵션 설명

이 옵션은 카테고리가 하위 카테고리로만 구성되는지를 제어합니다:

- **`true`**: 해당 카테고리는 하위 카테고리로만 구성됨
  - **조건(`conditions`) 무시**: `conditions` 배열이 있어도 무시됨
  - **동적 구성**: 하위 카테고리들의 조건을 통해 동적으로 구성
  - 직접적인 데이터 할당 없음
  - 모든 데이터는 하위 카테고리로 분류되어야 함

- **`false` 또는 생략 (기본값)**: 해당 카테고리에 직접 데이터 할당 가능
  - **조건 평가**: `conditions` 배열의 조건들을 평가하여 분류
  - 하위 카테고리와 함께 직접적인 데이터도 포함

### 3.2. 2단계: 세부전공 분류

검사종류가 결정된 후, 해당 검사종류의 세부전공으로 분류:

- `category_id`: (`string`, 필수) 세부전공 식별자 (예: `xray_abd`, `xray_ped`)
- `parent_category_id`: (`string`, 필수) 상위 검사종류 ID
- `name`: (`string`, 필수) 세부전공 이름
- `priority`: (`number`, 필수) 세부전공 내 우선순위
- `conditions`: (`array`, 필수) 세부전공 분류 조건
- `logic`: (`string`, 선택 사항, 기본값: `AND`) 조건 평가 방법 (`AND` 또는 `OR`)

### 3.3. 3단계: 비고 분류 (비고 레벨이 있는 세부전공의 경우)

비고 레벨을 가지는 세부전공이 결정된 후, 추가로 비고 레벨 분류:

- `category_id`: (`string`, 필수) 비고 식별자 (예: `xray_ped_abd`)
- `parent_category_id`: (`string`, 필수) 상위 세부전공 ID
- `name`: (`string`, 필수) 비고 이름
- `priority`: (`number`, 필수) 비고 내 우선순위
- `conditions`: (`array`, 필수) 비고 분류 조건
- `logic`: (`string`, 선택 사항, 기본값: `AND`) 조건 평가 방법 (`AND` 또는 `OR`)

## 4. `conditions` 상세 설명

각 `condition` 객체는 다음 속성을 가집니다.

- `operator`: (`string`, 필수) 조건을 평가하는 데 사용될 논리 연산자입니다. 지원되는 연산자는 다음과 같습니다.
  - `contains`: `field`의 값이 `value`를 포함하는지 확인합니다.
  - `not_contains`: `field`의 값이 `value`를 포함하지 않는지 확인합니다.
  - `equals`: `field`의 값이 `value`와 정확히 일치하는지 확인합니다.
  - `not_equals`: `field`의 값이 `value`와 정확히 일치하지 않는지 확인합니다.
  - `startsWith`: `field`의 값이 `value`로 시작하는지 확인합니다.
  - `endsWith`: `field`의 값이 `value`로 끝나는지 확인합니다.
  - `greaterThan`: `field`의 값이 `value`보다 큰지 확인합니다.
  - `lessThan`: `field`의 값이 `value`보다 작은지 확인합니다.
  - `between`: `field`의 값이 `value` (배열 `[min, max]` 형식) 범위 내에 있는지 확인합니다.
  - `regex`: `field`의 값이 `value` (정규 표현식)와 일치하는지 확인합니다.
- `field`: (`string`, 필수) 검사할 데이터 필드의 이름입니다. `column_mapping` 섹션에 정의된 `field` 중 하나여야 합니다.
  - **`age` 필드 특별 처리:** `age` 필드는 "00D 00M 00Y" 형식의 문자열 (예: "1Y 3M", "13M", "365D") 또는 순수 숫자(예: 17)를 모두 처리하며, 내부적으로는 `연도(years)` 단위의 정수로 변환되어 비교됩니다. 이 과정에서 `days`와 `months`는 `years`로 내림 처리됩니다 (예: 13M은 1년으로 처리). 따라서 `age` 필드에 조건을 적용할 때는 연도 단위의 숫자를 사용하십시오.
- `value`: (`string`, `number`, `[number, number]`, 필수) `operator`와 비교할 값입니다. `between` 연산자의 경우 `[min, max]` 형식의 숫자 배열입니다.
- `case_sensitive`: (`boolean`, 선택 사항, 기본값: `false`) 비교 시 대소문자를 구분할지 여부를 지정합니다.
  - `false` (기본값): 대소문자를 구분하지 않음 (예: "CT"와 "ct"가 같음)
  - `true`: 대소문자를 구분함 (예: "CT"와 "ct"가 다름)
  - 생략 시 자동으로 `false`로 처리됩니다.

  **사용 예시:**

  ```yaml
  # 기본값 사용 (대소문자 구분 안함)
  - operator: contains
    field: modality
    value: 'CT'
    # case_sensitive 생략 시 false로 처리

  # 명시적으로 대소문자 구분 안함
  - operator: contains
    field: modality
    value: 'CT'
    case_sensitive: false

  # 대소문자 구분함
  - operator: equals
    field: department
    value: 'ER'
    case_sensitive: true # "ER"과 "er"을 다르게 처리
  ```

- `regex_flags`: (`string`, 선택 사항) `regex` 연산자에 적용될 플래그 (예: `i` for case-insensitive). `regex` 연산자와 함께 사용됩니다.
- `logic`: (`string`, 선택 사항, 기본값: `AND`) 여러 하위 조건이 평가되는 방법을 정의합니다. `AND` 또는 `OR`이 될 수 있습니다. (이것은 중첩된 조건에 사용됩니다.)
- `conditions`: (`array`, 선택 사항) 중첩된 조건을 포함하는 목록입니다. `logic` 필드와 함께 사용됩니다.

## 5. 계층적 분류 로직

새로운 계층적 분류 시스템은 다음과 같은 순서로 동작합니다:

### 5.1. 분류 프로세스

1. **검사종류 결정**:
   - `modality` 필드를 기반으로 검사종류를 먼저 결정
   - `priority` 순서대로 규칙을 평가하여 일치하는 첫 번째 검사종류 선택

2. **세부전공 결정**:
   - 결정된 검사종류의 세부전공 목록에서 분류
   - `assign`, `exam_name`, `department`, `age` 등의 필드를 조합하여 판단
   - 검사종류별로 정의된 세부전공 규칙을 `priority` 순서로 평가

3. **비고 분류 (비고 레벨이 있는 세부전공의 경우)**:
   - 비고 레벨을 가지는 세부전공이 선택된 경우, 추가로 비고 레벨 분류 수행
   - 비고 분류는 해당 세부전공에 정의된 비고 분류 기준 사용
   - 비고가 없으면 세부전공 레벨만으로 분류 완료

### 5.2. 분류 우선순위

### 각 레벨의 priority 순서

분류 시스템의 각 계층(검사종류, 세부전공, 비고)에서는 `priority` 값이 낮을수록 높은 우선순위를 가집니다. 즉, 같은 레벨 내에서 `priority: 1`이 가장 먼저 평가되고, 숫자가 커질수록 나중에 평가됩니다.

- **검사종류(1레벨)**:  
  `classification_rules`에서 검사종류별로 `priority` 값을 지정합니다.  
  예시:

  ```yaml
  - category_id: ct
    name: 'CT'
    priority: 2
    conditions: ...
  - category_id: intervention
    name: '시술'
    priority: 1 # 시술이 CT보다 먼저 평가됨
    conditions: ...
  ```

- **세부전공(2레벨)**:  
  각 검사종류 내에서 세부전공별로 `priority`를 지정합니다.  
  예시:

  ```yaml
  - category_id: ct_ped
    parent_category_id: ct
    name: 'CT-소아'
    priority: 1 # 소아가 최우선
    conditions: ...
  - category_id: ct_neu
    parent_category_id: ct
    name: 'CT-신경두경부'
    priority: 2
    conditions: ...
  ```

- **비고(3레벨)**:  
  비고 레벨이 있는 세부전공 내에서 비고별로 `priority`를 지정합니다.  
  예시:
  ```yaml
  - category_id: ct_ped_neu
    parent_category_id: ct_ped
    name: 'CT-소아-신경두경부'
    priority: 1
    conditions: ...
  - category_id: ct_ped_etc
    parent_category_id: ct_ped
    name: 'CT-소아-기타'
    priority: 99 # 기타는 항상 마지막에 평가
    conditions: ...
  ```

#### 우선순위 지정 시 주의사항

- 같은 레벨에서 `priority` 값이 중복되면 예측 불가능한 분류가 발생할 수 있으므로 반드시 고유하게 지정해야 합니다.
- 소아 등 특수 분류는 항상 `priority: 1`로 설정하는 것을 권장합니다.
- `priority`를 지정하지 않으면 기본값(10)이 적용됩니다.
- `priority`가 낮은 규칙이 먼저 평가되어 해당 규칙에 일치하면 이후 규칙은 평가되지 않습니다.

#### `_etc` 카테고리의 동적 생성

시스템은 서브카테고리가 있는 상위 카테고리에서 모든 항목이 반드시 서브카테고리에 할당되도록 보장합니다. 이를 위해 `_etc` (기타) 카테고리가 중요한 역할을 합니다:

**동적 생성 규칙:**

- `categories`에 `_etc` 카테고리가 정의되어 있지만 `classification_rules`에 해당 규칙이 없는 경우
- 시스템이 자동으로 다음과 같은 기본 규칙을 동적으로 생성합니다:

```yaml
# 동적으로 생성되는 _etc 규칙 예시
- category_id: xray_etc
  parent_category_id: xray
  name: '기타'
  priority: 999 # 가장 낮은 우선순위
  logic: AND
  conditions: [] # 조건 없음 = 모든 데이터 받아들임
```

**동적 생성의 특징:**

- **조건 없음**: `conditions: []`로 설정되어 다른 모든 조건에 맞지 않는 항목을 받아들임
- **최저 우선순위**: `priority: 999`로 설정되어 다른 모든 규칙이 먼저 평가됨
- **자동 캐싱**: 생성된 규칙이 즉시 캐시에 추가되어 사용 가능
- **로그 출력**: `동적으로 생성된 _etc 규칙을 사용합니다: [카테고리ID]` 메시지 출력

**권장사항:**
명시적인 관리를 위해 `_etc` 규칙을 `classification_rules`에 직접 정의하는 것을 권장합니다:

```yaml
classification_rules:
  # 다른 규칙들...

  - category_id: xray_etc
    parent_category_id: xray
    name: '기타'
    priority: 99
    conditions: [] # 또는 특정 조건 정의

  - category_id: xray_ped_etc
    parent_category_id: xray_ped
    name: '기타'
    priority: 99
    conditions: []
```

### 5.3. 분류 결과 형태

최종 분류 결과는 계층적 구조로 저장됩니다:

#### 일반 분류

```
examination_type: "ct"
specialty: "neu"
full_category: "ct_neu"
display_name: "CT-신경두경부"
```

#### 비고 레벨이 없는 세부전공

```
examination_type: "ct"
specialty: "ped"
full_category: "ct_ped"
display_name: "CT-소아"
```

#### 비고 레벨이 있는 세부전공

```
examination_type: "ct"
specialty: "ped"
remarks: "neu"
full_category: "ct_ped_neu"
display_name: "CT-소아-신경두경부"
```

## 6. 중첩된 `conditions`

`conditions`는 논리 복잡성을 허용하기 위해 중첩될 수 있습니다. `logic` 속성을 사용하여 중첩된 조건 그룹이 `AND` 또는 `OR`로 연결될지 여부를 지정할 수 있습니다. 예를 들어:

```yaml
# 계층적 분류 규칙 예시

# 1단계: 검사종류 분류
- category_id: ct
  name: 'CT'
  priority: 2
  conditions:
    - operator: contains
      field: modality
      value: 'CT'
      case_sensitive: false
    - operator: not_contains
      field: modality
      value: 'PT'
      case_sensitive: false

# 1단계: 검사종류 분류 (composed_by_subcategories 예시)
- category_id: ct
  name: 'CT'
  composed_by_subcategories: true # 하위 세부전공으로만 구성
  # conditions는 무시됨

# 2단계: 세부전공 분류 (CT의 경우)
- category_id: ct_ped
  name: '소아'
  priority: 1 # 우선순위 설정 예시
  conditions:
    - operator: lessThan
      field: age
      value: 17

- category_id: ct_neu
  name: '신경두경부'
  priority: 2
  conditions:
    - logic: OR # 명시적 OR 사용 예시
      conditions:
        - operator: contains
          field: assign
          value: 'H&N'
        - operator: contains
          field: assign
          value: 'NEU'
    - operator: not_contains
      field: exam_name
      value: 'SPINE'

- category_id: ct_abd
  name: '복부'
  priority: 5
  conditions:
    - operator: contains
      field: assign
      value: 'ABD'

# 3단계: 비고 분류 (비고 레벨이 있는 세부전공의 경우)
- category_id: ct_ped_neu
  name: '신경두경부'
  priority: 1
  conditions:
    - logic: OR # 명시적 OR 사용 예시
      conditions:
        - operator: contains
          field: assign
          value: 'H&N'
        - operator: contains
          field: assign
          value: 'NEU'

- category_id: ct_ped_abd
  name: '복부'
  priority: 2
  conditions:
    - operator: contains
      field: assign
      value: 'ABD'
```

## 4. 분류 규칙 속성 참조

### 4.1. 공통 속성

모든 분류 규칙에서 사용할 수 있는 속성들:

| 속성                        | 타입    | 필수 | 설명                                              |
| --------------------------- | ------- | ---- | ------------------------------------------------- |
| `category_id`               | string  | ✓    | 대상 카테고리 ID                                  |
| `name`                      | string  | ✓    | 규칙 이름                                         |
| `priority`                  | number  |      | 우선순위 (기본값: 10, 낮을수록 높은 우선순위)     |
| `conditions`                | array   | ✓    | 분류 조건들                                       |
| `logic`                     | string  |      | 조건 결합 논리 (기본값: AND)                      |
| `inherit_conditions_from`   | string  |      | 다른 규칙의 조건을 상속받을 규칙 ID               |
| `composed_by_subcategories` | boolean |      | 하위 카테고리로만 구성되는지 여부 (기본값: false) |

### 4.2. 주요 특징

- **계층적 분류**: 검사종류 → 세부전공 → 비고 순서로 분류
- **우선순위 기반**: `priority` 값에 따라 규칙 평가 순서 결정
- **조건 기반**: `conditions` 배열의 조건들을 평가하여 분류
- **조건 상속**: `inherit_conditions_from`으로 다른 규칙의 조건 재사용 가능
- **기본값 `AND`**: `logic` 속성의 기본값은 `AND` (생략 가능)
- **하위 구성 옵션**: `composed_by_subcategories`로 카테고리 구성 방식 제어

## 5. 기본 분류 (`default_classification`)

정의된 규칙에 해당하지 않는 데이터의 기본 분류를 설정합니다:

```yaml
default_classification:
  category_id: 'unclassified'
  name: '미분류'
  description: '정의된 분류에 해당하지 않는 업무'
```

## 6. 분류 우선순위

### 6.1. 우선순위 원리

검사종류 분류 시 평가 순서는 **분류 규칙의 `priority` 값**으로 결정됩니다:

- **낮은 숫자 = 높은 우선순위**
- **기본값**: 10
- **소아 관련**: 일반적으로 `priority: 1` 설정 (가장 높은 우선순위)

```yaml
classification_rules:
  - category_id: nuclear_medicine
    name: '핵의학'
    priority: 3 # 높은 우선순위
    conditions: [...]

  - category_id: intervention
    name: '시술'
    priority: 5 # 중간 우선순위
    conditions: [...]

  - category_id: xray_ped
    name: '일반촬영-소아'
    priority: 1 # 최고 우선순위
    conditions: [...]
```

### 6.2. Priority 설정의 중요성

올바른 우선순위 설정은 정확한 분류를 위해 매우 중요합니다:

#### 소아 분류의 우선순위

소아 관련 분류는 **반드시 높은 우선순위**(`priority: 1`)를 가져야 합니다:

```yaml
# ✅ 올바른 예시
- category_id: ct_ped
  name: 'CT-소아'
  priority: 1 # 소아는 최고 우선순위
  conditions:
    - operator: lessThan
      field: age
      value: 18

- category_id: ct_neu
  name: 'CT-신경두경부'
  priority: 2 # 소아보다 낮은 우선순위
  conditions:
    - operator: contains
      field: body_part
      value: 'Brain'
```

#### 잘못된 우선순위의 문제점

```yaml
# ❌ 잘못된 예시 - 우선순위 충돌
- category_id: ct_neu
  name: 'CT-신경두경부'
  priority: 1 # 소아와 같은 우선순위 (문제!)

- category_id: ct_ped
  name: 'CT-소아'
  priority: 1 # 신경두경부와 같은 우선순위 (문제!)
```

**결과**: 17세 환자의 뇌 CT가 "소아"가 아닌 "신경두경부"로 잘못 분류될 수 있음

#### 시술 분류의 우선순위

시술 관련 분류도 적절한 우선순위가 필요합니다:

```yaml
# ✅ 올바른 예시
- category_id: intervention
  name: '시술'
  priority: 2 # 일반적인 높은 우선순위

- category_id: ct
  name: 'CT'
  priority: 5 # 시술보다 낮은 우선순위
```

이를 통해 "CT 유도하 생검"이 "CT"가 아닌 "시술"로 정확히 분류됩니다.

## 7. 조건 상속 기능

### 7.1. 개요

`inherit_conditions_from` 기능을 사용하면 다른 분류 규칙의 조건을 재사용할 수 있습니다. 특히 소아 분류에서 성인 세부전공의 조건을 그대로 활용할 때 매우 유용합니다.

### 7.2. 기본 원리

```yaml
classification_rules:
  # 기본 규칙 (성인 신경두경부)
  - category_id: xray_neu
    name: '신경두경부'
    priority: 10
    conditions:
      - logic: OR
        conditions:
          - operator: contains
            field: assign
            value: 'H&N'
          - operator: contains
            field: assign
            value: 'NEU'
      - operator: not_contains
        field: exam_name
        value: 'spine'

  # 소아 기본 규칙
  - category_id: xray_ped
    name: '소아'
    priority: 1
    conditions:
      - operator: lessThan
        field: age
        value: 17

  # 조건 상속을 사용한 소아 신경두경부
  - category_id: xray_ped_neu
    parent_category_id: xray_ped
    name: '신경두경부'
    inherit_conditions_from: xray_neu # 성인 신경두경부 조건 상속
    # 추가 conditions 없음 - 완전히 상속받은 조건만 사용
```

### 7.3. 상속 처리 과정

1. **파싱 단계**: YAML 파일을 읽을 때 상속 관계를 분석
2. **조건 복사**: 상속받을 규칙의 조건을 깊은 복사(deep copy)
3. **조건 병합**: 기존 조건과 상속받은 조건을 병합
4. **최종 적용**: 병합된 조건으로 분류 수행

위 예시에서 `xray_ped_neu`의 최종 조건:

```yaml
# 자동으로 생성되는 최종 조건 (내부적으로)
conditions:
  # 상속받은 조건들
  - logic: OR
    conditions:
      - operator: contains
        field: assign
        value: 'H&N'
      - operator: contains
        field: assign
        value: 'NEU'
  - operator: not_contains
    field: exam_name
    value: 'spine'
```

### 7.4. 실제 활용 사례

#### 소아 분류에서의 활용

```yaml
# 성인 세부전공 규칙들
- category_id: xray_neu
  name: '신경두경부'
  conditions: [복잡한 신경두경부 조건들]

- category_id: xray_cht
  name: '흉부'
  conditions: [복잡한 흉부 조건들]

- category_id: xray_abd
  name: '복부'
  conditions: [복잡한 복부 조건들]

# 소아 규칙 - 상속 활용
- category_id: xray_ped_neu
  parent_category_id: xray_ped
  name: '신경두경부'
  inherit_conditions_from: xray_neu

- category_id: xray_ped_cht
  parent_category_id: xray_ped
  name: '흉부'
  inherit_conditions_from: xray_cht

- category_id: xray_ped_abd
  parent_category_id: xray_ped
  name: '복부'
  inherit_conditions_from: xray_abd
```

### 7.5. 조건 병합 방식

#### 기존 조건 + 상속 조건

```yaml
- category_id: special_rule
  name: '특수 규칙'
  inherit_conditions_from: base_rule
  conditions:
    - operator: equals
      field: department
      value: 'SPECIAL' # 추가 조건
```

최종 결과: `base_rule의 모든 조건` + `department = "SPECIAL"`

#### 완전 상속 (추가 조건 없음)

```yaml
- category_id: inherited_rule
  name: '상속받은 규칙'
  inherit_conditions_from: base_rule
  # conditions 없음 - base_rule의 조건만 사용
```

### 7.6. 오류 처리 및 검증

#### 순환 참조 방지

```yaml
# ❌ 잘못된 예시 - 순환 참조
- category_id: rule_a
  inherit_conditions_from: rule_b

- category_id: rule_b
  inherit_conditions_from: rule_a # 오류 발생!
```

**오류 메시지**: `"순환 참조가 감지되었습니다: rule_a"`

#### 존재하지 않는 규칙 참조

```yaml
# ❌ 잘못된 예시
- category_id: test_rule
  inherit_conditions_from: nonexistent_rule # 오류 발생!
```

**오류 메시지**: `"상속받을 규칙을 찾을 수 없습니다: nonexistent_rule"`

### 7.7. 장점 및 활용 효과

#### 1. 중복 제거

- 복잡한 조건을 한 번만 정의하고 여러 곳에서 재사용
- 소아 분류에서 성인 세부전공 조건을 그대로 활용

#### 2. 유지보수 향상

- 기본 조건이 변경되면 상속받는 모든 규칙에 자동 반영
- 일관성 있는 분류 기준 유지

#### 3. 가독성 향상

- 복잡한 조건을 간단한 참조로 표현
- 규칙 파일의 크기 대폭 감소

#### 4. 오류 감소

- 중복 작성으로 인한 오타나 불일치 방지
- 자동 검증을 통한 안전성 확보

### 7.8. 사용 권장 사항

1. **소아 분류**: 소아 + 세부전공 조합에서 적극 활용
2. **공통 조건**: 여러 규칙에서 공통으로 사용되는 복잡한 조건
3. **복잡한 로직**: 중첩된 조건이나 복잡한 논리 연산
4. **일관성 유지**: 동일한 분류 기준이 여러 곳에서 사용될 때

### 7.9. 주의사항

1. **참조 순서**: 상속받을 규칙이 먼저 정의되어야 함
2. **순환 참조 방지**: A→B→A 같은 순환 구조 금지
3. **성능 고려**: 과도한 중첩 상속은 처리 시간에 영향
4. **문서화**: 상속 관계를 명확히 문서화하여 이해도 향상

이 조건 상속 기능을 통해 복잡한 분류 규칙을 효율적으로 관리하고, 특히 소아 분류의 일관성과 정확성을 크게 향상시킬 수 있습니다.
