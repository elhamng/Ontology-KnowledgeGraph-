# Lessons Learned: Implementing Semantic Data Systems in Healthcare

## Introduction

Building a semantic data system is technically interesting. Deploying it in a real healthcare environment is a different challenge.

This post shares practical lessons learned from implementing OWL pipelines, CDM mappings, and validation dashboards in production healthcare systems.

---

## Lesson 1: Start Small, Prove Value, Scale Up

### What We Learned

Many organizations want to **boil the ocean**: map 50 tables to OWL, create 200 validation rules, build a dashboard, all at once.

**Result**: 18-month project, politics, abandonment.

### Better Approach

1. **Pick one indicator** (e.g., "Average staff per care center")
2. **Map 3-4 tables** to OWL
3. **Build 20-30 validation rules** specific to that indicator
4. **Build a minimal dashboard** (5 charts)
5. **Deploy to 10 stakeholders** for feedback
6. **Measure impact** (time saved, errors caught)

### Timeline
- **Week 1-2**: OWL mapping for 3 tables
- **Week 3**: CSV validation + RDF conversion for those tables
- **Week 4**: Streamlit dashboard prototype
- **Week 5-8**: Feedback, refinement, deployment
- **Month 3**: Measure ROI → justify expansion to more indicators

### Result
You have **proof of concept** in month 1. Stakeholders see value. Funding for expansion becomes easy.

---

## Lesson 2: Stakeholder Communication: Translate Between Worlds

### The Problem

**OWL team says**: "We need to add a cardinality constraint to the Employee class—it must have exactly one birth date."

**SQL team hears**: "You're adding validation. Where do I add it in my schema?"

**Business stakeholder hears**: "More red tape? Are you going to slow down data loading?"

Same thing, three different interpretations.

### The Solution: Create Translation Artifacts

1. **For business stakeholders**: Plain English
   - "Every employee must have a birth date on file. If an employee record is missing a birth date, it gets flagged as incomplete."

2. **For data teams**: SQL constraints
   - "Column `Geboortedatum` is NOT NULL. If an insert tries to skip it, the insert fails."

3. **For semantic teams**: OWL
   - `onz-g:Human` has `owl:minCardinality 1` on `onz-g:hasDateOfBirth` with range `xsd:date`

Keep a "translation table" in your documentation:

| Business Language | SQL | OWL |
|---|---|---|
| "Employee must have birth date" | Geboortedatum NOT NULL | Human minCardinality 1 hasDateOfBirth |
| "Birth date must be in the past" | CHECK(Geboortedatum < GETDATE()) | hasDateOfBirth rdfs:range xsd:date with temporal constraint |
| "Employee can work multiple contracts" | Medewerker 1:N Werkovereenkomst | Employee 0..* worksAt EmploymentContract |

---

## Lesson 3: Date Parsing is Surprisingly Hard

### The Problem

You think "I'll just parse dates from CSV." Then:

- **AFAS system exports**: `15/03/1985` (European DD/MM/YYYY)
- **Excel dumps**: `1985-03-15` (ISO format)
- **Old export files**: `1985/03/15` (ISO with slashes)
- **Timestamps included**: `1985-03-15 14:30:00` (with time)
- **Invalid data**: `32/13/1985` (non-existent date)
- **Missing data**: Empty string or "N/A"

**Naive approach**: One format `strptime('%Y-%m-%d', ...)`

**Result**: 5-10% of dates parse correctly, the rest fail.

### Better Approach: Fallback Parser

```python
def parse_date_robust(val):
    """
    Try multiple date formats in order of confidence.
    """
    if pd.isna(val) or val == '' or val == 'N/A':
        return None
    
    val = str(val).strip()
    
    formats = [
        '%Y-%m-%d',              # ISO (most common in clean data)
        '%d/%m/%Y',              # European (AFAS export)
        '%d-%m-%Y',              # European with dashes
        '%Y/%m/%d',              # ISO with slashes
        '%d/%m/%Y %H:%M:%S',     # European with timestamp
        '%Y-%m-%d %H:%M:%S',     # ISO with timestamp
    ]
    
    for fmt in formats:
        try:
            dt = datetime.strptime(val, fmt)
            # Sanity check: no dates in future, no dates before 1900
            if 1900 < dt.year < 2100:
                return dt.date()
        except ValueError:
            continue
    
    # If nothing works, log the problematic value
    logger.warning(f"Could not parse date: {val}")
    return None

# Usage
df['Geboortedatum_clean'] = df['Geboortedatum'].apply(parse_date_robust)
```

### Result
90%+ of dates now parse correctly. Remaining 5% are genuinely invalid and warrant investigation.

---

## Lesson 4: Foreign Key Integrity is Your Canary in the Coal Mine

### Why It Matters

If you see this error:
```
Rule: FK integrity check
Table: Werkovereenkomst
Error: 47 records reference Medewerker.MedewerkerId that doesn't exist
```

**This usually means**: Data is being inserted into Werkovereenkomst before Medewerker is fully loaded. Or there's a truncation/deletion mid-pipeline.

Foreign key validation catches **orchestration problems**, not just data quality issues.

### Implementation

```python
def check_foreign_key_integrity(table_with_fk, fk_column, 
                                 parent_table, parent_key):
    """
    Verify that every FK in child table exists in parent table.
    """
    fk_values = set(df_child[fk_column].dropna().unique())
    parent_values = set(df_parent[parent_key].unique())
    
    missing_fks = fk_values - parent_values
    
    return {
        'rule': f'FK {table_with_fk}.{fk_column} → {parent_table}.{parent_key}',
        'total': len(df_child),
        'valid': len(df_child[df_child[fk_column].isin(parent_values)]),
        'invalid': len(missing_fks),
        'missing_parent_keys': list(missing_fks)[:10]  # Show first 10
    }

# Usage
check_foreign_key_integrity(
    df_contracts, 'MedewerkerId',
    df_employees, 'MedewerkerId'
)
```

### Action When FK Check Fails

1. **Is parent table fully loaded?** → Wait, then retry
2. **Is there a gap in IDs?** (e.g., IDs 1-100, then jump to 200) → Investigate source system
3. **Is there a deletion happening?** → Check audit logs
4. **Is there a data quality issue in source?** → Escalate to source system owner

---

## Lesson 5: Validation Rules Need Versioning Too

### The Problem

You write a validation rule:
```
EmploymentContract: startDate must be ≤ endDate
```

**Then business says**: "We have 47 contracts that violate this. They're in a special 'dispute' status. We'll fix them later."

**Do you**:
- **A)** Mark all 47 as broken and block everything?
- **B)** Change the rule to allow violations?
- **C)** Accept violations but flag them as "known issues"?

**Answer**: **A** in production, **C** in staging.

### Solution: Rule Versioning

```python
VALIDATION_RULES = {
    'v1.0': {
        'enforcement': 'strict',
        'rules': [
            {'check': 'mandatory_fields', 'tables': ['Medewerker', 'Werkovereenkomst']},
            {'check': 'datatype_match', 'tables': ['Medewerker', 'Werkovereenkomst']},
        ]
    },
    'v1.1': {
        'enforcement': 'strict',
        'rules': [
            # All of v1.0
            # PLUS: date sequence validation
            {'check': 'date_sequences', 'tables': ['Werkovereenkomst']},
        ]
    },
    'v2.0_staging': {
        'enforcement': 'warning',
        'rules': [
            # All of v1.1
            # PLUS: new ontology constraints (in testing)
            {'check': 'new_cardinality_rules', 'tables': ['Employee']},
        ]
    }
}

def get_validation_ruleset(environment='production'):
    if environment == 'production':
        return VALIDATION_RULES['v1.1']  # Proven version
    elif environment == 'staging':
        return VALIDATION_RULES['v2.0_staging']  # New rules being tested
```

**Result**:
- Production uses proven rules
- Staging tests new rules before they're enforced
- When a new rule causes issues, you have a rollback version
- Stakeholders know when new checks are coming

---

## Lesson 6: CSV Validation Catches 80% of Errors, RDF Catches the Rest

### Reality

- **CSV validation**: Finds missing fields, wrong datatypes, FK issues → ~80% of real-world errors
- **RDF semantic validation**: Finds logical inconsistencies, complex constraints → ~20%

### Why the 80/20 Split

```
Example errors caught by CSV validation:
  - 500 records missing birth date (mandatory field)
  - 47 records with invalid email format
  - 23 Werkovereenkomst records with non-existent MedewerkerId
  → Total: 570 errors

Example errors caught by RDF/SPARQL validation:
  - Contract end date before start date
  - Employee has 3 birth dates (data duplication)
  - Care center is in two places simultaneously (bad merge)
  → Total: 47 errors
```

### Deployment Strategy

1. **Run CSV validation** (5 seconds) → block bad data immediately
2. **If CSV passes**: Convert to RDF
3. **Run RDF semantic checks** (30 seconds) → deeper validation
4. **Report by severity**:
   - CSV failures: "Data doesn't match schema"
   - RDF failures: "Data violates business logic"

### Don't build perfect RDF validation; build fast CSV validation.

---

## Lesson 7: Dashboard Performance Matters for Adoption

### The Problem

You build a beautiful dashboard with 100 charts. It takes **45 seconds** to load.

**First time users wait**: "..."  
**Second time users skip it**: "Too slow, I'll use SQL instead"

### Solution: Tiered Caching

```python
import streamlit as st

# Cache expensive operations
@st.cache_data(ttl=3600)  # Re-compute every hour
def load_validation_results():
    return pd.read_csv('validation_results.csv')

@st.cache_data(ttl=3600)
def compute_metrics():
    results = load_validation_results()
    return {
        'total_checks': len(results),
        'total_records': results['total'].sum(),
        'pass_rate': results['pass'].sum() / results['total'].sum()
    }

# Dashboard loads in <2 seconds if data hasn't changed
# Loads in 5 seconds on first run or after cache invalidation
metrics = compute_metrics()
st.metric("Pass Rate", f"{metrics['pass_rate']:.1f}%")
```

### Rule of Thumb
- **< 2 seconds**: Users will refresh multiple times
- **2-5 seconds**: Acceptable, users will use regularly
- **5-15 seconds**: Tolerable for complex queries
- **> 15 seconds**: Users abandon and use static reports instead

---

## Lesson 8: You Need a Backup Demo Method

### The Problem (Real Experience)

You're demoing the dashboard to stakeholders in a Teams meeting. Screen share starts fine. Then:
- Teams RDP layer can't access localhost:8501
- Network firewall blocks 8501
- Streamlit crashes mid-demo
- WiFi drops

**You're now 5 minutes into a 20-minute meeting with no demo.**

### Solution: Prepare Multiple Delivery Methods

1. **Live Streamlit** (ideal)
   ```bash
   streamlit run app.py --server.port=8501
   ```

2. **Recorded video** (backup 1)
   - 5-min video walkthrough, pre-recorded
   - Zero latency, no crashes
   - Share link in Teams chat if live fails

3. **PDF/Screenshots** (backup 2)
   - Screenshot dashboard, export key charts to PDF
   - Have it ready in Teams before meeting starts

4. **Jupyter notebook** (backup 3)
   - Can run cells live, show charts
   - More technical but guaranteed to work

**Pre-Meeting Checklist**:
- [ ] Live dashboard tested
- [ ] Video link ready
- [ ] PDF screenshots in Teams
- [ ] Jupyter notebook with sample data ready

---

## Lesson 9: Semantic Systems Are a Long Game

### Realistic Timeline

| Phase | Duration | Outcome |
|-------|----------|---------|
| Phase 1: Proof of concept | 2-3 months | 1 indicator, 3 tables, working dashboard |
| Phase 2: Expansion | 6 months | 8-12 indicators, 13 tables, production deployment |
| Phase 3: Operationalization | Ongoing | 24+ indicators, automated maintenance, stakeholder self-service |
| Phase 4: Evolution | Year 2+ | New ontology versions, multi-system integration, competitive advantage |

**Don't expect ROI in month 1.** Expect it in quarter 2.

---

## Lesson 10: Invest in Documentation

### Why It Matters

When a team member leaves, does your semantic system leave with them?

If your documentation is:
- Just code comments → lost
- Scattered across multiple wikis → lost
- In someone's head → definitely lost

### What to Document

1. **Architecture diagrams**: How does data flow from CSV → RDF → Dashboard?
2. **Mapping CSV**: What's the contract between CDM and OWL?
3. **Ontology primer**: What do the 25 OWL classes represent?
4. **Validation rules catalog**: What's checked and why?
5. **Dashboard user guide**: What does each chart mean?
6. **Runbooks**: "Dashboard is slow—what do I do?"
7. **Disaster recovery**: "Database crashed—what's the restore procedure?"

### Tool Recommendation
- **Markdown files in git**: Version controlled, easy to update
- **Mermaid diagrams**: Architecture + data flow diagrams as code
- **Jupyter notebooks**: Interactive documentation + example queries
- **One-page architecture summary**: Print-friendly reference

---

## Key Takeaways

1. **Start small**: One indicator, not 50
2. **Translate between worlds**: OWL, SQL, English—use translation tables
3. **Date parsing is hard**: Build a robust fallback parser
4. **FK integrity is your canary**: Catches orchestration problems
5. **Version your rules**: Production vs staging versions
6. **CSV catches 80%, RDF catches 20%**: Optimize accordingly
7. **Dashboard speed matters**: Aim for < 5 second load times
8. **Have backup demos**: Streamlit can fail; have a Plan B
9. **It's a 2-3 year journey**: Set realistic expectations
10. **Document relentlessly**: Future team members will thank you

---

## Conclusion

Building semantic data systems is hard. But the payoff—single source of truth, automatic schema evolution, AI-friendly data—is worth it.

The journey isn't "build perfect OWL ontology." It's "solve real business problems using semantic techniques, iterate, scale, repeat."

Start small. Measure impact. Scale with confidence.

---

**Image Suggestions**:
1. Timeline chart (proof of concept → full deployment)
2. Error distribution (80% CSV, 20% RDF)
3. Dashboard load time comparison (good vs bad)
4. Documentation checklist
5. Stakeholder communication translation table
