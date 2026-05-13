---
title: "Use Case: HemaFAIR Training School"
authors:
  - name: "Daphne Wijnbergen"
    orcid: "https://orcid.org/0000-0002-7449-6657"
  - name: "Rowdy de Groot"
    orcid: "https://orcid.org/0000-0002-1248-1986"
  - name: "Martijn Kersloot"
    orcid: "https://orcid.org/0000-0003-3357-3027"
---

# Use Case: HemaFAIR Training School

This use case uses synthetic data generated from one of the registries from the
[HemaFAIR](https://github.com/AmsterdamUMC/Ontop4OMOP/tree/hemafair) project.
The dataset contains data of patients with rare inherited haematological disorders,
focusing on three primary diagnoses:

- **Beta-thalassaemia**
- **Alpha-thalassaemia**
- **Sickle cell disease**

The dataset contains demographics, genotypes, iron burden measurements (ferritin,
serum iron, cardiac and liver MRI T2\*), transfusion history, chelation and other
treatments, and comorbidities. The data were mapped to the
[OMOP CDM](https://www.ohdsi.org/data-standardization/) and loaded into PostgreSQL.

This chapter demonstrates how to expose that OMOP PostgreSQL database as a
SPARQL endpoint using Ontop and the
[OMOP OWL ontology](https://github.com/AmsterdamUMC/Ontop4OMOP/blob/main/ontology/OMOP.ttl),
then answer six clinical registry questions with SPARQL.

:::{note}
A live SPARQL endpoint for this use case is available at
[http://51.15.26.149:8081/](http://51.15.26.149:8081/).
You can paste any query from this chapter directly into the Ontop web UI.
:::

---

## The mappings

The OBDA mapping file targets the `omop` schema in PostgreSQL and uses the
OMOP OWL ontology namespace (`https://w3id.org/omop/ontology/`).

A key design choice: the ontology defines a single property `omop:has_concept`
for **all** concept relationships (condition, measurement, observation, procedure,
drug, visit). The domain of each event class is established by the `a` type
triple, not by separate concept properties.

### Person mapping

```
mappingId      person_class
target         ex:person/{person_id} a omop:Person .
source         SELECT person_id FROM omop.person

mappingId      person_gender
target         ex:person/{person_id} omop:has_gender ex:concept/{gender_concept_id} .
source         SELECT person_id, gender_concept_id FROM omop.person
```

### Event-to-person direction

The ontology models the relationship from **Person → event**, not event → person.
This reverses the direction compared to the original SQL foreign key:

```
mappingId      condition_person
target         ex:person/{person_id} omop:has_condition_occurrence ex:condition/{condition_occurrence_id} .
source         SELECT condition_occurrence_id, person_id FROM omop.condition_occurrence
```

### Concept labels

Concept names are mapped as both `omop:name` (ontology-defined) and `rdfs:label`
(standard RDF), so queries can use the familiar `rdfs:label` predicate:

```
mappingId      concept_class
target         ex:concept/{concept_id} a omop:Concept ;
                   omop:name "{concept_name}" ;
                   rdfs:label "{concept_name}" .
source         SELECT concept_id, concept_name FROM omop.concept
```

### Tables covered

| OMOP domain | IRI pattern | Key properties mapped |
|---|---|---|
| Person | `ex:person/{id}` | `omop:has_gender`, `omop:year_of_birth` |
| VisitOccurrence | `ex:visit/{id}` | `omop:has_concept` |
| ConditionOccurrence | `ex:condition/{id}` | `omop:has_concept` |
| Measurement | `ex:measurement/{id}` | `omop:has_concept`, `omop:value_as_number` |
| Observation | `ex:observation/{id}` | `omop:has_concept`, `omop:has_value_as_concept`, `omop:value_source_value` |
| ProcedureOccurrence | `ex:procedure/{id}` | `omop:has_concept` |
| DrugExposure | `ex:drug/{id}` | `omop:has_concept` |
| Concept | `ex:concept/{id}` | `omop:name`, `rdfs:label` |

---

## Prefixes used in all queries

```sparql
PREFIX omop:         <https://w3id.org/omop/ontology/>
PREFIX omop_concept: <http://example.org/omop/concept/>
PREFIX rdfs:         <http://www.w3.org/2000/01/rdf-schema#>
```

:::{note}
Concept IRIs are constructed as `omop_concept:{concept_id}`, e.g.
`omop_concept:4278669` for Beta-thalassaemia. This prefix avoids forward
slashes inside SPARQL `FILTER` expressions.
:::

---

## Example SPARQL queries

### Overview — row counts across clinical tables

A quick sanity check that all tables are populated after loading the data.

```sparql
PREFIX omop: <https://w3id.org/omop/ontology/>

SELECT ?table_name (COUNT(?x) AS ?row_count) WHERE {
  { ?x a omop:Person .              BIND("person"               AS ?table_name) }
  UNION
  { ?x a omop:VisitOccurrence .     BIND("visit_occurrence"     AS ?table_name) }
  UNION
  { ?x a omop:ConditionOccurrence . BIND("condition_occurrence" AS ?table_name) }
  UNION
  { ?x a omop:Measurement .         BIND("measurement"          AS ?table_name) }
  UNION
  { ?x a omop:Observation .         BIND("observation"          AS ?table_name) }
  UNION
  { ?x a omop:ProcedureOccurrence . BIND("procedure_occurrence" AS ?table_name) }
  UNION
  { ?x a omop:DrugExposure .        BIND("drug_exposure"        AS ?table_name) }
} GROUP BY ?table_name ORDER BY ?table_name
```

Expected results:

| table_name | row_count |
|---|---|
| condition_occurrence | 2685 |
| drug_exposure | 0 |
| measurement | 2263 |
| observation | 6556 |
| procedure_occurrence | 1070 |
| person | 999 |
| visit_occurrence | 999 |

---
### Exploring the RDF using DESCRIBE
Query to find all properties of person 1 in the dataset.

```sparql
PREFIX omop:         <https://w3id.org/omop/ontology/>
PREFIX omop_concept: <http://example.org/omop/concept/>
PREFIX rdfs:         <http://www.w3.org/2000/01/rdf-schema#>

DESCRIBE <http://example.org/omop/person/1>
```
Expected result:

```ttl
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix ex: <http://example.org/> .
@prefix omop: <https://w3id.org/omop/ontology/> .
@prefix rdf4j: <http://rdf4j.org/schema/rdf4j#> .
@prefix sesame: <http://www.openrdf.org/schema/sesame#> .
@prefix owl: <http://www.w3.org/2002/07/owl#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .
@prefix fn: <http://www.w3.org/2005/xpath-functions#> .

<http://example.org/omop/person/1> a omop:Person;
  omop:has_gender <http://example.org/omop/concept/8507>;
  omop:year_of_birth 1957;
  omop:has_visit_occurrence <http://example.org/omop/visit/1>;
  omop:has_condition_occurrence <http://example.org/omop/condition/1>, <http://example.org/omop/condition/1000>,
    <http://example.org/omop/condition/1177>;
  omop:has_measurement <http://example.org/omop/measurement/1>, <http://example.org/omop/measurement/666>;
  omop:has_observation <http://example.org/omop/observation/1>, <http://example.org/omop/observation/1000>,
    <http://example.org/omop/observation/1999>, <http://example.org/omop/observation/2998>,
    <http://example.org/omop/observation/3997>, <http://example.org/omop/observation/4996>,
    <http://example.org/omop/observation/5995>;
  omop:has_procedure_occurrence <http://example.org/omop/procedure/1>, <http://example.org/omop/procedure/764> .
```

```sparql
PREFIX omop:         <https://w3id.org/omop/ontology/>
PREFIX omop_concept: <http://example.org/omop/concept/>
PREFIX rdfs:         <http://www.w3.org/2000/01/rdf-schema#>

DESCRIBE <http://example.org/omop/observation/1>
```

Query to find properties of the first obervation about person 1

Expected result:

```ttl
@prefix omop: <https://w3id.org/omop/ontology/> .
@prefix omop_concept: <http://example.org/omop/concept/> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdf4j: <http://rdf4j.org/schema/rdf4j#> .
@prefix sesame: <http://www.openrdf.org/schema/sesame#> .
@prefix owl: <http://www.w3.org/2002/07/owl#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .
@prefix fn: <http://www.w3.org/2005/xpath-functions#> .

<http://example.org/omop/observation/1> a omop:Observation;
  omop:has_concept omop_concept:4152209;
  omop:value_source_value "Cyprus" .

```

Query to find the properties of the concept id describing this observation

```sparql
PREFIX omop:         <https://w3id.org/omop/ontology/>
PREFIX omop_concept: <http://example.org/omop/concept/>
PREFIX rdfs:         <http://www.w3.org/2000/01/rdf-schema#>

DESCRIBE omop_concept:4152209
```

Expected result:

```ttl
@prefix omop: <https://w3id.org/omop/ontology/> .
@prefix omop_concept: <http://example.org/omop/concept/> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdf4j: <http://rdf4j.org/schema/rdf4j#> .
@prefix sesame: <http://www.openrdf.org/schema/sesame#> .
@prefix owl: <http://www.w3.org/2002/07/owl#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .
@prefix fn: <http://www.w3.org/2005/xpath-functions#> .

omop_concept:4152209 a omop:Concept;
  omop:name "Born in Cyprus";
  rdfs:label "Born in Cyprus" .
```

### Q1 — How many patients have each diagnosis?

Counts distinct patients per primary diagnosis using the three haematological
condition concept IDs. Labels are resolved from the concept table via `rdfs:label`.

```sparql
PREFIX omop:         <https://w3id.org/omop/ontology/>
PREFIX omop_concept: <http://example.org/omop/concept/>
PREFIX rdfs:         <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?diagnosis (COUNT(DISTINCT ?person) AS ?n_patients) WHERE {
  ?person omop:has_condition_occurrence ?condition .
  ?condition omop:has_concept ?concept .
  ?concept rdfs:label ?diagnosis .
  FILTER(?concept IN (omop_concept:4287844, omop_concept:4278669, omop_concept:25518))
} GROUP BY ?diagnosis ORDER BY DESC(?n_patients)
```

Expected results:

| diagnosis | n_patients |
|---|---|
| Beta thalassemia | 739 |
| Alpha thalassemia | 258 |
| Sickle cell trait | 2 |

---

### Q2 — What is the ferritin level by sex?

Iron overload (high ferritin) is a major complication in thalassaemia, driven
by transfusions and increased gut iron absorption. This query retrieves all
ferritin measurements (LOINC concept 37208753) with the patient's sex label.

SPARQL does not support a MEDIAN aggregate. The query below uses `AVG` as a
summary; for the true median, download the raw values (Q2b) and compute
client-side.

**Q2a — Average ferritin by sex:**

```sparql
PREFIX omop:         <https://w3id.org/omop/ontology/>
PREFIX omop_concept: <http://example.org/omop/concept/>
PREFIX rdfs:         <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?sex (AVG(?value) AS ?avg_ferritin) (COUNT(?meas) AS ?n_measurements)
WHERE {
  ?person omop:has_measurement ?meas .
  ?meas omop:has_concept omop_concept:37208753 ;
        omop:value_as_number ?value .
  ?person omop:has_gender ?genderConcept .
  ?genderConcept rdfs:label ?sex .
} GROUP BY ?sex
```

**Q2b — Raw ferritin values with sex (for median computation):**

```sparql
PREFIX omop:         <https://w3id.org/omop/ontology/>
PREFIX omop_concept: <http://example.org/omop/concept/>
PREFIX rdfs:         <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?sex ?value WHERE {
  ?person omop:has_measurement ?meas .
  ?meas omop:has_concept omop_concept:37208753 ;
        omop:value_as_number ?value .
  ?person omop:has_gender ?genderConcept .
  ?genderConcept rdfs:label ?sex .
}
```

Reference median from SQL (ng/mL): Female ≈ 885.4 · Male ≈ 764.3

---

### Q3 — Transfusion status distribution per diagnosis

Joins each patient's primary diagnosis with their qualitative transfusion
status (observation concept 40758326). Transfusion dependency is a key
phenotypic stratifier in thalassaemia.

:::{note}
Transfusion status in this dataset is stored in `value_source_value` as free
text. The ontology property `omop:value_source_value` is used here rather than
`omop:has_value_as_concept` because `value_as_concept_id` is not populated for
this observation type.
:::

```sparql
PREFIX omop:         <https://w3id.org/omop/ontology/>
PREFIX omop_concept: <http://example.org/omop/concept/>
PREFIX rdfs:         <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?diagnosis ?transfusion_status (COUNT(DISTINCT ?person) AS ?n_patients)
WHERE {
  ?person omop:has_condition_occurrence ?condition .
  ?condition omop:has_concept ?diagConcept .
  ?diagConcept rdfs:label ?diagnosis .
  FILTER(?diagConcept IN (omop_concept:4287844, omop_concept:4278669, omop_concept:25518))

  ?person omop:has_observation ?obs .
  ?obs omop:has_concept omop_concept:40758326 ;
       omop:value_source_value ?transfusion_status .
} GROUP BY ?diagnosis ?transfusion_status ORDER BY ?diagnosis ?transfusion_status
```

Expected results:

| diagnosis | transfusion_status | n_patients |
|---|---|---|
| Alpha thalassemia | No | 134 |
| Alpha thalassemia | Yes (Occasional) | 103 |
| Alpha thalassemia | Yes (Regular) | 21 |
| Beta thalassemia | No | 230 |
| Beta thalassemia | Yes (Occasional) | 345 |
| Beta thalassemia | Yes (Regular) | 164 |
| Sickle cell trait | No | 2 |

---

### Q4 — Most common comorbidities

Excludes the three primary diagnoses and unmapped concepts (concept_id = 0),
then counts how many patients have each secondary condition. Bone disease and
endocrine complications are hallmarks of thalassaemia.

```sparql
PREFIX omop:         <https://w3id.org/omop/ontology/>
PREFIX omop_concept: <http://example.org/omop/concept/>
PREFIX rdfs:         <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?comorbidity (COUNT(DISTINCT ?person) AS ?n_patients)
WHERE {
  ?person omop:has_condition_occurrence ?condition .
  ?condition omop:has_concept ?concept .
  ?concept rdfs:label ?comorbidity .
  FILTER(?concept NOT IN (omop_concept:4287844, omop_concept:4278669,
                          omop_concept:25518, omop_concept:0))
} GROUP BY ?comorbidity ORDER BY DESC(?n_patients)
```

Expected results (percentage of 999 patients):

| comorbidity | n_patients | % |
|---|---|---|
| Vitamin D deficiency | 605 | 60.6 |
| Osteopenia | 503 | 50.4 |
| Osteoporosis | 399 | 39.9 |
| Infertile | 177 | 17.7 |
| Acute chest syndrome | 2 | 0.2 |

---

### Q5 — Patients on chelation with no transfusion history

Chelation therapy removes excess iron from the body. It is typically
prescribed to transfusion-dependent patients, but some non-transfused patients
also develop iron overload from increased intestinal absorption.

This query counts patients who are on chelation (procedure concept 4068544)
**and** whose recorded transfusion status is "No".

```sparql
PREFIX omop:         <https://w3id.org/omop/ontology/>
PREFIX omop_concept: <http://example.org/omop/concept/>

SELECT (COUNT(DISTINCT ?person) AS ?n_chelation_no_transfusion) WHERE {
  ?person omop:has_procedure_occurrence ?proc .
  ?proc omop:has_concept omop_concept:4068544 .
  ?person omop:has_observation ?obs .
  ?obs omop:has_concept omop_concept:40758326 ;
       omop:value_source_value "No" .
}
```

Expected result: **249**

---

### Q6 — Measurement completeness

For each numeric measurement type, counts how many distinct patients have a
value recorded. Patients without a value are excluded by the mapping
(NULL `value_as_number` rows are filtered at the SQL level), so
`n_missing = 999 − n_with_value`.

```sparql
PREFIX omop: <https://w3id.org/omop/ontology/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?measurement (COUNT(DISTINCT ?person) AS ?n_with_value)
WHERE {
  ?person omop:has_measurement ?meas .
  ?meas omop:has_concept ?concept ;
        omop:value_as_number ?value .
  ?concept rdfs:label ?measurement .
} GROUP BY ?measurement ORDER BY DESC(?n_with_value)
```

Expected results:

| measurement | n_with_value | n_missing |
|---|---|---|
| Ferritin mass concentration in serum | 863 | 136 |
| Serum iron measurement | 15 | 984 |

