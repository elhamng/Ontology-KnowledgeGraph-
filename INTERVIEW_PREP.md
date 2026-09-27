# Interview Preparation: KIK-V Linked Data Projects

This document prepares you for a linked data / semantic modeling interview by grounding everything in your actual KIK-V projects. Use this as a reference to tell concrete stories about what you built, the problems you solved, and the technical choices you made.

---

## Part 1: Three Concrete Project Examples

### Project 1: OWL Pipeline & Indicator Ontology System

#### **The Problem**
You needed to translate business requirements (indicators like "What is the average number of staff?") into formal semantic constraints. The challenge: indicators are defined by business analysts in markdown documents with links to OWL concepts, but you need to:
- Extract semantic meaning from those links
- Validate which OWL classes and properties apply to each indicator
- Generate downstream artefacts (Excel exports, DDL, validation rules) from the ontology
- Manage multiple ontology versions simultaneously

#### **Your Architecture**
```
Markdown Indicators (indicator 1.1, 18.1, etc.)
    ↓ step1_extract_indicator_concepts.py
Used Classes Registry (used_classes.txt)
    ↓ step2_extract_owl_constraints.py
OWL Constraint Matrix (constraints.csv)
    ↓ step3_build_indicator_export.py
Excel Indicator Reference (indicator_concepts_full.xlsx)
```

**Key components:**
- **Step 1**: Parses markdown frontmatter (title, YAML) + extracts links to OWL URIs from three sections:
  - Concepten (owl:Class) — e.g., `[EindSaldo](http://purl.org/ozo/onz-fin#EindSaldo)`
  - Relaties (ObjectProperty) — e.g., `[deel van](http://purl.org/ozo/onz-g#partOf)`
  - Eigenschappen (DatatypeProperty) — e.g., `[heeftGeldBedrag](http://purl.org/ozo/onz-fin#heeftGeldBedrag)`
  
- **Step 2**: Reads OWL files, extracts constraints (cardinality, range, domain, XSD types)

- **Step 3**: Cross-references indicator concepts with constraints, produces enriched Excel for business stakeholders

#### **Your Role**
- Designed the three-stage pipeline architecture
- Implemented semantic extraction (parsing RDF URIs, managing namespaces)
- Chose regex-based link extraction for robustness with markdown variations
- Handled version merging: ontology can have 4–5 versions in parallel, each with different constraints

#### **Technical Choices & Reasoning**
1. **Markdown + URI links** (vs. structured formats like TOML):
   - Stakeholders already write markdown, integrates naturally
   - URIs are self-documenting and namespace-aware
   
2. **Three-stage pipeline** (vs. monolithic):
   - Separation of concerns: extraction → constraint logic → export
   - Reusable intermediate outputs (used_classes.txt used by validation system)
   - Faster iteration: fixing Step 3 doesn't re-run Step 1
   
3. **XSD type mapping** (decimal → DECIMAL(18,4), date → DATE):
   - Bridges semantic (OWL/XSD) and relational (SQL Server) worlds
   - Ensures type safety downstream in CDM validation

#### **Result & Impact**
- Enables 24+ indicators to be validated against 25 OWL classes
- Single source of truth: changes to ontology automatically propagate through pipeline
- Business and tech teams share one reference: the Excel export
- Reduced manual validation errors by ~80% (rough estimate based on handoff improvements)

---

### Project 2: CDM → OWL Mapping & Ontology-Driven Schema Generation

#### **The Problem**
You have:
- A relational Common Data Model (CDM) with 13 tables (Medewerker, Werkovereenkomst, VerpleegProcess, etc.)
- An OWL ontology describing healthcare/personnel/financial concepts
- Multiple stakeholders (AFAS ERP users, care coordinators, finance team) who need to understand how relational tables map to semantic concepts

The challenge: **Keep the CDM aligned with the ontology as it evolves, without manual re-mapping.**

#### **Your Architecture**

```
Ontology Versions (runs/ folder)
  ├─ run_2024_01 (v1.3.2)
  ├─ run_2024_06 (v1.3.4)
  └─ run_2024_09 (v2.0.0)
    ↓ cdm_generate.py
Merged Constraints
    ├─ cdm_class_matrix.csv       (which classes cover which versions/profiles)
    ├─ cdm-owl-mapping.csv         (13 tables → 25 classes, 96 column mappings)
    └─ 03_cdm_ontology_ddl.sql     (SQL Server schema derived from OWL)
```

**Key decisions:**
- **Single source of truth**: OWL ontology, not the relational schema
- **Bidirectional traceability**: Every SQL column traces back to an OWL property and XSD type
- **Version coverage matrix**: Shows which tables/classes exist in which profile (ZK vs IGJ)

#### **Technical Choices**

1. **XSD → SQL type mapping**:
   ```python
   XSD_TO_SQL = {
       "date":        "DATE",
       "dateTime":    "DATETIME",
       "decimal":     "DECIMAL(18,4)",
       "string":      "VARCHAR(255)",
       "boolean":     "BIT",
   }
   ```
   - Enforces semantic types at storage layer
   - SQL Server constraints prevent invalid data at insert time
   
2. **Semantic versioning** for ontology releases:
   - Parse '1.3.4' → (1, 3, 4) for proper sorting
   - Tracks breaking changes (major), new classes (minor), fixes (patch)
   
3. **Profiles** (ZK vs IGJ):
   - ZK profile: 24 indicators covering HR + care + finance
   - IGJ profile: 4 indicators covering care + workforce + planning
   - CDM accommodates both without duplication

#### **Your Role**
- Designed the mapping framework and profile concept
- Implemented constraint merging across ontology versions (handling duplicates, conflicts)
- Built the type mapping layer ensuring semantic → relational coherence

#### **Result & Impact**
- When ontology adds a new property, you regenerate DDL and all downstream validation
- Stakeholders can trace: "Why does this field exist?" → "Because the ontology requires it for indicator 11.3"
- Reduced schema drift: ontology and relational model stay synchronized

---

### Project 3: Validation Dashboard & CSV → RDF Conversion

#### **The Problem**
You have CDM data in CSV files (13 tables exported from data warehouse) and an OWL ontology describing what that data *should* look like. How do you:
1. Validate CSV data against semantic constraints (mandatory fields, cardinality, date ordering)?
2. Expose validation failures to business users (data stewards, care coordinators)?
3. Convert CSV to RDF/Turtle for Linked Data systems?
4. Let stakeholders explore data quality visually?

#### **Your Architecture**

**Validation Pipeline:**
```
CDM CSV Files (Medewerker.csv, Werkovereenkomst.csv, etc.)
    ↓
validate_csv.py (checks against cdm-owl-mapping.csv rules)
    ├─ Mandatory fields present?
    ├─ FK integrity between tables?
    ├─ Date sequences valid (start ≤ end)?
    └─ Datatypes correct (string vs decimal)?
    ↓
Validation Report (pass/fail per rule)
```

**RDF Conversion:**
```
CDM CSV Files + cdm-owl-mapping.csv
    ↓ csv_to_rdf_lib.py
RDF Graph (rdflib Graph object)
    ├─ Pattern: onz-zorg:medewerker{key}
    │   a onz-g:Human
    │   rdfs:label "{key}"
    │   onz-g:hasDateOfBirth "{date}"^^xsd:date
    ├─ Namespaces: onz-g, onz-pers, onz-fin, onz-zorg, onz-org
    └─ ~500 rows per table (configurable limit for dev)
    ↓
Turtle Output (rdf_output.ttl) + Statistics (rdf_stats.json)
```

**Dashboard (Streamlit):**
```
📋 Concepts          → All OWL classes, colour-coded by ontology
📐 OWL Rules         → Constraint table with filters
✅ PULSE Validation  → Enriched columns (ontology rules vs CSV rules)
🗂️ Field Mapping     → CDM table → OWL class mappings
🧪 Data Validation   → Pass/fail per rule, violation samples
📈 Analysis          → Charts of rule types, scope, class coverage
🌐 Overview          → Summary metrics + all indicators
🔗 RDF Graph         → Triple stats, namespace viewer, TTL preview
```

#### **Technical Choices**

1. **rdflib for CSV → RDF/Turtle conversion** (vs. raw string templates or streaming tools):
   - **Why rdflib**: Proper URI escaping, namespace binding, XSD literal type safety, Pythonic API
   - **Composable**: Build RDF graph programmatically from CSV rows, then serialize to Turtle format
   - **Type safety**: Declare data types explicitly (`Literal("1985-03-15", datatype=XSD.date)`)
   - **Namespace management**: Bind five namespaces to graph, reuse consistently across 13 tables
   
   Example from your code:
   ```python
   from rdflib import Graph, Literal, Namespace, RDF, RDFS, XSD, URIRef
   
   g = Graph()
   g.bind("onz-g", ONZ_G)      # General healthcare namespace
   g.bind("onz-fin", ONZ_FIN)  # Financial namespace
   
   # For each row in CSV:
   individual = URIRef(f"{ONZ_ZORG}medewerker{safe_local(key)}")
   g.add((individual, RDF.type, ONZ_G.Human))
   g.add((individual, RDFS.label, Literal(key)))
   g.add((individual, ONZ_G.hasDateOfBirth, Literal(date_val, datatype=XSD.date)))
   
   # Serialize to Turtle:
   g.serialize(destination="output.ttl", format="turtle")
   ```
   
   **Advantages over alternatives:**
   - **vs. raw templates**: Handles escaping, namespace prefixes automatically; less error-prone
   - **vs. RML/YARRRML**: For straightforward CSV mapping, simpler to implement; rdflib better for custom logic (like fallback date parsing)
   - **vs. streaming tools**: rdflib loads graph in memory (fine for 500 rows per table); streaming tools needed only if datasets > GBs

2. **Five parallel namespaces** for semantic domain coverage:
   - `onz-g` (general): Human, hasDate, partOf, isAbout
   - `onz-pers` (personnel): Werkovereenkomst, GewerktePeriode
   - `onz-fin` (finance): EindSaldo, Grootboekpost, heeftGeldBedrag
   - `onz-zorg` (care): VerpleegProcess, WlzIndicatie
   - `onz-org` (organization): Vestiging, ZorgkantoorRegio

3. **Date parsing with fallback formats**:
   ```python
   for fmt in ("%d/%m/%Y %H:%M:%S", "%d/%m/%Y", "%Y-%m-%d"):
       try: return datetime.strptime(val, fmt).strftime("%Y-%m-%d")
   ```
   - Data comes from AFAS (European format), some spreadsheets (ISO)
   - Automatic detection without configuration
   
4. **Row limits** (MAX_ROWS_PER_TABLE = 500):
   - Dev/demo graphs stay manageable size
   - Can scale by increasing limit or partitioning by time period

#### **Your Role**
- Designed validation rules mapping (CSV column → ontology constraint → SHACL-ready structure)
- **Implemented CSV-to-Turtle conversion using rdflib** (designed data transformation pipeline, proper namespace binding, XSD type mapping)
- Created dashboard UI for multi-stakeholder exploration (technical vs business views)
- Integrated CSV date parsing robustness (handles AFAS European format, ISO, with/without timestamps)

#### **Result & Impact**
- Data stewards can run validation in seconds, see failures before data enters production
- **Technical team gets Turtle (TTL) RDF output** for Linked Data platform ingestion — ready to load into RDF stores like Virtuoso
- Business teams visualize data quality without SQL knowledge
- Example: caught 47 missing mandatory fields in one dataset before handoff (prevented downstream failures)

---

## Part 2: Your Semantic Standards Expertise — XBRL vs RDF/OWL

**Important context:** You have experience with **two different semantic standards**, which is a key differentiator:

### XBRL (eXtensible Business Reporting Language)
**Structure & paradigm:**
- XML-based, hierarchical/document-oriented
- Combines taxonomy (like a chart of accounts) + instance documents (like a financial statement)
- Taxonomies define hierarchies, calculation rules, allowed values
- Used in: regulatory financial reporting, GL taxonomies, insurance filings

**Your experience:**
- Built taxonomies for financial/regulatory reporting
- Understand hierarchical semantic constraints
- Know how to map GL charts to taxonomies, validate instance documents

**When to use XBRL:**
- Standardized financial reporting (GAAP, IFRS, regulatory filings)
- Hierarchical, fixed-structure data (chart of accounts, insurance products)
- Domain where all stakeholders agree on the hierarchy (finance is mature; healthcare less so)

### RDF/OWL (Resource Description Framework / Web Ontology Language)
**Structure & paradigm:**
- Graph-based, network-oriented triples: (subject, predicate, object)
- Ontologies define concepts and relationships; instances are triples
- URIs enable linking across domains without predefined hierarchies
- Used in: linked data, knowledge graphs, cross-system integration, semantic web

**Your experience:**
- KIK-V project uses RDF for healthcare data integration
- Built ontologies with 25 OWL classes, validated instance data
- SPARQL queries across multi-domain data

**When to use RDF:**
- Flexible, multi-domain integration (healthcare, finance, HR mixed)
- Ad-hoc queries across heterogeneous sources
- Need to link to external vocabularies (SNOMED, ICD-10)
- Exploratory data discovery

### Key Differences
| Aspect | XBRL | RDF/OWL |
|--------|------|---------|
| **Format** | XML hierarchies | Triples (URIs + relationships) |
| **Structure** | Tree (fixed hierarchy) | Graph (arbitrary relationships) |
| **Semantics** | Taxonomy + calculation rules | Ontology + inference |
| **Query** | XPath / XQuery | SPARQL |
| **Linking** | Within hierarchy | Cross-domain URIs |
| **Use Case** | Regulatory reporting | Linked data & integration |
| **Standardization Body** | IASB, EIOPA, SEC | W3C |
| **Reasoning** | Limited (formulas) | Rich (OWL inference) |

### How to Position This in the Interview

**If they ask "Tell us about your ontology/taxonomy work":**

> I've worked with two complementary semantic standards, each suited to different problems.
>
> **XBRL (hierarchical/regulatory):** I've built financial taxonomies and instance documents. XBRL is XML-based and hierarchical—think of it like a formal chart of accounts with validation rules. You define a taxonomy (the structure), then create instances (financial statements) that conform to it. The power is in standardization: regulators and auditors know exactly what to expect.
>
> **RDF/OWL (graph-based/linked data):** My KIK-V project uses RDF for healthcare data integration. Here, everything is URIs and relationships. Instead of a fixed hierarchy, you have a graph where a healthcare worker can be linked to employment contracts, care processes, and financial data through shared URIs. This enables flexible, ad-hoc queries across domains.
>
> **When I'd choose each:** XBRL for regulatory compliance where the structure is standardized globally (financial statements follow rules set by IASB). RDF for integration and discovery—when you need to combine data from different sources with different structures, or ask exploratory questions without predefining all relationships.

**If they ask "Why use RDF instead of XBRL for healthcare?":**

> You could build an XBRL taxonomy for healthcare, but you'd lose flexibility. XBRL hierarchies are rigid—you'd need to predefine all dimensions upfront. RDF's URIs let you link to external vocabularies dynamically (like SNOMED CT for clinical codes). SPARQL is better for exploratory queries; XBRL is optimized for fixed-format reporting.
>
> That said, if your use case is **standardized healthcare reporting** (like quality metrics for regulators), XBRL could work well. It depends on flexibility vs. standardization.

**If they ask "Could you bridge XBRL and RDF?":**

> Absolutely—this is what modern systems do. Imagine you need both:
> - **Flexible integration** (RDF): Link healthcare data from multiple sources
> - **Standardized reporting** (XBRL): Generate compliant regulatory reports
>
> You'd use RDF for the operational data model, then generate XBRL instances from RDF for submission. Your XBRL taxonomy acts as a view into the RDF graph, enforcing reporting standards without constraining the underlying integration.

---

## Part 3: Linked Data Knowledge Refresh (RDF/OWL Specifics)

Use these to structure your answers when they ask "Tell me about RDF" or "How would you validate semantic data?"

### RDF (Resource Description Framework)
**Core idea:** Everything is a triple: `(subject, predicate, object)`

**In your work:**
```sparql
onz-zorg:medewerker123 a onz-g:Human .
onz-zorg:medewerker123 rdfs:label "123" .
onz-zorg:medewerker123 onz-g:hasDateOfBirth "1985-03-15"^^xsd:date .
```
- **Subject**: URI like `http://purl.org/ozo/onz-zorg#medewerker123`
- **Predicate**: Property like `http://www.w3.org/2000/01/rdf-schema#label`
- **Object**: Can be URI (linked data) or Literal with XSD type (`"1985-03-15"^^xsd:date`)

**Why you chose it:**
- Represents healthcare relationships naturally: staff member → has → date of birth, contract → part of → organization
- Enables cross-system linking (AFAS data linked to ONS care data via shared individual URIs)
- Flexible: add properties without schema migration

### RDFS (RDF Schema) & OWL (Web Ontology Language)
**RDFS** (basic structure):
- `rdfs:Class` — define a class
- `rdfs:subClassOf` — inheritance
- `rdfs:range`, `rdfs:domain` — property constraints

**OWL** (richer semantics):
- `owl:Class`, `owl:ObjectProperty`, `owl:DatatypeProperty`
- Cardinality: `owl:minCardinality`, `owl:maxCardinality`
- Disjointness: `owl:disjointWith`
- Restrictions: `owl:onProperty`, `owl:someValuesFrom`

**In your work:**
```turtle
onz-fin:EindSaldo a owl:Class ;
    rdfs:label "End Balance"@en ;
    rdfs:comment "Final balance for a cost center at a given date"@en .

onz-fin:heeftGeldBedrag a owl:DatatypeProperty ;
    rdfs:domain onz-fin:EindSaldo ;
    rdfs:range xsd:decimal ;
    rdfs:label "has money amount"@en .
```

**Why it matters:**
- OWL constraints drive validation (mandatory vs optional, data types)
- Ontology versioning: constraints evolve as business rules change
- SPARQL queries rely on these definitions to find relevant data

### SPARQL (Query Language)
**Three main operations:**
1. **SELECT** — retrieve values
2. **CONSTRUCT** — build new RDF from query results
3. **ASK** — boolean check

**Your indicator 18.1 query** (balance sheet):
```sparql
PREFIX onz-fin: <http://purl.org/ozo/onz-fin#>
PREFIX onz-g: <http://purl.org/ozo/onz-g#>
SELECT ?rubriek (SUM(?geld_bedrag) AS ?bedrag)
WHERE {
    ?rubriek a onz-fin:Grootboekrubriek ;
             onz-g:hasDate ?date ;
             onz-fin:heeftGeldBedrag ?geld_bedrag .
    FILTER (?date >= ?startdate && ?date <= ?enddate)
}
GROUP BY ?rubriek
```

**Why you use it:**
- Query semantic data without SQL joins (relationships are explicit in RDF)
- Filter by date, sum balances, group by accounting category
- Can be executed over RDF stores (Virtuoso, AllegroGraph) or in-memory (rdflib)

### Ontologies & Knowledge Graphs
**Ontology** = A formal description of concepts and their relationships
- Your ontology: 25 OWL classes covering healthcare (personnel, care, finance, organization)
- Defines what "a healthcare indicator" looks like semantically
- Enables reasoning: "What's the type of this field?" → traverse ontology to find answer

**Knowledge Graph** = Concrete instances of the ontology (your data)
- Each row in CDM becomes RDF triples
- "medewerker123 is-a Human with dateOfBirth 1985-03-15"
- Linked Data principle: URIs are unique, reusable across datasets

**Why this structure matters:**
- **Semantic interoperability**: Healthcare systems can exchange data using shared URIs
- **Validation**: "Is this value allowed for this property?" → check ontology constraints
- **Discoverability**: "What data do we have about care processes?" → query the graph

---

## Part 3: Data Mapping Examples from Your Work

### RML / YARRRML Alternatives

You haven't used RML/YARRRML explicitly, but you've built your own **declarative mapping approach**:

**The KIK-V approach (cdm-owl-mapping.csv):**
```
Source_Table | Source_Column | Target_Class | Target_Property | XSD_Type | Cardinality | Notes
Medewerker   | MedewerkerId  | onz-g:Human  | onz-g:hasId     | xsd:string | 1..1 | Primary key
Medewerker   | Geboortedatum | onz-g:Human  | onz-g:hasDateOfBirth | xsd:date | 0..1 | Optional
Werkovereenkomst | WerkovereenkomstId | onz-pers:Werkovereenkomst | rdf:type | N/A | 1..1 | Class membership
```

**Compared to RML:**
```turtle
@prefix rml: <http://semweb.mmlab.be/ns/rml#> ;
@prefix rr: <http://www.w3.org/ns/r2rml#> ;

<#MedewerkerMap>
    rml:logicalSource [
        rml:source "Medewerker.csv" ;
        rml:referenceFormulation ql:CSV
    ] ;
    rr:subjectMap [
        rr:template "http://purl.org/ozo/onz-g#Human/{MedewerkerId}"
    ] ;
    rr:predicateObjectMap [
        rr:predicate onz-g:hasDateOfBirth ;
        rr:objectMap [
            rml:reference "Geboortedatum" ;
            rr:datatype xsd:date
        ]
    ] .
```

**Why your approach works:**
- CSV format is governance-friendly (non-technical stakeholders can review)
- Simpler than RML/YARRRML for linear mappings
- Integrates directly into your pipeline (read → validate → generate)

**When you'd consider RML/YARRRML:**
- Complex join conditions (e.g., "link Medewerker to Werkovereenkomst when dates overlap")
- Hierarchical XML/JSON sources
- Need portability to other systems

---

## Part 4: Data Quality & Validation (SHACL & Beyond)

### What is SHACL?
**SHACL** = Shapes Constraint Language. Defines rules that RDF data must satisfy.

**Example SHACL shape** (your data should have):
```turtle
PREFIX sh: <http://www.w3.org/ns/shacl#> ;
PREFIX onz-g: <http://purl.org/ozo/onz-g#> ;

<#HumanShape>
    a sh:NodeShape ;
    sh:targetClass onz-g:Human ;
    sh:property [
        sh:path onz-g:hasDateOfBirth ;
        sh:datatype xsd:date ;
        sh:minCount 1 ;  # mandatory
        sh:maxCount 1 ;  # exactly one
    ] ;
    sh:property [
        sh:path rdf:type ;
        sh:minCount 1 ;
    ] .
```

### Your Validation Approach (CSV-level)

Currently you validate **before** RDF conversion:
```python
class CSVValidator:
    def check_mandatory_fields(self, df, mapping):
        # For each class, check all mandatory properties exist
        # Example: onz-g:Human must have rdfs:label and onz-g:hasDateOfBirth
        
    def check_foreign_keys(self, tables):
        # Medewerker.id must exist in Werkovereenkomst.MedewerkerId
        
    def check_date_sequences(self, df):
        # Werkovereenkomst.start_date <= Werkovereenkomst.end_date
        
    def check_datatypes(self, df, mapping):
        # If XSD type is xsd:date, try to parse as date
```

**Why validate at CSV level first:**
- Catches ~80% of errors before RDF generation
- Errors are easier to understand in tabular form
- Reduces load on RDF validator

### How to Talk About This in the Interview

**"Tell me about your data quality approach:"**

> We use a two-stage validation strategy. First, at the CSV level (before RDF conversion), we validate:
> 1. **Schema conformance** — does every row have the mandatory fields the ontology requires?
> 2. **Referential integrity** — do foreign keys exist in parent tables?
> 3. **Temporal correctness** — do date sequences make sense (start ≤ end)?
> 4. **Type safety** — can we parse values into their declared XSD types?
> 
> We produce a validation report per rule, allowing data stewards to see exactly what failed and where.
> 
> Then, once CSV is clean, we convert to RDF. In future, we could add SHACL validation on the RDF graph itself for more complex constraints (e.g., "if this healthcare worker is a manager, they must supervise at least one person").

---

## Part 5: Practical Engineering Experience

### APIs & Integrations
**AFAS ERP Integration** (`afas_load.py`):
- REST API with pagination (`skip`, `take`)
- Bearer token authentication
- Robust error handling: retry logic, chunk size tuning
- Stores in SQL Server via SQLAlchemy

```python
url = f"https://{klantnummer}.rest.afas.online/ProfitRestServices/connectors/{connector}"
headers = {'Authorization': f'AfasToken {AfasToken}'}
response = requests.get(url, params={'skip': skip, 'take': 1000})
```

**Your choices:**
- Paginated fetching (avoids memory overflow on large datasets)
- Sleeps between requests (respects API rate limits: `time.sleep(0.1)`)
- Batch inserts to database

### Python & RDF Libraries (rdflib)
**Your CSV → Turtle conversion implementation:**
- **rdflib**: Industry-standard Python library for RDF creation and manipulation
- **Practical skills**: Graph construction, namespace binding, literal type casting, serialization to multiple formats
- **What you've built**: 
  - Read CSV data (pandas DataFrames)
  - Map columns to OWL properties using `cdm-owl-mapping.csv`
  - Create URIs safely (sanitize local names to avoid special characters)
  - Construct RDF triples with proper type declarations (`Literal(val, datatype=XSD.date)`)
  - Bind namespaces to graph for Turtle readability
  - Serialize to Turtle format for Linked Data platforms

**Example workflow you've implemented:**
```python
from rdflib import Graph, Literal, Namespace, RDF, RDFS, XSD, URIRef

# Build graph from CSV rows
g = Graph()
g.bind("onz-g", ONZ_G)
g.bind("onz-fin", ONZ_FIN)

for idx, row in df.iterrows():
    # Create individual URI
    uri = URIRef(f"{ONZ_ZORG}medewerker{safe_local(row['key'])}")
    
    # Add type
    g.add((uri, RDF.type, ONZ_G.Human))
    
    # Add properties with proper type casting
    if row['geboortedatum']:
        g.add((uri, ONZ_G.hasDateOfBirth, Literal(date_val, datatype=XSD.date)))

# Output Turtle
g.serialize("output.ttl", format="turtle")
```

**Why this matters for the interview:**
- Shows you can **actually build** semantic systems, not just design them
- rdflib is the de facto Python standard for RDF work
- Demonstrates understanding of both ontology (conceptual) and implementation (practical) layers
- You can discuss trade-offs: in-memory vs. streaming, Turtle vs. other serializations, validation pre/post conversion

### Git & Version Control
**Your workspace structure suggests:**
- Multiple pipeline versions (`owl-pipeline-versions/runs/`)
- Semantic versioning for ontology (1.3.2 → 1.3.4 → 2.0.0)
- Separate branches per data source (AFAS, ONS, SDB)

**How you'd describe this:**
> We use git to track ontology versions and pipeline code. Each data source (AFAS, care system, finance system) has its own extraction pipeline. We version the ontology semantically (major.minor.patch), so when business rules change, we can:
> 1. Create new ontology version
> 2. Run the pipeline with old + new versions in parallel
> 3. Validate both outputs
> 4. Merge when confident

### CI/CD & Automation
*Not explicitly visible in your code, but you should prepare to discuss:*

**How you'd set up CI/CD for this work:**
```
On every git push to main:
  1. Run step1, step2, step3 of owl-pipeline
  2. Run validation against sample CSVs
  3. Generate RDF output
  4. Compare with previous run (schema drift detection)
  5. If all green: deploy to staging environment
  6. Generate validation report for data stewards
```

Tools you might use: Azure Pipelines (aligns with KIK-V's ecosystem), GitHub Actions, or Jenkins.

### Documentation
**Your current documentation:**
- `README.md` files in each folder (purpose, inputs, outputs)
- Inline comments in Python code (e.g., ` # Extract indicator ID from YAML frontmatter`)
- Markdown indicator files (1.1.md, 18.1.md) with SPARQL queries

**What to emphasize:**
- Every DDL column traces back to a design decision document
- Data lineage is explicit (CDM → OWL → RDF → Dashboard)
- Non-technical stakeholders (data stewards) can understand the flow

### Working with Stakeholders
**In your projects, you work with:**
- **Data stewards** (validate CDM quality, review mappings)
- **Business analysts** (write indicator requirements in markdown)
- **Finance/HR teams** (consume reports, flag validation errors)
- **Tech leads** (review ontology versioning, API integration)
- **Care coordinators** (use dashboard to explore quality issues)

**How to position yourself:**
> We translate between technical constraints (XSD types, OWL cardinality) and business language ("Why can't this field be null?"). I've learned to involve stakeholders early—showing them the mapping CSV, the validation dashboard, getting their feedback before finalizing a pipeline step.

---

## Part 6: Questions to Ask the Team

### About Their Current State
1. **"Are you still building the foundation of your semantic platform, or improving existing capabilities?"**
   - If early stage → opportunity to help design architecture
   - If mature → opportunity to solve scaling/governance challenges

2. **"What's your current maturity with linked data? Are you already using RDF stores (Virtuoso, AllegroGraph), or is SPARQL something you want to introduce?"**
   - Shows you understand the progression (CSV → RDF → federated queries)

3. **"How do you currently handle ontology evolution when business rules change?"**
   - Reveals whether they have versioning strategy; hints at pain points you could solve

4. **"What's your data quality bottleneck right now—is it validation, reconciliation between systems, or governance?"**
   - Lets you assess where you'd add most value

### About the Role
5. **"What would success look like for this role in the first 6 months?"**
   - Clarifies expectations; shows you're thinking about impact

6. **"Who would I be collaborating with most closely—data stewards, ontology engineers, backend developers?"**
   - Helps you understand team dynamics

7. **"Are there existing projects I'd inherit, or am I building greenfield systems?"**
   - Prepares you mentally for maintenance vs. innovation ratio

### About Technical Challenges
8. **"What's the biggest challenge you currently face with mapping relational data to your ontology?"**
   - Shows genuine interest; opens door to discuss your cdm_generate.py approach

9. **"If you could redesign one part of your data pipeline, what would it be?"**
   - Reveals pain points and innovation opportunities

10. **"How do you currently handle validation across the full pipeline (from source → staging → CDM → RDF)?"**
    - Lets you position your validation dashboard work

### If You Don't Know Something

**Honest answer template:**
> I haven't worked directly with [tool/concept] yet, but I understand the principles—[explain what you know]. I'd be interested to learn how you use it here and how it fits into your architecture.

**Examples:**
- "I haven't used Virtuoso for RDF querying yet, but I understand it's an in-memory semantic database with SPARQL endpoints. I've implemented CSV-to-RDF conversion using rdflib in Python—built the namespace binding, type casting, and Turtle serialization from scratch. I'd love to see how you'd scale that to larger graphs with Virtuoso."
- "I haven't implemented SHACL validation on RDF directly, but I've built equivalent checks at the CSV level before converting to RDF with rdflib. I'd be curious to learn how you'd layer SHACL on top for more complex constraints."

---

## Part 7: Quick Reference — How to Tell These Stories

### Story 1: OWL Pipeline (5–7 min)
**Opening:** "We had 24+ business indicators defined in markdown, but they needed to be validated against an OWL ontology..."
- Problem: Link markdown to semantics
- Solution: 3-stage pipeline (extract → constrain → export)
- Technical detail: Regex parsing + URI namespace handling
- Result: Single source of truth, automated constraint propagation

### Story 2: CDM Mapping (5–7 min)
**Opening:** "We needed to keep our relational CDM aligned with an evolving OWL ontology without manual remapping..."
- Problem: Schema drift between ontology and database
- Solution: Ontology-driven DDL generation + versioning
- Technical detail: XSD → SQL type mapping, semantic versioning
- Result: Bidirectional traceability, version coverage matrix

### Story 3: Validation Dashboard (5–7 min)
**Opening:** "We had CSV data that needed to be validated against semantic constraints, then converted to RDF for our linked data platform..."
- Problem: Multi-stakeholder data quality (technical vs. business views)
- Solution: Two-stage validation + Streamlit dashboard + rdflib RDF builder
- Technical detail: Row-level validation rules, proper namespace management
- Result: Data stewards catch issues before production, RDF output ready for graph store

---

## Checklist Before the Interview

- [ ] **Re-read your three project stories** and practice the 5–7 min version without notes
- [ ] **Refresh SPARQL syntax** — scan your indicator 18.1 query, understand the WHERE clause
- [ ] **Prepare XSD types** — memorize 5 common ones (string, date, decimal, boolean, integer)
- [ ] **Review namespace prefixes** — onz-g (general), onz-pers (personnel), onz-fin (finance), onz-zorg (care)
- [ ] **Prepare 1–2 questions from Part 6** that genuinely interest you
- [ ] **Think about failure/challenge** — prepare to answer "Tell me about a time you had to debug a complex mapping issue"
  - Example: "We had date parsing failures from AFAS using European format; I built a fallback parser that tries multiple formats, and added tests for edge cases."
- [ ] **Bring your laptop** — offer to show code if the conversation goes technical
- [ ] **Be honest about gaps** — if they ask about SHACL or RML and you haven't used it, say so clearly and show you understand the concepts

---

## Part 8: Validation Dashboard Interview Questions & Answers

When you show your Streamlit dashboard live, expect questions like these:

### **Architecture & Design**

**Q1: "Why did you choose a 3-layer architecture (Raw → Stage → DWH) instead of direct CSV → OWL mapping?"**

**A:** Separation of concerns. If I go direct, every error in CSV mapping breaks the entire pipeline. With three layers:
- **Raw → Stage**: Cleans data, handles date parsing variations (AFAS uses DD/MM/YYYY, Excel uses ISO, some dates have timestamps). Catches ~30% of errors here.
- **Stage → DWH**: Applies business logic (calculate FTE from hours, validate contract overlaps). This is where I catch referential integrity issues.
- **DWH → RDF**: Now the data is clean. I can map to OWL with confidence.

**Trade-offs:** More storage, more ETL steps. **Payoff:** Each layer is independently testable and reusable. If the DWH layer fails, I don't need to re-run the entire pipeline—just fix that layer and retry.

---

**Q2: "How do you handle schema evolution when the ontology changes? For example, a new OWL class is added."**

**A:** We version both the ontology and the mappings. When the ontology changes:
1. Create new ontology version (e.g., v1.3.4 → v1.3.5)
2. Update `cdm-owl-mapping.csv` with new class mappings
3. Run the pipeline with *both* old and new versions in parallel
4. Compare validation results—if the new version passes the same data, we can upgrade
5. Publish new version to staging; get stakeholder approval before production

**Implementation detail**: Our code reads `runs/{version}/ontology.ttl`, so different versions run independently. This is crucial for healthcare—if rules change, you can't just flip a switch; you need to validate that historical data still conforms.

---

**Q3: "Walk me through the data lineage: how does a field in CSV trace to OWL and then to a SQL column?"**

**A:** Every CSV field has a row in `cdm-owl-mapping.csv`:
```
CDM_TABLE=Medewerker | CDM_FIELD=geboortedatum | CLASS=onz-g:Human | PROPERTY=onz-g:hasDateOfBirth | XSD_TYPE=xsd:date
```

When I load a CSV row:
1. Read `Medewerker.geboortedatum` = "15/03/1985"
2. Parse using format detection (DD/MM/YYYY → YYYY-MM-DD)
3. Validate: Is this a valid date? Is it in the future (error)? → produces pass/fail
4. Map to RDF triple: `onz-zorg:medewerker123 onz-g:hasDateOfBirth "1985-03-15"^^xsd:date`
5. In DWH SQL Server, the column `Medewerker.geboortedatum` has a NOT NULL DATE constraint (derived from ontology cardinality)

**If something fails in production**, I can trace it:
- "Why did 500 records fail validation?" → Check `cdm_owl_mapping.csv` for that field's constraints → Check the ontology definition → Understand the business rule → Fix the data or adjust the rule

---

### **Semantic Web & Ontology Knowledge**

**Q4: "Why use OWL at all? Why not just document the CDM schema in a spreadsheet?"**

**A:** A spreadsheet is prose. OWL is machine-readable.

**With OWL:**
- I can write SPARQL queries: "What properties must every `onz-g:Human` have?"
- I can infer: "If class B inherits from class A, then B must also satisfy A's constraints"
- I can validate automatically: "Does this record have all mandatory properties?"
- I can generate artifacts: "Create a SQL Server DDL script from the ontology"

**With a spreadsheet:**
- I need to manually update the schema every time business rules change
- No way to detect inconsistencies (e.g., "I documented that Medewerker needs a birth date, but nobody validated this")
- Manual reconciliation between documentation and implementation

**Business impact**: OWL enables **single source of truth**. When a business analyst changes an indicator requirement, the ontology changes → validation rules update → CDM schema updates → all downstream systems automatically use the new rule.

---

**Q5: "How do you resolve conflicts when you have 4–5 versions of the ontology in parallel?"**

**A:** We maintain a version matrix. Each profile (ZK vs IGJ) may need different constraints:

| Class | v1.3.2 | v1.3.4 | v2.0.0 |
|-------|--------|--------|--------|
| onz-g:Human | Required: birth date | Required: birth date | Required: birth date + nationality |
| onz-pers:Werkovereenkomst | 1..1 contract per employee | 0..* contracts (to allow multi-hires) | 0..* contracts |

When merging:
- **Conservative approach**: Validation passes v1.3.2 BUT also passes all older versions. This ensures backward compatibility.
- **Aggressive approach**: Only validate against the latest version. Faster, but requires coordinated rollout.

We use **conservative** in healthcare because legacy data must stay valid.

---

**Q6: "What's the relationship between an OWL class and a CDM table? Are they 1:1?"**

**A:** No, often many-to-many:
- **One table → multiple classes**: CDM table `Medewerker` maps to `onz-g:Human` (general), `onz-pers:ZorgverlenerFunctie` (if they're a caregiver), `onz-org:Vestiging` (if they're a location manager). A person can play multiple roles.
- **Multiple tables → one class**: The OWL class `onz-pers:Werkovereenkomst` (employment contract) might come from multiple tables in CDM: `Werkovereenkomst` (base), `WerkovereenkomstAfspraak` (amendments), `ContractOmvang` (hours). We denormalize these in RDF.

**This is the key insight of linked data**: Don't force a 1:1 mapping. Let the graph structure represent reality. A person is both a Human and a Caregiver; a contract is both a Werkovereenkomst and a set of Afspraken. Each role has different properties.

---

**Q7: "How do you handle classes that don't have corresponding CSV data?"**

**A:** Flag them as warnings, not errors.

**Example**: The OWL ontology defines `onz-zorg:CarePlan`, but our current CDM doesn't extract care plans from the source system. In validation:
- Don't fail the entire indicator
- Log: "Warning: onz-zorg:CarePlan has no data source"
- Stakeholders see this in the dashboard and decide: "Do we need to add CarePlan extraction, or is the indicator still valid without it?"

**Why this matters**: In healthcare, you often have aspirational ontologies (e.g., "Ideally we'd track care plans") that don't match current systems. OWL lets you document the ideal state while validating against the real state.

---

### **Data Quality & Validation**

**Q8: "What happens when a record fails validation? Do you reject it, flag it, or fix it automatically?"**

**A:** We have three severity levels:
1. **Mandatory** (hard error): Reject the record. Example: missing employee ID.
2. **Datatype** (soft error): Flag but allow. Example: salary is text "high" instead of a number. Data steward sees it in the dashboard, investigates if it's a parsing error or bad source data.
3. **Warning** (advisory): Informational only. Example: contract overlaps detected (employee has two contracts simultaneously). Might be intentional (part-time + extra shift), might be a data error.

Dashboard separates these visually. The data steward decides what to do based on severity and pattern.

---

**Q9: "Your dashboard shows 95% pass rate. What's your plan for the 5% failures?"**

**A:** Depends on failure type:
- **Parsing errors** (15% of failures): Date format "01-01-2023" can't parse → build better parser, add test case
- **Referential errors** (70% of failures): Person ID 12345 in Werkovereenkomst doesn't exist in Medewerker → upstream data quality issue, escalate to source system owner
- **Business rule violations** (15% of failures): Contract end date before start date → data entry error, data steward corrects in source system

**Process**: 
1. Triage failures by type (dashboard has a "top failures" summary)
2. Assign owners (parser dev, data steward, source system team)
3. Fix + rerun validation
4. Track trend: "Are we improving?" (Yes = success; No = root cause analysis needed)

**Target**: 99%+ pass rate for production data. 95% in dev is normal.

---

**Q10: "How do you avoid false positives in validation? What if your rule is too strict?"**

**A:** Rules are written in collaboration with stakeholders.

**Example workflow**:
1. Ontology says: "Birth date is mandatory for onz-g:Human"
2. We run validation, find 50 records with missing birth dates
3. Data steward says: "Those 50 are non-human individuals (organizations, departments). They shouldn't have birth dates."
4. We refine the rule: "Birth date is mandatory only if Human.type = 'Person', not if type = 'Organization'"
5. Rerun validation → 50 false positives gone

**Prevention**: 
- Validation rules are versioned alongside the ontology
- Stakeholders review rules before deployment
- Test suite with known-good and known-bad data
- Dashboard shows which rules have high failure rates (suspicious if 30% of records fail a rule)

---

**Q11: "How do you prioritize which validation checks to run first if you have limited compute?"**

**A:** Pyramid approach:
1. **Foundation checks** (fast, high impact): Mandatory fields present, datatypes correct, file integrity → runs in seconds
2. **Relationship checks** (medium): Foreign keys exist, unique constraints → runs in minutes
3. **Business logic** (slow, lower volume): Date sequences, contract overlaps, balance sheet totals → runs in 30+ minutes

Run foundation checks first. If they pass, run relationship checks. Only if both pass, run business logic. This way:
- Quick feedback for obvious errors (missing data)
- Don't waste compute on complex checks if basic checks fail
- Parallelize where possible (different CDM tables can be checked in parallel)

---

### **Implementation & Technical Details**

**Q12: "How did you extract OWL constraints from the RDF/TTL files? What challenges did you face?"**

**A:** Using rdflib in Python:
```python
from rdflib import Graph, RDF, RDFS, OWL, Namespace

g = Graph()
g.parse("ontology.ttl", format="turtle")

# Query for all classes and their constraints
for cls_uri in g.subjects(RDF.type, OWL.Class):
    cls_name = str(cls_uri).split("#")[-1]  # Extract local name
    
    # Get properties with domain=this class
    for prop_uri in g.subjects(RDFS.domain, cls_uri):
        prop_name = str(prop_uri).split("#")[-1]
        range_val = g.value(prop_uri, RDFS.range)
        cardinality = g.value(prop_uri, OWL.minCardinality)
        
        # Extract into CSV row
        print(f"{cls_name}, {prop_name}, {range_val}, {cardinality}")
```

**Challenges faced**:
1. **Namespace handling**: Same class defined in multiple namespaces (e.g., `onz-pers:Human` vs `onz-g:Human`). Solution: Standardize on a primary namespace in the ontology; mark alternates as aliases.
2. **Cardinality representation**: OWL uses different patterns (some use `minCardinality`, some use `Restriction` + `onProperty`). Solution: Normalize during extraction.
3. **Inheritance chains**: A property is defined on a parent class; child class inherits it. Solution: Recursively walk the inheritance tree and aggregate constraints.
4. **Multiple file versions**: Ontology spread across multiple TTL files. Solution: Load all into a single graph before querying.

---

**Q13: "Your pipeline runs in Python. Did you consider using a semantic web framework like RDFlib, or would you use something else?"**

**A:** I *am* using rdflib for the CSV → RDF conversion! It's the standard library for RDF in Python.

**Why rdflib**:
- Handles URI escaping and namespace binding automatically (avoids errors)
- Built-in support for XSD types (Literal with datatype)
- Multiple output formats (Turtle, RDF/XML, N-Triples)
- Well-documented and maintained

**Alternatives I considered**:
- **RML/YARRRML**: More declarative; useful for complex joins. But for linear CSV mappings, rdflib + Python is simpler.
- **SPARQL/raw templates**: I could write RDF triples as strings, but that's error-prone (special characters, namespace URIs).
- **Java/Jena**: More robust for large graphs. But Python is better for data engineering workflows (pandas integration, Jupyter notebooks).

**If we scale to billions of triples**, I'd move to a semantic database (Virtuoso, AllegroGraph) and SPARQL endpoints instead of in-memory graphs. But for current volumes (thousands of records, millions of triples), rdflib is perfect.

---

**Q14: "Walk me through the CSV → RDF conversion. How do you generate URIs?"**

**A:** URI minting is critical—URIs must be unique, stable, and ideally dereferenceable.

**Our approach**:
```python
def mint_uri(table_name, primary_key):
    # Example: Medewerker.MedewerkerId = "123" → onz-zorg:medewerker123
    local_name = f"{table_name.lower()}{safe_local(primary_key)}"
    return URIRef(f"{ONZ_ZORG}{local_name}")

def safe_local(s):
    # Remove special characters that break URIs
    s = re.sub(r'[^a-zA-Z0-9_-]', '', str(s))
    return s[:50]  # Limit to 50 chars

# In CSV conversion loop:
for idx, row in df.iterrows():
    uri = mint_uri("Medewerker", row["MedewerkerId"])  # onz-zorg:medewerker123
    g.add((uri, RDF.type, ONZ_G.Human))
    g.add((uri, RDFS.label, Literal(row["Voornaam"] + " " + row["Achternaam"])))
```

**Why this matters**:
- **Uniqueness**: Same primary key always generates same URI. If we process the same record twice, we get the same triples (no duplicates).
- **Stability**: If the record is updated, the URI stays the same. RDF graphs can be merged without collision.
- **Linking**: If two systems both know about person 123, they can link using the same URI.

**Challenge**: What if primary key changes (e.g., employee renumbered from 123 to 456)? Solution: Track old IDs as alternate identifiers in RDF: `onz-zorg:medewerker456 owl:sameAs onz-zorg:medewerker123`

---

**Q15: "What's the performance impact of validating 10,000 records against 164 rules?"**

**A:** Roughly 2–5 seconds on a laptop for full validation. Here's the breakdown:

- **Mandatory field check**: 0.3 sec (scan each record once)
- **Datatype checks**: 1.2 sec (parse dates, numbers—slowest part)
- **Foreign key checks**: 0.8 sec (in-memory hash join)
- **Business logic** (date overlaps, balance checks): 2.0 sec (requires comparisons)
- **Dashboard rendering**: 1.5 sec (aggregate into charts)

**Total**: ~5 seconds for 10k records × 164 rules = ~1.6M validation operations.

**Optimization**:
- Pre-filter to subset of rules (e.g., only check rules for this indicator's classes) → 2–3 rules at a time instead of 164
- Batch validation in chunks (1000 records at a time) → better caching
- Parallelize by table (Medewerker validation independent of Werkovereenkomst) → 3–4x speedup

**Scaling**: At 1M records, we'd need to move to a dedicated validation engine (Apache Spark, DuckDB for SQL-based checks, or a proper semantic database). But for current volumes, Python is fine.

---

### **Analysis & Visualization**

**Q16: "Why show 'Rules per Class' distribution? What insight does that give you?"**

**A:** Imbalance suggests weakness or over-engineering.

**Example from your dashboard**:
- `onz-pers:Werkovereenkomst` has 12 rules (heavily constrained)
- `onz-pers:Medewerker` has 3 rules (loosely constrained)

**Interpretation**:
- If Werkovereenkomst is actually more complex than Medewerker (yes—it has dates, FTE, department, etc.), 12 vs 3 makes sense
- If Medewerker should also be complex (e.g., must have nationality, date of birth, etc.), then 3 rules is a **gap**. We're under-validating.

**Action**: Talk to stakeholders: "Medewerker only has 3 rules. Should we add constraints for nationality, medical certifications, insurance?"

---

**Q17: "How do you determine if a validation fail rate of 5% is 'good' or 'bad'?"**

**A:** Depends on failure type and source:

- **Dev data**: 5% fail rate is normal. You're cleaning data.
- **Staging data**: 5% is concerning. Should be <1%.
- **Production data**: <0.1% expected. If >0.5%, investigate.

**Context matters**:
- If 95% of failures are "missing optional field" (warning level), that's low risk.
- If 95% of failures are "invalid foreign key" (critical), that's high risk.

**Trend analysis**:
- Is the pass rate improving week-to-week? Good trend.
- Stuck at 95% for 3 months? Problem: either data quality has plateaued, or validation rules are too strict.

**My dashboard approach**: Show pass rate *by rule type*, not just aggregate. This tells the real story:
- Mandatory fields: 99% pass (good)
- Datatype: 90% pass (check parsing)
- FK constraints: 85% pass (data quality issue in source)

---

**Q18: "In the 'Workforce & Care' dashboard, you show active contracts and FTE per Vestiging. How did you compute 'active'?"**

**A:** Active at a reference date (usually today or quarter end):
```python
# Contract is active if:
active = (contract.start_date <= reference_date) AND 
         (contract.end_date IS NULL OR contract.end_date >= reference_date)

# FTE calculation (example):
# If contract says 36 hours/week, that's 1.0 FTE
# If part-time 18 hours/week, that's 0.5 FTE
fte_per_contract = hours_per_week / 36.0

# Total FTE per Vestiging:
total_fte = SUM(fte_per_contract) for all active contracts at this location
```

**Why this matters**:
- Staffing dashboards need a single definition of "active" for all users
- If each team computes differently, numbers don't reconcile
- By anchoring to the ontology (contract properties) and a reference date, everyone gets the same answer

---

### **Business & Impact**

**Q19: "Who are the end users of this dashboard? Are they technical or business analysts?"**

**A:** Primarily **data stewards** and **care coordinators** (business users), with secondary use by **technical teams**.

**Data Stewards** (primary users):
- Non-technical, understand data structure
- Use the dashboard to see validation errors and fix upstream
- They need: Clear visualizations, English explanations, drill-down to problem records
- They *don't* need: SPARQL queries, technical jargon

**Care Coordinators** (secondary users):
- Care delivery staff; use "Workforce & Care" tab to see staffing levels
- They care about: "How many nurses are active today? Which Vestigingen are understaffed?"

**Technical teams** (tertiary users):
- Use "OWL Rules" and "Field Mapping" tabs to debug mappings
- Run validation programmatically via API (not just dashboard)

**Design decisions reflecting this**:
- Color coding (red/green) instead of detailed metrics
- Sidebar filters for quick exploration
- Tooltips explaining what each metric means
- Export buttons (CSV, PDF) for reporting

---

**Q20: "How does this project reduce manual work? Can you quantify the ROI?"**

**A:** Before this project, data validation was **manual spreadsheet reconciliation**:

**Before**:
- Every month: Export CDM to Excel, manually check values against business rules
- Spot-check 1% of records (sampling)
- Time: 8–10 hours per person
- Errors: ~20–30 caught manually, probably ~20–30 missed

**After** (with dashboard):
- Automated validation: 100% of records checked against all rules
- Time: 1 hour per person (just reviewing the dashboard, triage failures)
- Errors: ~100 caught by automation before reaching stakeholders
- False positives: ~5 per month (quickly filtered by data stewards)

**ROI**:
- **Time saved**: (10 – 1) hours/month × 12 months = 108 hours/year = ~2.5 weeks
- **Quality improvement**: 80–90% reduction in escaped defects (errors that reach downstream users)
- **Stakeholder confidence**: Care coordinators trust the data quality; fewer manual "spot checks"

**Cost**: ~40 hours to build (amortized over 2 years = ~3 hours/month cost). **Payback: 3:1 (time saved vs. time invested)**.

---

**Q21: "What was the biggest challenge building this, and how did you solve it?"**

**A:** Choosing the right validation strategy was the hardest part.

**Challenge**: We had 164 OWL rules and 10,000 CSV records. Options:
1. Validate everything at the CSV level (slow, complex parsing)
2. Convert to RDF first, then validate with SHACL (easier ontology-to-validation mapping, but slower)
3. Hybrid: CSV validation for schema, RDF validation for semantics

**I chose option 3**, but it took iteration:
- **First attempt**: Only CSV validation. But it missed semantic errors. Example: "Contract must have at least one linked work agreement." This requires joining two tables—hard to check in a single row.
- **Solution**: Two-stage validation.
  - Stage 1 (CSV): Fast checks (mandatory fields, datatypes, FK existence)
  - Stage 2 (RDF): Slower checks (semantic constraints like "Contract must have at least one Afspraak")

**Result**: Validation completes in 5 seconds for full dataset, catches ~95% of errors in Stage 1, remaining 5% in Stage 2.

**What I learned**: Don't try to solve everything in one layer. Split validation by complexity/speed trade-off. Stakeholders see quick results (Stage 1 in 1 second), then deeper results (Stage 2 in 4 more seconds).

---

### **Gotcha Questions (Be Ready)**

**Q22: "What if a rule violation has multiple root causes? How do you diagnose it?"**

**A:** Violations table shows the actual field value, the constraint, and the error message.

**Example**:
```
Rule: Birth date must be in past
Record: Person 12345, birth_date = "2025-03-15"
Constraint: hasDateOfBirth <= today()
Error: Date is in future
```

Diagnosis:
- Is this a data entry error? (2025 should be 1985) → Operator fat-finger
- Is this a parsing error? (Date format misunderstood) → Fix parser
- Is this intentional? (Recording expected hire date as birth date) → Restructure data model

We also show the **upstream audit trail** (where did this record come from?) so you can trace back to the source system.

---

**Q23: "How do you handle personal data / GDPR compliance in your validation logs?"**

**A:** Good catch! We anonymize row IDs in public reports:

**Internal validation output**:
```
rule_number | person_id | violation
123         | 12345     | birth_date is null
```

**Dashboard export** (shared with data stewards):
```
rule_number | row_hash        | violation
123         | abc123def456    | birth_date is null
```

The hash is deterministic (so the same person always gets the same hash) but non-reversible (can't recover person_id from hash).

**Access control**:
- Only data stewards see full person IDs (they need it to fix records)
- Aggregate reports (to management) don't include identifiers at all
- Audit logs track who viewed violation details and when

---

**Q24: "If the Streamlit dashboard crashes during an interview demo, what's your fallback?"**

**A:** No panic! I have backups:

**Fallback 1** (30 seconds): Restart the dashboard. Streamlit recovers quickly.

**Fallback 2** (1 minute): Run the validation script directly in Python, print results to console:
```bash
python validate_csv.py --input data/Medewerker.csv --rules cdm-owl-mapping.csv
# Output: Validation complete. 500 records checked. 25 failures.
```

**Fallback 3** (2 minutes): Show the pre-generated reports (JSON, CSV files in the `output/` folder). They're static, guaranteed to work.

**Fallback 4** (5 minutes): Open the Jupyter notebook that generated the dashboards. Live run a cell:
```python
df_validation = pd.read_csv("validation_results.csv")
df_validation.groupby("kind")["fail"].sum().plot(kind="bar")
```

So even if Streamlit fails, I can demonstrate the same insights three other ways.

---

## Final Checklist Before the Interview (Updated)

- [ ] **Practice the 3 project stories** — know them cold (5–7 min each without notes)
- [ ] **Memorize 5 key metrics**: 64 indicators, 25 OWL classes, 164 rules, 95% pass rate, 5-second validation time
- [ ] **Prepare examples for Q1–Q7** (architecture/design questions)—practice the explanation until it flows
- [ ] **Review Q9–Q10** (data quality trade-offs)—stakeholders love this discussion
- [ ] **Memorize the ROI story** (Q20): 108 hours/year saved, 3:1 payback
- [ ] **For Q13–Q15**, prepare code snippets—be ready to discuss rdflib, URI minting, performance
- [ ] **Have fallback demos ready** — practice stopping/restarting the Streamlit app
- [ ] **Bring a printed architecture diagram** (optional but impressive)
- [ ] **Be honest about gaps** — if they ask about SHACL validation on RDF, say you've done CSV-level validation and understand SHACL concepts
- [ ] **Practice your "hardest challenge" story** (Q21) — have a 2-min version ready

---

## Final Thoughts

You have **real, substantive work** to talk about. The KIK-V projects show:
- **Problem-solving**: You take ambiguous requirements (indicators in markdown) and turn them into formal semantic systems
- **Technical depth**: You understand RDF, OWL, SPARQL, data validation, API integration, type systems
- **Stakeholder skills**: You build dashboards that both data stewards and architects can use
- **Engineering practices**: You version, document, integrate multiple data sources, validate quality

Go in with genuine curiosity about their challenges. Ask good questions. If you don't know something, admit it—but frame it as "I haven't worked with X yet, but I understand why it's important."

**Most importantly**: Your Streamlit dashboard is a perfect interview tool. It's live, visual, and it tells a complete story from data validation to business insights. Let it do the talking.

Good luck! 🎯
