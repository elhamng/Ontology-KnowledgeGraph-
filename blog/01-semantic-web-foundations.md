# Semantic Web Foundations: RDF, OWL, and Healthcare Data Integration

## Introduction

Healthcare systems today operate in silos. A patient record lives in one hospital's system, financial data in another, care plans in a third. Each system uses different data models, different terminology, different ways of storing the same information.

Semantic web technologies—RDF, OWL, SPARQL—solve this problem by providing a **universal language for data integration**. This post explains the foundations and why they matter for healthcare.

---

## What is RDF? (Resource Description Framework)

RDF represents everything as **triples**: `(subject, predicate, object)`.

### Real Healthcare Example

```turtle
onz-zorg:medewerker123 a onz-g:Human .
onz-zorg:medewerker123 rdfs:label "Employee 123" .
onz-zorg:medewerker123 onz-g:hasDateOfBirth "1985-03-15"^^xsd:date .
```

This says:
- **Subject**: `onz-zorg:medewerker123` (a unique identifier using a URI)
- **Predicate**: `rdfs:label` (a property name)
- **Object**: `"Employee 123"` (a value)

### Why URIs Matter

Instead of saying "Employee ID 123" (which could mean different things in different systems), we use a **URI** (Uniform Resource Identifier):

```
http://purl.org/ozo/onz-zorg#medewerker123
```

This URI is:
- **Unique**: No two systems will accidentally use the same URI for different employees
- **Standardized**: Anyone can reference "this exact employee" using the same URI
- **Dereferenceable**: You can look up the definition (ideally)

### RDF Triple Store

Multiple triples form a **graph**:

```turtle
onz-zorg:medewerker123 a onz-g:Human ;
    rdfs:label "John Doe" ;
    onz-g:hasDateOfBirth "1985-03-15"^^xsd:date ;
    onz-pers:worksAt onz-org:vestiging456 .

onz-org:vestiging456 a onz-org:Vestiging ;
    rdfs:label "Amsterdam Care Center" ;
    onz-g:hasAddress "Herengracht 100, Amsterdam" .
```

Now the **graph shows relationships**: Employee 123 works at Vestiging 456. These are not separate tables—they're linked by shared URIs.

---

## What is OWL? (Web Ontology Language)

OWL defines the **rules and structure** that RDF data should follow. It answers: "What properties must every employee have? What data types are allowed?"

### OWL Classes

A class is a category of thing:

```turtle
onz-g:Human a owl:Class ;
    rdfs:label "Human Being"@en ;
    rdfs:comment "A person in the healthcare system"@en .
```

### OWL Properties

Properties define relationships:

```turtle
onz-g:hasDateOfBirth a owl:DatatypeProperty ;
    rdfs:domain onz-g:Human ;
    rdfs:range xsd:date ;
    rdfs:label "has date of birth"@en .
```

This says:
- **Domain**: Only `Human` objects can have this property
- **Range**: The value must be an `xsd:date` (a specific date format)

### Cardinality Constraints

How many times can a property appear?

```turtle
onz-g:Human rdfs:subClassOf [
    a owl:Restriction ;
    owl:onProperty onz-g:hasDateOfBirth ;
    owl:minCardinality 1 ;  # At least one
    owl:maxCardinality 1 .  # At most one
] .
```

Translation: Every Human must have exactly one birth date.

---

## How OWL Differs From SKOS, Dublin Core, FOAF

These semantic standards serve different purposes:

| Standard | Purpose | Example |
|----------|---------|---------|
| **SKOS** | Concept relationships | "Broader term: Revenue; Narrower term: Care Revenue" |
| **Dublin Core** | Document metadata | "Author: Jane Smith; Created: 2024-01-15" |
| **FOAF** | Person/org profiles | "Person knows Person; Person worksWith Person" |
| **OWL** | Formal constraints for data validation | "Employee must have birth date (xsd:date); must have exactly one employment contract" |

For healthcare and finance, you need **OWL**, not SKOS or Dublin Core. SKOS is too loose; OWL provides the formal structure needed for regulatory compliance.

---

## RDFS vs OWL: A Quick Comparison

### RDFS (RDF Schema)
- **Basic structure**: Classes, inheritance, property definitions
- **Use case**: Simple taxonomies ("Person is subclass of Agent")

### OWL (Web Ontology Language)
- **Rich semantics**: Cardinality, disjointness, restrictions
- **Use case**: Complex domains requiring validation (healthcare, finance)

**Example**: RDFS says "Employee is a Person." OWL says "Employee is a Person AND must have a birth date (xsd:date) AND must have between 1 and 3 employment contracts."

---

## SPARQL: Querying Semantic Data

SPARQL is like SQL, but for RDF graphs:

```sparql
PREFIX onz-g: <http://purl.org/ozo/onz-g#>
PREFIX onz-org: <http://purl.org/ozo/onz-org#>

SELECT ?employee ?workplace
WHERE {
    ?employee a onz-g:Human ;
              onz-pers:worksAt ?workplace .
    ?workplace a onz-org:Vestiging .
}
```

This finds: "Which employees work at which care centers?"

---

## Why Semantic Web for Healthcare?

### Problem 1: Data Silos
- Hospital A calls it "Employee ID"
- Hospital B calls it "Staff Number"
- They're the same thing, but systems can't talk about it

**Semantic Web Solution**: Use shared URIs. Both hospitals reference `http://purl.org/ozo/onz-zorg#medewerker123`.

### Problem 2: Regulatory Requirements
Healthcare has strict rules:
- "Every employee must have a birth date"
- "Every patient must have a care plan"
- "Every contract must have a start and end date (or end date is null for permanent)"

**Semantic Web Solution**: OWL constraints make these rules machine-readable. Validation is automatic.

### Problem 3: Ad-Hoc Queries
Analyst asks: "Show me all active contracts that started more than 5 years ago, grouped by care center."

**Traditional SQL**: Join 4 tables, write complex WHERE clause, hope joins are correct

**SPARQL**: Query the semantic graph directly. Relationships are explicit; no joins needed.

---

## Key Takeaways

1. **RDF = Triples**: Everything is (subject, predicate, object). URIs enable linking across systems.
2. **OWL = Constraints**: Defines rules that RDF data must follow (datatypes, cardinality, hierarchies).
3. **SPARQL = Query Language**: Query RDF graphs like you query databases, but more intuitive for relationships.
4. **SKOS/Dublin Core are NOT OWL**: They're too generic for healthcare/finance. Use OWL for regulatory domains.
5. **Why it matters**: One semantic language (RDF+OWL) lets you integrate data from 10 different hospital systems without custom mapping code.

---

## Next Steps

In the next post, we'll see this in action: building an **OWL Pipeline** that extracts business requirements (from markdown documents) and automatically generates validation rules, SQL schemas, and Excel exports.

---

## Resources

- W3C RDF Primer: https://www.w3.org/TR/rdf-primer/
- W3C OWL Overview: https://www.w3.org/TR/owl2-overview/
- SPARQL Query Language: https://www.w3.org/TR/sparql11-query/

---

**Image Suggestions**:
1. RDF Triple diagram (subject → predicate → object)
2. Healthcare data silos before/after semantic web
3. OWL vs SKOS/Dublin Core comparison table
4. SPARQL query example with graph visualization
