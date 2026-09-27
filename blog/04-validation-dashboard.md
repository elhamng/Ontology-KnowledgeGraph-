# Building a Data Validation Dashboard: Making Quality Visible

## Introduction

You have 10,000 healthcare records and 164 validation rules. Do they all pass? Which ones fail? Which records are problematic?

Without visibility, data quality issues slip through to downstream systems. This post shows how to build an **interactive Streamlit dashboard** that makes data quality visible to all stakeholders.

---

## The Problem: Data Quality Blindness

### Scenario

Your data warehouse processes 10,000 employee records monthly:

```csv
EmployeeId,FirstName,DateOfBirth,StartDate,EndDate
123,John,1985-03-15,2020-01-01,2023-06-30
124,Jane,,2020-01-15,  # Missing birth date!
125,Bob,1990-12-25,2023-01-01,2022-12-31  # End < Start!
...
```

**Current situation**:
- Data lands in the warehouse
- Stakeholders discover errors weeks later
- By then, errors have propagated to financial reports, staffing dashboards, etc.
- Audit requires explaining why bad data reached production

**Better way**: Validate immediately, show results visually, let data stewards fix before handoff.

---

## Solution: Two-Stage Validation + Streamlit Dashboard

```
CSV Data (Employee.csv, ProjectAssignment.csv, ...)
    ↓
Stage 1: CSV Validation (fast, schema-level)
  ✓ Mandatory fields present?
  ✓ Datatypes correct?
  ✓ Foreign keys exist?
    ↓
Stage 2: RDF Validation (slower, semantic-level)
  ✓ Date sequences valid?
  ✓ Business logic (e.g., "active" contracts)
    ↓
Streamlit Dashboard (visualize results)
  ✓ Pass/fail metrics
  ✓ Failure patterns
  ✓ Drill-down to problem records
```

---

## Stage 1: CSV Validation

### Implementation

```python
import pandas as pd
from datetime import datetime

def validate_csv(csv_file, mapping_file):
    """
    CSV-level validation: mandatory fields, datatypes, FK integrity.
    """
    df = pd.read_csv(csv_file)
    mapping = pd.read_csv(mapping_file)
    
    results = []
    
    # For this table, get all constraints
    table_constraints = mapping[mapping['CDM_TABLE'] == 'Employee']
    
    for _, constraint in table_constraints.iterrows():
        col_name = constraint['CDM_FIELD']
        xsd_type = constraint['XSD_TYPE']
        cardinality = constraint['CARDINALITY']
        is_mandatory = cardinality.split('..')[0] == '1'
        
        # Test 1: Mandatory field check
        if is_mandatory:
            missing_count = df[col_name].isna().sum()
            total_count = len(df)
            
            results.append({
                'rule_number': f"{constraint['DOMAIN']}_R{len(results)+1}",
                'check_kind': 'mandatory',
                'cdm_table': 'Employee',
                'cdm_field': col_name,
                'total': total_count,
                'pass': total_count - missing_count,
                'fail': missing_count,
                'fail_pct': round(100 * missing_count / total_count, 2),
                'ontology_rule': f"minCardinality 1"
            })
        
        # Test 2: Datatype check
        if xsd_type == 'xsd:date':
            date_errors = 0
            for val in df[col_name].dropna():
                try:
                    datetime.strptime(str(val), '%Y-%m-%d')
                except ValueError:
                    date_errors += 1
            
            results.append({
                'rule_number': f"{constraint['DOMAIN']}_R{len(results)+1}",
                'check_kind': 'datatype',
                'cdm_table': 'Employee',
                'cdm_field': col_name,
                'total': len(df),
                'pass': len(df) - date_errors,
                'fail': date_errors,
                'fail_pct': round(100 * date_errors / len(df), 2),
                'ontology_rule': f"range xsd:date"
            })
    
    return pd.DataFrame(results)

# Usage
validation_results = validate_csv('data/Employee.csv', 'cdm-owl-mapping.csv')
```

**Output Example**:
```
rule_number | check_kind | cdm_table | cdm_field | total | pass | fail | fail_pct | ontology_rule
R1          | mandatory  | Employee | FirstName | 10000 | 10000 | 0    | 0.0  | minCardinality 1
R2          | mandatory  | Employee | DateOfBirth | 10000 | 9950 | 50   | 0.5  | minCardinality 1
R3          | datatype   | Employee | DateOfBirth | 10000 | 9998 | 2    | 0.02 | range xsd:date
```

---

## Stage 2: CSV → RDF Conversion

Convert valid CSV rows to RDF triples:

```python
from rdflib import Graph, Literal, Namespace, RDF, RDFS, XSD, URIRef

def csv_to_rdf(csv_file, mapping_file, output_ttl):
    """
    Convert CSV to RDF Turtle, creating URIs and proper datatype literals.
    """
    df = pd.read_csv(csv_file)
    mapping = pd.read_csv(mapping_file)
    
    g = Graph()
    
    # Bind namespaces
    CORP_CORE = Namespace('http://corporate.example.org/ontology#')
    CORP_PERSONNEL = Namespace('http://corporate.example.org/ontology#')
    CORP_BUSINESS = Namespace('http://corporate.example.org/ontology#')

    g.bind('corp', CORP_CORE)
    g.bind('corp-personnel', CORP_PERSONNEL)
    g.bind('corp-business', CORP_BUSINESS)
    
    # For each row
    for idx, row in df.iterrows():
        # Create individual URI
        person_id = row['EmployeeId']
        uri = URIRef(f"{CORP_CORE}employee{person_id}")
        
        # Add type
        g.add((uri, RDF.type, CORP_CORE.Person))
        
        # Add properties with type casting
        g.add((uri, RDFS.label, Literal(f"{row['FirstName']}")))
        
        if pd.notna(row['DateOfBirth']):
            # Parse and cast to xsd:date
            birth_date = datetime.strptime(str(row['DateOfBirth']), '%Y-%m-%d').date()
            g.add((uri, CORP_CORE.hasDateOfBirth, Literal(birth_date, datatype=XSD.date)))
    
    # Serialize to Turtle
    g.serialize(destination=output_ttl, format='turtle')

# Usage
csv_to_rdf('data/Employee.csv', 'cdm-owl-mapping.csv', 'output/employee.ttl')
```

**Output Example** (`employee.ttl`):
```turtle
@prefix corp: <http://corporate.example.org/ontology#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

corp:employee123 a corp:Person ;
    rdfs:label "John" ;
    corp:hasDateOfBirth "1985-03-15"^^xsd:date .

corp:employee124 a corp:Person ;
    rdfs:label "Jane" .
    # Note: no birth date (null)
```

---

## Streamlit Dashboard: Visualizing Quality

### Overview Page

```python
import streamlit as st
import plotly.express as px
import pandas as pd

st.set_page_config(page_title="Data Quality Dashboard", layout="wide")

st.title("Data Quality Validation Dashboard")

# Load validation results
results = pd.read_csv('validation_results.csv')

# KPI row
col1, col2, col3, col4, col5 = st.columns(5)
col1.metric("Checks run", len(results))
col2.metric("Records tested", int(results['total'].sum()))
col3.metric("Pass", int(results['pass'].sum()))
col4.metric("Fail", int(results['fail'].sum()))
col5.metric("Pass rate", f"{100*results['pass'].sum()/results['total'].sum():.1f}%")

# Pass/Fail by rule (bar chart)
st.subheader("Pass/Fail per Rule")
df_chart = results.copy()
df_chart['label'] = df_chart['cdm_table'] + ' · ' + df_chart['cdm_field']

df_long = df_chart.melt(
    id_vars=['label'], value_vars=['pass', 'fail'],
    var_name='Status', value_name='Count'
)

fig = px.bar(
    df_long, x='Count', y='label', color='Status', orientation='h', barmode='stack',
    color_discrete_map={'pass': '#70AD47', 'fail': '#C0504D'},
    height=400
)
st.plotly_chart(fig, use_container_width=True)

# Failure breakdown (pie chart)
st.subheader("Failures by Check Kind")
failures_by_kind = results.groupby('check_kind')['fail'].sum()
fig2 = px.pie(
    names=failures_by_kind.index, values=failures_by_kind.values,
    title="Failure Type Distribution",
    color_discrete_map={
        'mandatory': '#C0504D',
        'datatype': '#ED7D31',
        'fk_exists': '#9B59B6'
    }
)
st.plotly_chart(fig2, use_container_width=True)
```

### Drill-Down Page

```python
# Show actual violations
st.subheader("Problem Records (Sample)")

violations = pd.read_csv('violations.csv')

# Filter controls
kind_filter = st.multiselect(
    "Filter by check kind",
    options=violations['check_kind'].unique(),
    default=violations['check_kind'].unique()
)

violations_filtered = violations[violations['check_kind'].isin(kind_filter)]

st.dataframe(
    violations_filtered[[
        'row_id', 'cdm_field', 'actual_value', 'violation_message'
    ]].head(100),
    use_container_width=True
)

# Summary per table
st.subheader("Failures by Table")
by_table = results.groupby('cdm_table').agg({
    'fail': 'sum',
    'pass': 'sum',
    'fail_pct': 'mean'
}).reset_index()

st.dataframe(by_table.sort_values('fail', ascending=False))
```

---

## Real-World Dashboard: Workforce & Care Metrics

For non-technical stakeholders (care coordinators, managers):

```python
# Show business metrics derived from validated data
st.subheader("Staffing Dashboard")

# Active contracts
active_contracts = len(df[
    (df['StartDatum'] <= today) & 
    ((df['EindDatum'].isna()) | (df['EindDatum'] >= today))
])

col1, col2, col3 = st.columns(3)
col1.metric("Active Contracts", active_contracts)
col2.metric("Total FTE", round(total_hours / 36.0, 1))
col3.metric("Sick Leave (90d)", sick_periods_count)

# Contract type pie chart
contract_types = df['ContractType'].value_counts()
fig = px.pie(names=contract_types.index, values=contract_types.values)
st.plotly_chart(fig, use_container_width=True)

# FTE per care center bar chart
fte_per_location = df.groupby('OfficeLocation')['FTE'].sum().sort_values()
fig = px.bar(
    x=fte_per_location.values, y=fte_per_location.index, orientation='h'
)
st.plotly_chart(fig, use_container_width=True)
```

---

## Impact: Before vs After

### Before Dashboard
- Data quality issues discovered weeks later
- Manual reconciliation: 10 hours/month per data steward
- Stakeholders ask "Is the data ready?" → Takes 2 hours to answer
- Error escape rate: ~20% (bad data reaches production)

### After Dashboard
- Issues caught immediately (same day validation)
- Self-serve validation: 1 hour/month (data stewards use dashboard)
- Stakeholders answer themselves: "Is data ready?" → 30 seconds
- Error escape rate: ~2% (90% improvement)

**Business impact**: 
- 72 hours saved per person per year (8 hrs → 1 hr monthly)
- At $100/hour = $7,200 per person annually
- With 5 data stewards = $36,000 annual savings
- Plus avoided downstream costs from bad data reaching production

---

## Key Takeaways

1. **Two-stage validation**: CSV (fast) + RDF (semantic)
2. **Visibility = Ownership**: Stakeholders see quality issues, take responsibility for fixing upstream
3. **Self-serve reduces dependency**: Data stewards use dashboard instead of asking engineers
4. **Visual + drill-down**: High-level KPIs for executives, details for data quality teams
5. **Business metrics**: Show non-technical stakeholders why quality matters (staffing, costs)

---

## Next Steps

In the next post, we'll discuss how to **deploy this dashboard** to production and make it **accessible to remote teams** via cloud platforms.

---

**Image Suggestions**:
1. KPI cards showing pass rate, fail count
2. Pass/Fail bar chart (colored by status)
3. Failure breakdown pie chart (by check kind)
4. Sample violations table
5. Staffing dashboard (FTE per location, contract types)
6. Before/after metrics comparison
