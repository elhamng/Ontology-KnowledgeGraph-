# Building an OWL Pipeline: From Business Requirements to Validation Rules

## Introduction

Most organizations document business requirements in prose:
- "Employees must have a birth date"
- "Employment contracts must have a start and end date"
- "Care centers must have at least one supervisor"

These documents are human-readable but not machine-readable. How do you automatically validate that your data follows these rules?

This post shows how to build an **OWL Pipeline** that extracts business requirements from markdown documents, converts them to formal semantic constraints, and generates downstream artifacts (validation rules, SQL schemas, Excel exports).

---

## The Problem: Manual Requirement Management

### Scenario

You have 24 business indicators (requirements) defined in markdown:

```markdown
# Indicator 1.1: Average Staff per Care Center

## Description
What is the average number of healthcare workers per care center?

## Concepts
- [Employee](http://corporate.example.org/ontology#Person)
- [Works at](http://corporate.example.org/ontology#worksAt)
- [Office Location](http://business.example.org/ontology#OfficeLocation)

## Properties
- [Birth date](http://corporate.example.org/ontology#hasDateOfBirth)
- [Employment status](http://corporate.example.org/ontology#employmentStatus)
```

**Current pain points**:
1. Manually extract which OWL classes each indicator uses
2. Manually look up constraints for each class in the OWL file
3. Manually create Excel reports for business stakeholders
4. Manually update all downstream systems when an indicator changes

**Better way**: Automate all of this.

---

## The OWL Pipeline Architecture

```
Input: Markdown Indicators (indicator_1.1.md, indicator_18.1.md, ...)
    ↓ Step 1: Extract Indicator Concepts
Intermediate: Used Classes Registry (used_classes.txt)
    ↓ Step 2: Extract OWL Constraints
Intermediate: OWL Constraint Matrix (constraints.csv)
    ↓ Step 3: Build Indicator Export
Output: Excel Indicator Reference (indicator_concepts_full.xlsx)
```

Each step is:
- **Independent**: Can be run separately
- **Reusable**: Intermediate outputs are consumed by other processes
- **Debuggable**: Errors in Step 3 don't require re-running Steps 1-2

---

## Step 1: Extract Indicator Concepts from Markdown

**Goal**: Read markdown files, extract links to OWL URIs

**Implementation**:
```python
import re
from pathlib import Path

def extract_concepts_from_markdown(md_file):
    """
    Parse markdown and extract OWL concept links.
    Looks for: [Label](http://corporate.example.org/ontology#ConceptName)
    """
    with open(md_file, encoding='utf-8') as f:
        content = f.read()
    
    # Pattern: [text](http://corporate.example.org/ontology#ConceptName)
    pattern = r'\[([^\]]+)\]\((http://corporate\.example\.org/ontology/[^)]+)\)'
    matches = re.findall(pattern, content)
    
    concepts = []
    for label, uri in matches:
        concept_name = uri.split('#')[-1]  # Get local name
        concepts.append({
            'label': label,
            'uri': uri,
            'concept_name': concept_name
        })
    
    return concepts

# Usage
indicator_file = 'indicators/indicator_1.1.md'
concepts = extract_concepts_from_markdown(indicator_file)
# Result: [
#   {'label': 'Employee', 'uri': 'http://corporate.example.org/ontology#Person', 'concept_name': 'Person'},
#   {'label': 'Works at', 'uri': 'http://corporate.example.org/ontology#worksAt', 'concept_name': 'worksAt'},
#   ...
# ]
```

**Output**: `used_classes.txt`
```
indicator_1.1 → Person, worksAt, OfficeLocation
indicator_1.2 → Assignment, startDate, endDate
indicator_18.1 → Invoice, hasAmount, Client
...
```

---

## Step 2: Extract OWL Constraints

**Goal**: For each used class, look up its constraints in the OWL ontology

**Implementation**:
```python
from rdflib import Graph, RDF, RDFS, OWL, Namespace

def extract_owl_constraints(ontology_file, class_name):
    """
    Read OWL file, extract constraints for a given class.
    Returns: cardinality, datatype, range, domain restrictions.
    """
    g = Graph()
    g.parse(ontology_file, format='turtle')
    
    constraints = []
    
    # Find the class URI
    for class_uri in g.subjects(RDF.type, OWL.Class):
        if str(class_uri).endswith(f'#{class_name}'):
            # Get all properties with this class as domain
            for prop_uri in g.subjects(RDFS.domain, class_uri):
                prop_name = str(prop_uri).split('#')[-1]
                
                # Extract constraint info
                constraint = {
                    'property': prop_name,
                    'domain': class_name,
                    'range': str(g.value(prop_uri, RDFS.range) or 'Unknown'),
                    'min_cardinality': g.value(prop_uri, OWL.minCardinality),
                    'max_cardinality': g.value(prop_uri, OWL.maxCardinality),
                }
                constraints.append(constraint)
    
    return constraints

# Usage
constraints = extract_owl_constraints('ontology.ttl', 'Human')
# Result: [
#   {
#       'property': 'hasDateOfBirth',
#       'domain': 'Human',
#       'range': 'http://www.w3.org/2001/XMLSchema#date',
#       'min_cardinality': 1,
#       'max_cardinality': 1
#   },
#   ...
# ]
```

**Output**: `constraints.csv`
```
CLASS_NAME | PROPERTY | RANGE | MIN_CARDINALITY | MAX_CARDINALITY | RULE_TYPE
Human | hasDateOfBirth | xsd:date | 1 | 1 | Mandatory
OfficeLocation | hasAddress | xsd:string | 1 | null | Mandatory
EmploymentContract | startDate | xsd:date | 1 | 1 | Mandatory
EmploymentContract | endDate | xsd:date | 0 | 1 | Optional
```

---

## Step 3: Build Indicator Excel Export

**Goal**: Cross-reference indicators with constraints, produce business-friendly Excel report

**Implementation**:
```python
import pandas as pd
from openpyxl import Workbook
from openpyxl.styles import PatternFill, Font

def build_indicator_export(indicators_file, constraints_file):
    """
    Combine indicator metadata + constraints → Excel report for stakeholders.
    """
    # Load intermediate data
    indicators = pd.read_csv(indicators_file)  # indicator_id, class_name
    constraints = pd.read_csv(constraints_file)
    
    # Create workbook
    wb = Workbook()
    ws = wb.active
    ws.title = "Indicator Concepts"
    
    # Headers
    headers = ['Indicator', 'Class', 'Property', 'Data Type', 'Required?', 'Min', 'Max', 'Business Rule']
    ws.append(headers)
    
    # For each indicator, list its classes and their constraints
    for _, indicator in indicators.iterrows():
        indicator_id = indicator['indicator_id']
        class_names = indicator['classes'].split(',')
        
        for class_name in class_names:
            # Get constraints for this class
            class_constraints = constraints[constraints['CLASS_NAME'] == class_name]
            
            for _, constraint in class_constraints.iterrows():
                is_mandatory = 'Yes' if constraint['MIN_CARDINALITY'] >= 1 else 'No'
                
                row = [
                    indicator_id,
                    class_name,
                    constraint['PROPERTY'],
                    constraint['RANGE'].split('#')[-1],  # Shorten datatype name
                    is_mandatory,
                    constraint['MIN_CARDINALITY'],
                    constraint['MAX_CARDINALITY'],
                    f"Must be {constraint['RANGE']}"
                ]
                ws.append(row)
    
    # Format: color mandatory fields red
    red_fill = PatternFill(start_color='FFCCCC', end_color='FFCCCC', fill_type='solid')
    for row in ws.iter_rows(min_row=2):
        if row[4].value == 'Yes':  # "Required?" column
            for cell in row:
                cell.fill = red_fill
    
    wb.save('indicator_concepts_full.xlsx')
    return wb

# Usage
build_indicator_export('indicators_used_classes.csv', 'constraints.csv')
# Output: Excel file with 24 indicators × multiple classes × properties = ~300 rows
```

**Output**: Excel file
```
| Indicator | Class | Property | Data Type | Required? | Business Rule |
|-----------|-------|----------|-----------|-----------|---|
| 1.1 | Human | hasDateOfBirth | date | Yes | Must be xsd:date |
| 1.1 | OfficeLocation | hasAddress | string | Yes | Must be xsd:string |
| 1.2 | EmploymentContract | startDate | date | Yes | Must be xsd:date |
| 1.2 | EmploymentContract | endDate | date | No | Optional xsd:date |
```

---

## Why This Pipeline Matters

### Benefit 1: Single Source of Truth
Change the OWL ontology → all three steps re-run → Excel export updates automatically.

No more manual sync between ontology, documentation, and validation rules.

### Benefit 2: Version Control
Each indicator version is tracked:
- v1.0: Indicator 1.1 uses [Person, OfficeLocation]
- v1.1: Indicator 1.1 adds [Employment Contract]
- v2.0: Indicator 1.1 changes to [Human, CareCenter] (ontology refactoring)

### Benefit 3: Stakeholder Communication
Excel is familiar. Non-technical users can review the requirements without understanding RDF syntax.

### Benefit 4: Downstream Automation
Other systems consume `constraints.csv`:
- **Validation system**: "Is this record valid?"
- **Schema generator**: "Create SQL DDL from OWL"
- **Data quality dashboard**: "Which rules failed?"

---

## Real-World Example: Handling Multiple Ontology Versions

You have 3 versions of the healthcare ontology running in parallel:

- **v1.3.2**: Old version (15 concepts)
- **v1.3.4**: Current version (20 concepts, bug fixes)
- **v2.0.0**: New version (25 concepts, breaking changes)

**Question**: Which indicators work with which versions?

**Solution**: Build a version matrix:

```
indicator_id | v1.3.2 | v1.3.4 | v2.0.0 | Status
1.1          | ✓      | ✓      | ✓      | Compatible
1.2          | ✓      | ✓      | ✗      | Breaks in v2
18.1         | ✗      | ✓      | ✓      | New in v1.3.4
```

This tells you:
- Indicators 1.1 can run on any version (upgrade any time)
- Indicator 1.2 breaks in v2.0 (needs code changes before upgrade)
- Indicator 18.1 only works with v1.3.4+ (don't run on v1.3.2)

---

## Key Takeaways

1. **Markdown → OWL → Validation**: Don't manually maintain these relationships
2. **Three-stage pipeline**: Extraction → Constraint extraction → Export
3. **Intermediate files are valuable**: Use them in other processes (validation, schema generation)
4. **Stakeholder-friendly output**: Excel exports build trust with business teams
5. **Version management**: Track which indicators work with which ontology versions

---

## Next Steps

In the next post, we'll see how to use these constraints to automatically **validate data** and generate **SQL schemas** from the OWL ontology.

---

**Image Suggestions**:
1. Pipeline diagram (Markdown → Extract → Constraints → Excel)
2. Sample Excel export (colored by mandatory vs optional)
3. Version matrix table (indicators × ontology versions)
4. Code snippet showing regex extraction
