# Quick Reference: Semantic Data Systems Architecture

## The Three-Layer System

```
LAYER 1: Semantic Definition (OWL Ontology)
├─ 25 OWL classes (Employee, Contract, CareProcess, etc.)
├─ 164 validation rules (constraints, cardinality, datatypes)
└─ Source of truth for all downstream systems

        ↓ cdm_generate.py

LAYER 2: Relational Schema (SQL CDM)
├─ 13 SQL tables (Employee, ProjectAssignment, ClientProject, BillingEntry, etc.)
├─ Columns derived from OWL properties via cdm-owl-mapping.csv
└─ Automatically regenerated when OWL changes

        ↓ csv_to_rdf.py + validation

LAYER 3: Data Validation & Visualization
├─ CSV Validation (mandatory fields, datatypes, FK integrity)
├─ RDF Conversion (CSV → Turtle triples with proper types)
├─ Streamlit Dashboard (pass/fail metrics, drill-down, business KPIs)
└─ Stakeholder self-serve (50+ users daily)
```

---

## Key Files & Their Purpose

| File | Purpose | Input | Output |
|------|---------|-------|--------|
| `ontology.ttl` | OWL semantic definitions | N/A | RDF triples + constraints |
| `indicators/*.md` | Business requirements | N/A | Markdown with OWL concept links |
| `cdm-owl-mapping.csv` | Relational ↔ Semantic bridge | OWL ontology + SQL schema | Bidirectional mapping |
| `extract_concepts.py` | Parse markdown indicators | `indicators/*.md` | `used_classes.txt` |
| `extract_constraints.py` | Extract OWL rules | `ontology.ttl` + used classes | `constraints.csv` |
| `cdm_generate.py` | Generate SQL DDL | OWL ontology + mapping | `generated_ddl.sql` |
| `csv_to_rdf.py` | Convert CSV to RDF | CSV files + mapping | `output.ttl` + statistics |
| `validate_csv.py` | Validate CSV data | CSV files + constraints | `validation_results.csv` |
| `app.py` (Streamlit) | Interactive dashboard | Validation results + RDF | Web UI for stakeholders |

---

## The Pipeline in Action: Example

**Scenario**: Add a new requirement: "Every employee must have a nationality"

### Step 1: Update OWL Ontology
```turtle
corp:hasNationality a owl:DatatypeProperty ;
    rdfs:domain corp:Person ;
    rdfs:range xsd:string ;
    owl:minCardinality 1 .
```

### Step 2: Update Mapping CSV
```
Personnel | Employee | EmergencyContact | corp:Person | corp:hasEmergencyContact | xsd:string | 1..1 | Mandatory
```

### Step 3: Run Pipeline
```bash
python cdm_generate.py → Regenerates SQL DDL
python csv_to_rdf.py → Converts CSV to RDF with new property
python validate_csv.py → Checks all records have nationality
```

### Step 4: Results in Dashboard
- KPI: "Pass rate: 89% (150 employees missing nationality)"
- Chart: Shows which care centers have the most gaps
- Violations table: Lists the 150 employees who need nationality added

### Step 5: Stakeholder Action
Data steward sees dashboard → fixes data in source system → re-runs validation → pass rate improves

---

## Validation Strategy

### CSV Validation (Stage 1) - Fast (~5 sec for 10k records)

```python
Check Type | Description | Example
-----------|-------------|----------
mandatory | Required field present? | Voornaam NOT NULL → 50 missing
datatype | Type correct? | Geboortedatum as xsd:date → 2 format errors
fk_integrity | Foreign key exists? | EmployeeId in ProjectAssignment exists in Employee → 0 orphans
```

### RDF Validation (Stage 2) - Slower (~30 sec for 10k records)

```python
Check Type | Description | Example
-----------|-------------|----------
date_sequences | Dates make sense? | StartDatum ≤ EndDatum → 3 violations
cardinality | Property occurs N times? | Employee has exactly 1 birth date → OK
logical_consistency | Business rules? | "Active contract" = start ≤ today ≤ end → 500 active
```

### Results Distribution
- **CSV failures**: ~80% of real-world errors (missing data, wrong types, bad joins)
- **RDF failures**: ~20% (logical inconsistencies, business logic violations)

---

## Stakeholder Views

### For Data Stewards (Technical)
```
Dashboard Tab: "Data Validation"
├─ KPIs: Checks run | Records tested | Pass | Fail | Pass rate
├─ Charts: Pass/Fail per rule | Failures by check kind
├─ Table: Rule-by-rule summary (total, pass, fail, fail%)
└─ Drill-down: Sample of actual violations with field values
```

### For Care Coordinators (Business)
```
Dashboard Tab: "Workforce & Care"
├─ KPIs: Active contracts | Total FTE | Active clients | Sick leave (90d)
├─ Charts: Contract type mix | FTE per care center | Sick leave trend
└─ Action: "Regional center is understaffed. Need 2 more FTE."
```

### For Finance (Business)
```
Dashboard Tab: "Financial Summary"
├─ KPIs: Total bookings | Total charges | Reconciled | Pending
├─ Charts: Charges by cost center | Outstanding balance | Trend
└─ Action: "Cost center XYZ has $50k pending reconciliation."
```

---

## Common Patterns & Solutions

### Pattern 1: Multi-Source Same Data
**Problem**: Employee data comes from both HRDataFlow (HR system) and care system

**Solution**: Create separate OWL individuals, link with `owl:sameAs`
```turtle
hrflow:employee123 owl:sameAs care:person456 .
```

### Pattern 2: Temporal Data
**Problem**: "Show contracts active on specific date"

**Solution**: Query with date filter
```sparql
?contract corp:startDate ?start ;
          corp:endDate ?end .
FILTER (?start <= ?targetDate && (?end > ?targetDate || !BOUND(?end)))
```

### Pattern 3: Hierarchical Structures
**Problem**: Care centers organized by region, then location

**Solution**: Build graph with explicit relationships
```turtle
office123 corp:partOf region_north .
region_north corp:partOf country_nl .
```

### Pattern 4: Validation at Different Stages
**Problem**: Need strict validation in production, looser in staging

**Solution**: Version your rule sets
```python
if environment == 'production':
    RULES = STRICT_RULES  # Fail on warnings
elif environment == 'staging':
    RULES = WARNING_RULES  # Log warnings, allow
```

---

## Performance Expectations

| Operation | Input | Duration | Notes |
|-----------|-------|----------|-------|
| CSV validation | 10k records × 164 rules | 5 sec | Mandatory, datatype, FK checks |
| CSV → RDF conversion | 10k records | 10 sec | Create URIs, build graph, serialize |
| RDF semantic validation | 10k triples | 30 sec | Date checks, cardinality, logic |
| Dashboard load | 10k validation results | 2-5 sec | Cached after first load |
| Full pipeline | CSV → RDF → Dashboard | ~50 sec | Can be parallelized |

**Scaling**: These are laptop/single-core timings. For 1M records, use Spark or partition by time.

---

## Deployment Checklist

### Before Going to Production

- [ ] OWL ontology finalized and tested (version v1.x)
- [ ] CDM-OWL mapping CSV reviewed by all stakeholders
- [ ] Validation rules documented (why does each rule exist?)
- [ ] SQL DDL generated and deployed to production database
- [ ] CSV-to-RDF conversion tested on sample data
- [ ] Validation dashboard deployed (Streamlit on server or cloud)
- [ ] Stakeholders trained on dashboard (read 2 charts, drill-down, export)
- [ ] Runbook written (what do I do if dashboard is slow?)
- [ ] Disaster recovery plan (what if ontology is corrupted?)
- [ ] Monitoring set up (alert if pass rate drops)

---

## Troubleshooting

**"Dashboard is slow (>10 sec load)"**
- Check: Is database query slow? Cache the results. Is Streamlit rendering slow? Reduce chart count.

**"Pass rate suddenly dropped from 95% to 60%"**
- Check: Did source system change? Did new validation rule get added? Did schema migration happen?

**"Getting lots of date parsing errors"**
- Check: Is data in different format than expected? Use robust date parser with fallback formats.

**"Foreign key validation finds 100s of orphaned records"**
- Check: Is child table fully populated? Are records deleted after insertion? Is there a data sync issue?

---

## Cost-Benefit Analysis

### Inputs
- **Build time**: 6 weeks for Phase 1
- **Team size**: 2 engineers (semantic + data)
- **Infrastructure**: Laptop + SQL Server (existing)
- **Annual maintenance**: 40 hours

### Outputs
- **Manual validation time saved**: 10 hrs/mo → 1 hr/mo = 108 hrs/year
- **Errors prevented**: ~80% reduction (estimate: 20 → 4 per month)
- **Downstream impact**: Fewer bad data incidents = fewer emergency fixes

### ROI at $100/hour
- **Labor savings**: 108 hrs/yr × $100 = $10,800/yr
- **Emergency fixes prevented**: ~20 × $500 per fix = $10,000/yr
- **Total annual benefit**: ~$20,800/yr
- **Build cost**: 6 wks × 2 people = 480 hrs = $48,000
- **Payback period**: 480 / 20,800 × 12 = 2.8 months (in Year 1)

**Year 2+**: Ongoing maintenance cost $4,000/yr vs benefit $20,800/yr = $16,800 net gain

---

## Resources

**Standards & Specs**:
- W3C RDF Primer: https://www.w3.org/TR/rdf-primer/
- W3C OWL 2 Guide: https://www.w3.org/TR/owl2-primer/
- SPARQL Query Language: https://www.w3.org/TR/sparql11-query/

**Libraries**:
- rdflib (Python RDF): https://rdflib.readthedocs.io/
- Streamlit (dashboards): https://streamlit.io/
- pandas (data wrangling): https://pandas.pydata.org/

**Real-World Ontologies**:
- SNOMED CT (clinical terminology): https://www.snomed.org/
- LOINC (lab results): https://loinc.org/
- FHIR (healthcare data exchange): https://www.hl7.org/fhir/

---

## Key Takeaways

1. **OWL is the source of truth** → SQL DDL is derived
2. **Validation happens in two stages** → CSV (fast) + RDF (thorough)
3. **Dashboard makes quality visible** → Stakeholders take ownership
4. **Map CSV ↔ OWL bidirectionally** → Traceability from business rule to column
5. **Version your rules and ontologies** → Production vs staging versions
6. **Start small, prove value, scale** → 1 indicator → 8 indicators → 24+ indicators
7. **It's a 2-3 year journey** → Realistic expectations
8. **ROI is measurable** → Document time saved + errors prevented

---

**This is a living document.** As you implement, add your own patterns, lessons, and solutions.
