# Bridging Relational and Semantic: CDM to OWL Mapping

## Introduction

Most organizations use **relational databases** (SQL Server, PostgreSQL) for operational data. But modern systems increasingly use **semantic/graph** approaches (RDF, SPARQL).

How do you connect the two worlds?

This post explains how to map a **Common Data Model (CDM)** in SQL to an **OWL ontology**, and automatically generate SQL schemas from semantic definitions.

---

## The Problem: Schema Drift

### Scenario

You have a **multi-office consulting firm** with:
- **Relational CDM**: 13 SQL tables (Employee, ProjectAssignment, ClientProject, BillingEntry, etc.)
- **OWL Ontology**: 25 OWL classes describing business concepts (Person, Assignment, Client, Invoice, etc.)
- **Business requirement**: Keep them in sync across all offices and clients

**What happens**:
1. Business changes requirement: "Employees must have emergency contact info"
2. Ontology team updates OWL: Add `hasEmergencyContact` property to `Person` class
3. Data team manually updates SQL: `ALTER TABLE Employee ADD EmergencyContact VARCHAR(50)`
4. Validation team manually updates rules: "EmergencyContact must be a valid phone number"
5. BI team manually updates reports: Add emergency contact field to HR dashboards

**Result**: 4 manual steps, high error rate, OWL and SQL gradually drift apart.

**Better way**: Automation.

---

## Solution: Ontology-Driven Schema Generation

```
OWL Ontology (source of truth)
    ↓ cdm_generate.py
SQL DDL (derived from OWL)
```

When ontology changes, SQL automatically regenerates.

---

## The CDM-OWL Mapping Framework

### Step 1: Define the Mapping

Create a CSV file (`cdm-owl-mapping.csv`) that documents the relationship between CDM tables and OWL classes:

```
DOMAIN | CDM_TABLE | CDM_FIELD | CLASS_NAME | ONTOLOGY_PROPERTY | XSD_TYPE | CARDINALITY | CONSTRAINT_TYPE
Personnel | Employee | EmployeeId | biz:Person | rdf:type | N/A | 1..1 | Key
Personnel | Employee | FirstName | biz:Person | rdfs:label | xsd:string | 1..1 | Mandatory
Personnel | Employee | DateOfBirth | biz:Person | biz:hasDateOfBirth | xsd:date | 0..1 | Optional
Personnel | Employee | EmergencyContact | biz:Person | biz:hasEmergencyContact | xsd:string | 0..1 | Optional
Assignments | ProjectAssignment | AssignmentId | biz:Assignment | rdf:type | N/A | 1..1 | Key
Assignments | ProjectAssignment | StartDate | biz:Assignment | biz:startDate | xsd:date | 1..1 | Mandatory
Assignments | ProjectAssignment | EndDate | biz:Assignment | biz:endDate | xsd:date | 0..1 | Optional
Clients | ClientProject | ProjectId | biz:ClientProject | rdf:type | N/A | 1..1 | Key
Clients | ClientProject | ClientId | biz:ClientProject | biz:forClient | biz:Client | 1..1 | Foreign Key
Finance | BillingEntry | EntryId | biz:Invoice | rdf:type | N/A | 1..1 | Key
Finance | BillingEntry | Amount | biz:Invoice | biz:hasAmount | xsd:decimal | 1..1 | Mandatory
```

This CSV is the **contract** between relational and semantic teams.

### Step 2: XSD Type Mapping

Map OWL XSD types to SQL types:

```python
XSD_TO_SQL = {
    'xsd:string':      'VARCHAR(255)',
    'xsd:date':        'DATE',
    'xsd:dateTime':    'DATETIME',
    'xsd:decimal':     'DECIMAL(18,4)',
    'xsd:integer':     'INT',
    'xsd:boolean':     'BIT',
    'xsd:anyURI':      'VARCHAR(500)',
}

# Example: onz-g:hasDateOfBirth with range xsd:date → SQL column DATE
```

### Step 3: Generate SQL DDL

```python
def generate_ddl_from_mapping(mapping_csv, output_file):
    """
    Read CDM-OWL mapping, generate SQL CREATE TABLE statements.
    """
    import pandas as pd
    
    df = pd.read_csv(mapping_csv)
    
    sql_statements = []
    
    # Group by CDM table
    for table_name, table_df in df.groupby('CDM_TABLE'):
        columns = []
        
        for _, row in table_df.iterrows():
            col_name = row['CDM_FIELD']
            xsd_type = row['XSD_TYPE']
            cardinality = row['CARDINALITY']
            
            # Map XSD to SQL
            sql_type = XSD_TO_SQL.get(xsd_type, 'VARCHAR(255)')
            
            # Determine if NOT NULL
            is_mandatory = cardinality.split('..')[0] == '1'
            not_null = 'NOT NULL' if is_mandatory else 'NULL'
            
            columns.append(f"    {col_name} {sql_type} {not_null}")
        
        # Build CREATE TABLE statement
        create_stmt = f"""
CREATE TABLE dbo.{table_name} (
{',\n'.join(columns)},
    PRIMARY KEY ([{df[df['CDM_TABLE']==table_name].iloc[0]['CDM_FIELD']}])
);
"""
        sql_statements.append(create_stmt)
    
    # Write to file
    with open(output_file, 'w') as f:
        f.write('\n'.join(sql_statements))

# Usage
generate_ddl_from_mapping('cdm-owl-mapping.csv', 'generated_ddl.sql')
```

**Output**: `generated_ddl.sql`
```sql
CREATE TABLE dbo.Employee (
    EmployeeId INT NOT NULL,
    FirstName VARCHAR(255) NOT NULL,
    DateOfBirth DATE NULL,
    EmergencyContact VARCHAR(255) NULL,
    PRIMARY KEY (EmployeeId)
);

CREATE TABLE dbo.ProjectAssignment (
    AssignmentId INT NOT NULL,
    StartDate DATE NOT NULL,
    EndDate DATE NULL,
    PRIMARY KEY (AssignmentId)
);
```

---

## Handling Ontology Versions

Healthcare systems often run multiple ontology versions:

- **CareFund Profile** (healthcare fund management): 24 indicators covering HR + care + finance
- **HealthAudit Profile** (regulatory compliance): 4 indicators covering care + workforce + planning

**Question**: Which CDM tables are required for which profile?

**Solution**: Version coverage matrix

```python
def build_version_coverage_matrix(mapping_file, ontology_versions):
    """
    For each CDM table and each ontology version, track:
    - Does this table exist in this version?
    - Are its constraints the same or different?
    """
    results = []
    
    for version in ontology_versions:
        ontology = load_ontology(f'ontology_v{version}.ttl')
        
        for table in unique_tables(mapping_file):
            classes_in_table = mapping_file[mapping_file['CDM_TABLE'] == table]['CLASS_NAME'].unique()
            
            # Check if all classes exist in this ontology version
            all_exist = all(class_uri_exists(c, ontology) for c in classes_in_table)
            
            results.append({
                'table': table,
                'version': version,
                'exists': all_exist,
                'num_constraints': count_constraints(ontology, classes_in_table)
            })
    
    return pd.DataFrame(results)

# Output:
# table          | v1.3.2 | v1.3.4 | v2.0.0
# Medewerker     | ✓ (8)  | ✓ (8)  | ✓ (9)    # v2 added 1 constraint
# Werkovereenkomst | ✓ (5) | ✓ (6)  | ✗        # v2 removed this table
# VerpleegProcess | ✗      | ✓ (4)  | ✓ (4)    # New in v1.3.4
```

This tells you:
- **Medewerker**: Same in all versions (can upgrade anytime)
- **Werkovereenkomst**: Removed in v2 (breaking change, needs code review)
- **VerpleegProcess**: New in v1.3.4 (don't reference in old code)

---

## Bidirectional Traceability

The mapping enables **traceability in both directions**:

### Question 1: "Why does CDM table Employee have a DateOfBirth column?"

**Answer** (via mapping):
- → `onz-g:hasDateOfBirth` property
- → `onz-g:Human` class
- → Required by indicators 1.1, 3.2, 5.0
- → Regulatory requirement (healthcare law requires birth date tracking)

### Question 2: "If I change the OWL ontology to make birth date optional, what breaks?"

**Answer** (via mapping):
- Birth date constraint changes from `minCardinality 1` → `minCardinality 0`
- CDM column `Geboortedatum` changes from `NOT NULL` → `NULL`
- All validation rules that check "birth date is required" must be updated
- Indicators 1.1, 3.2, 5.0 must be re-tested

---

## Real-World Example: Adding a New Field

**Scenario**: Healthcare regulator now requires all employees to have a nationality field.

**Step 1**: Update OWL ontology
```turtle
onz-g:hasNationality a owl:DatatypeProperty ;
    rdfs:domain onz-g:Human ;
    rdfs:range xsd:string ;
    owl:minCardinality 1 .
```

**Step 2**: Update mapping CSV
```
Personnel | Employee | EmergencyContact | biz:Person | biz:hasEmergencyContact | xsd:string | 1..1 | Mandatory
```

**Step 3**: Regenerate SQL
```bash
python cdm_generate.py --mapping cdm-owl-mapping.csv --output generated_ddl.sql
```

**Step 4**: New SQL reflects the change
```sql
ALTER TABLE dbo.Employee ADD EmergencyContact VARCHAR(255) NOT NULL;
```

**Step 5**: Validation rules auto-update
- "Employee.EmergencyContact is mandatory" ← derived from OWL minCardinality

**Result**: One change in OWL → cascades through SQL, validation, BI dashboards automatically.

---

## Key Benefits

| Benefit | Impact |
|---------|--------|
| **Single source of truth** | OWL ontology is canonical; SQL is derived |
| **Automatic schema evolution** | No manual SQL DDL edits |
| **Regulatory compliance** | Audit trail shows which requirements drive which columns |
| **Multi-profile support** | Same CDM can serve CareFund, HealthAudit, and custom profiles |
| **Version compatibility matrix** | Know which tables work with which ontology versions |
| **Breaking change detection** | Upgrade automated but breaking changes flagged |

---

## Limitations & Caveats

**What this approach handles**:
- Primitive types (string, date, decimal, boolean)
- 1:1 mappings (one column → one property)
- Simple hierarchies

**What it doesn't handle** (yet):
- Complex types (nested objects, arrays)
- Many-to-many relationships (e.g., employee works at multiple locations)
- Full SHACL shape language (only basic cardinality)
- Dynamic/temporal constraints ("contract must have start ≤ end")

For complex constraints, you'd add:
- SHACL definitions (semantic validation)
- Custom Python validation logic
- Database triggers/constraints

---

## Key Takeaways

1. **Mapping file is the contract**: CDM-OWL mapping CSV documents the relationship
2. **XSD to SQL type mapping**: Bridges semantic and relational type systems
3. **Ontology-driven generation**: SQL schemas are derived from OWL, not independent
4. **Version management**: Track which tables exist in which ontology versions
5. **Bidirectional traceability**: Navigate from business requirement → OWL class → SQL column and back

---

## Next Steps

In the next post, we'll see how to **validate data** against these ontology constraints, and build a dashboard for stakeholders to see data quality issues in real time.

---

**Image Suggestions**:
1. Mapping CSV table (CDM ↔ OWL alignment)
2. XSD to SQL type mapping table
3. Version compatibility matrix (tables × versions)
4. Bidirectional traceability diagram (Requirement → OWL → SQL → Validation)
5. Before/after: manual updates vs automatic generation
