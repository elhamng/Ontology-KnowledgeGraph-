# My Learning Journey & Knowledge Repository

> A personal documentation of my study, learning, and the books that shaped my understanding.

---

## 📚 About This Repository

This is my **knowledge base**—a living document of:
- **What I've studied** and continue to learn
- **Books that influenced my thinking**
- **Technical domains** I specialize in
- **Key concepts and insights** from my learning

Last updated: **September 2026**

---

## 🎓 Core Knowledge Areas

### 1. **Semantic Web & Linked Data**
*Building systems where data understands itself*

**Topics I've Mastered**:
- RDF (Resource Description Framework) — triples, URIs, graphs
- OWL (Web Ontology Language) — classes, properties, constraints
- SPARQL — querying semantic data
- Ontology design and versioning
- Bridging relational and semantic worlds

**Why It Matters**: 
One semantic language enables data integration across systems without custom mapping code. Critical for healthcare where multiple sources (HR, care, finance) need to communicate.

**Key Insight**: 
OWL is machine-readable. SKOS/Dublin Core are too loose for regulated domains. Use OWL when you need formal constraints.

---

### 2. **Healthcare Data Systems**
*Understanding data in care delivery, finance, and workforce management*

**Domains I Work In**:
- Healthcare data warehousing (Common Data Model / CDM)
- Financial data (balance sheets, GL accounts, XBRL)
- Personnel management (contracts, schedules, certifications)
- Care processes (patient journeys, clinical coding)
- Workforce analytics (FTE, staffing levels, sick leave)

**Real Systems I've Built**:
- 13-table CDM covering personnel, finance, and care
- 25 OWL classes describing healthcare concepts
- 64 business indicators with automated validation
- Multi-stakeholder dashboards for care coordinators, finance teams, data stewards

**Key Insight**:
Healthcare data quality isn't technical—it's organizational. You need alignment between OWL (ontology), SQL (schema), and business users.

---

### 3. **Data Validation & Quality**
*Making data quality visible and measurable*

**Techniques I've Implemented**:
- Two-stage validation (CSV schema + RDF semantic)
- 164 automated validation rules covering:
  - Mandatory fields, datatypes, referential integrity
  - Date sequences, cardinality, business logic
- Automated detection of 80% of real-world errors in CSV stage
- Semantic validation for remaining 20% in RDF stage

**Metrics I Track**:
- Pass rate (goal: 95%+)
- Error types (mandatory, datatype, FK, business logic)
- Error escape rate (bad data reaching production)
- Time saved by automation vs manual reconciliation

**Key Insight**:
CSV validation catches 80% of errors in 5 seconds. Don't over-engineer RDF validation for simple schema checks. Use RDF for complex business logic.

---

### 4. **Python Data Engineering**
*Building ETL pipelines and automation*

**Libraries & Tools**:
- **rdflib** — RDF/OWL manipulation (my go-to for CSV→RDF conversion)
- **pandas** — data wrangling, validation, analysis
- **streamlit** — interactive dashboards for non-technical stakeholders
- **plotly** — business intelligence visualizations
- **SQLAlchemy** — database abstraction layer

**Pipelines I've Built**:
- Markdown indicators → OWL constraints → SQL DDL (3-stage automation)
- CSV → RDF Turtle (proper URI generation, XSD type casting)
- Data validation pipeline (mandatory checks → semantic checks)
- Dashboard backend (caching, filtering, aggregations)

**Key Insight**:
Python is perfect for semantic data work because of rdflib and jupyter notebooks. Use it for data transformation; use databases for scale.

---

### 5. **Streamlit Dashboards & Data Visualization**
*Translating data insights for non-technical audiences*

**What I've Built**:
- **Data quality dashboard** — 95% pass rate, failure patterns, drill-down
- **Workforce & care dashboard** — staffing levels, FTE per location, sick leave trends
- **Financial dashboard** — balance sheets, cost allocations, variances
- **Analysis views** — rule distribution, data completeness, trends

**Key Features**:
- Multi-tab interface (concepts, rules, validation, field mapping, analysis)
- Real-time metrics (KPI cards with pass/fail rates)
- Interactive filters (by class, rule type, check kind)
- Color coding for quick insights (red=fail, green=pass)

**Key Insight**:
Dashboard speed matters more than perfection. Load in < 5 seconds or users abandon it. Cache aggressively.

---

## 📖 Books That Shaped My Thinking

### **Semantic Web & Data**

1. **"Semantic Web for the Working Ontologist" — Allemang & Hendler**
   - *Why I read it*: Deep dive into OWL design patterns
   - *Key takeaway*: Ontologies are about agreed-upon structure, not perfect knowledge representation
   - *Relevance*: Foundation for designing healthcare ontologies

2. **"Learning SPARQL" — Bob DuCharme**
   - *Why I read it*: Practical guide to querying RDF
   - *Key takeaway*: SPARQL is more intuitive than SQL joins for graph data
   - *Relevance*: Enabled me to write efficient queries over 10k+ triples

3. **"Linked Data: Evolving the Web into a Global Data Space" — Heath & Bizer**
   - *Why I read it*: Understanding the principles behind data integration
   - *Key takeaway*: URIs are the key to decentralized data integration
   - *Relevance*: Why we use namespaces in our healthcare ontology

### **Data Engineering & Pipelines**

4. **"Fundamentals of Data Engineering" — Joe Reis & Matt Housley**
   - *Why I read it*: Modern data stack architecture
   - *Key takeaway*: Good orchestration > perfect pipelines
   - *Relevance*: Shaped how I think about validation stages and data flows

5. **"The Data Warehouse Toolkit" — Ralph Kimball**
   - *Why I read it*: Star schema design for healthcare
   - *Key takeaway*: Dimensional modeling makes business queries fast
   - *Relevance*: Our CDM uses Kimball principles (dimensions + facts)

### **Healthcare & Domain Knowledge**

6. **"Electronic Health Records: A Practical Guide" — Oemig & Snelick**
   - *Why I read it*: Understanding clinical data standards
   - *Key takeaway*: Healthcare data is messy by nature (legacy systems, manual entry)
   - *Relevance*: Why robust validation and multiple data sources matter

7. **"Health Information Exchanges and Medical Records Management" — various**
   - *Why I read it*: FHIR, HL7, interoperability standards
   - *Key takeaway*: Industry standards exist but adoption varies; ontologies bridge the gap
   - *Relevance*: Informed OWL namespace choices (aligned with FHIR where possible)

### **Software Engineering & Architecture**

8. **"System Design Interview" — Alex Xu**
   - *Why I read it*: Designing systems at scale
   - *Key takeaway*: Start simple, identify bottlenecks, then optimize
   - *Relevance*: Why we started with 3 tables, not 50. Proof-of-concept before scaling.

9. **"The Pragmatic Programmer" — Hunt & Thomas**
   - *Why I read it*: Philosophy of practical engineering
   - *Key takeaway*: "Make it work, make it right, make it fast" (in that order)
   - *Relevance*: Build MVP, validate with users, then optimize

10. **"Documentation as Code" — Anne Gentle**
    - *Why I read it*: How to document systems sustainably
    - *Key takeaway*: Keep docs in git, treat them like code
    - *Relevance*: All our ontologies, mappings, and architecture live in git + markdown

### **Business & Communication**

11. **"Made to Stick" — Chip & Dan Heath**
    - *Why I read it*: Communicating complex ideas simply
    - *Key takeaway*: Stories stick; metrics don't
    - *Relevance*: How I explain OWL to stakeholders: "It's like a contract for data"

12. **"The Goal" — Eliyahu Goldratt**
    - *Why I read it*: Systems thinking and bottlenecks
    - *Key takeaway*: Optimize the constraint, not the non-constraint
    - *Relevance*: Our validation bottleneck was manual reconciliation (10 hrs/mo). Dashboards solved it.

---

## 🔑 Key Insights & Lessons

### On Ontology Design
- **Start small**: One indicator, not 50. Prove value before scaling.
- **Involve stakeholders early**: OWL is technical; business users must validate the concepts.
- **Version everything**: Ontology versions, rule versions, schema versions. You'll need rollback.
- **Map bidirectionally**: Every SQL column should trace back to an OWL property for auditing.

### On Data Quality
- **CSV validation is 80% of the work**: Don't overcomplicate RDF validation for schema checks.
- **Foreign key integrity is your canary**: If FKs break, it signals orchestration problems, not just data errors.
- **Date parsing needs fallback formats**: AFAS exports European dates; Excel exports ISO. Handle both.
- **Visualization > Statistics**: A dashboard showing 95% pass rate is worth 100 pages of validation logs.

### On Implementation
- **Timeline is 2-3 years**: Proof of concept (month 1) → Production (month 6) → Full adoption (month 18+).
- **ROI is measurable**: Track hours saved + errors prevented. Our payback was 2.8 months.
- **Have backup demo methods**: Live Streamlit fails in Teams meetings. Have recorded video + PDF ready.
- **Documentation is an asset**: When people leave, does your system leave with them?

### On Teamwork
- **Translate between worlds**: OWL team, SQL team, and business users need different languages for the same concept.
- **Demonstrate impact early**: Catch 47 defects in month 1 → stakeholders fund expansion immediately.
- **Build with stakeholders, not for them**: Involve data stewards in dashboard design. They'll use it; executives won't.

---

## 🛠️ My Toolkit

### **Core Languages**
- **Python 3.9+** — primary language for data work
- **SQL** — SQL Server, PostgreSQL
- **Turtle/RDF** — semantic data representation
- **SPARQL** — querying semantic data
- **YAML/JSON** — configuration and metadata

### **Libraries & Frameworks**
- **rdflib** (RDF/OWL) | **pandas** (data wrangling) | **streamlit** (dashboards)
- **plotly** (visualization) | **sqlalchemy** (ORM) | **jupyter** (notebooks)

### **Platforms & Infrastructure**
- **SQL Server** | **PostgreSQL** | **Git** | **Jupyter Notebooks**
- **Streamlit** (dashboards) | **Microsoft Teams** (presentations)

### **Methodologies**
- **Semantic Web W3C Standards** — RDF, OWL, SPARQL
- **Dimensional Modeling** — Kimball approach for data warehouses
- **Agile/Iterative** — MVP first, feedback loops, incremental scaling
- **Documentation as Code** — Markdown + git for all design decisions

---

## 📈 Learning Goals (Next 12 Months)

- [ ] **SHACL Validation**: Move from CSV-level validation to RDF-level SHACL shapes
- [ ] **GraphQL for Semantic Data**: Query graphs with GraphQL instead of SPARQL
- [ ] **Multi-tenant Ontologies**: Design ontologies that serve multiple organizations/profiles
- [ ] **LLM-powered Data Quality**: Use LLMs to auto-generate validation rules from natural language requirements
- [ ] **RDF Stores at Scale**: Move from in-memory rdflib to Virtuoso or AllegroGraph
- [ ] **Knowledge Graph Applications**: Build recommendation engines, anomaly detection on semantic data

---

## 🔗 Resources I Reference

### **Websites**
- W3C Semantic Web: https://www.w3.org/standards/semanticweb/
- SPARQL Query Language: https://www.w3.org/TR/sparql11-query/
- Healthcare Ontologies: https://www.snomed.org/, https://loinc.org/, https://hl7.org/fhir/

### **Communities**
- Stack Overflow (tag: `rdf` `owl` `sparql`)
- W3C Semantic Web mailing lists
- Healthcare data standards groups

---

## 💡 How to Use This Document

**If you're learning semantics**:
- Start with books #1-3 (Semantic Web section)
- Build a small OWL ontology (5-10 classes)
- Write 10 SPARQL queries against it

**If you're building a data warehouse**:
- Read books #4-5 (Data Engineering)
- Design your CDM with dimensional modeling
- Map to OWL ontology (don't do it backwards)

**If you're implementing this in healthcare**:
- Read books #6-7 (Healthcare)
- Understand FHIR standards first
- Involve clinical teams early

**If you're struggling with communication**:
- Read book #11 (Made to Stick)
- Use the "translation table" approach (OWL ↔ SQL ↔ English)
- Show metrics, not methodology

---

## 📝 Continuous Learning

This README is a **living document**. As I read more books, learn new tools, or implement new patterns, I update it.

**Current focus areas**:
- Scaling semantic systems beyond laptop (~100M triples)
- Automating ontology generation from data (reverse engineering)
- Multi-domain ontology integration (healthcare + finance)

---

## 🙏 Acknowledgments

My learning has been shaped by:
- Mentors who explained complex concepts simply
- Teams who challenged me to solve real problems
- Communities (Stack Overflow, W3C) who shared knowledge freely
- Authors whose books made the abstract concrete

---

## 📧 Notes

**This is a personal learning repository.** Designed for:
- Reflecting on what I've learned
- Communicating my knowledge to others
- Tracking my growth over time
- Building a reference for future projects

**Not designed for:**
- Comprehensive coverage of any topic (read the books for that!)
- Advanced research or novel insights
- Step-by-step tutorials (see the blog posts for that)

---

**Last Updated**: September 25, 2026  
**Status**: Active learning (updated monthly)
