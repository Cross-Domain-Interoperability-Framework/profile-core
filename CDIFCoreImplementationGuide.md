# CDIF Core Classes and Properties

2026-04-05

This document includes all of the classes and properties for the CDIF Core profile implementation using the schema.org vocabulary. The Core profile provides the mandatory and optional base properties for any CDIF metadata record. The profile consists of 1) a set of classes and properties containing content that is required in every valid CDIF document instance; and 2) a set of classes and properties that are expected to be applicable to a wide variety of resources across knowledge domains. Although classes and properties in the second group are optional, this profile provides recommendations for implementation that will enable interoperability. As used here, classes are named objects with a set of named properties, each of which has an expected data type that specifies its value. Datatypes can be simple literals (number, string, boolean), references to objects that are not serialized inline, or instances of a class. Conformance to the profile entails populating all mandatory content, and using the recommended classes and properties for the optional content. The implementation target is an rdf serialization, which is an open world logical model; users are thus free to add additional properties that they find useful for dataset documentation in their community, but these can be ignored by other users without penalty.

The JSON syntax is defined by the [ECMA JSON specification](https://www.ecma-international.org/publications-and-standards/standards/ecma-404/), and JSON-LD is specified in the [JSON-LD 1.1 recommendation](https://www.w3.org/TR/json-ld11/) from the World Wide Web Consortium (W3C). This serialization is designed for linked data applications that will translate the JSON into a set of {subject, predicate, object} triples that can be loaded into an RDF database for processing. The JSON-LD context binds JSON keys to URIs for more precise semantics, and the use of URIs to identify entities and property values in the metadata will maximize the linkage with resources on the wider web to build an ever-expanding global knowledge graph.

# Table of contents

- [Model](#model)
- [Notes on schema.org implementation](#notes-on-schemaorg-implementation)
  - [JSON-LD @type](#json-ld-type)
  - [Object reference](#object-reference)
  - [Repeating values](#repeating-values)
  - [Namespace prefixes and JSON validation.](#namespace-prefixes-and-json-validation)
  - [Use of dcat:CatalogRecord](#use-of-dcatcatalogrecord)
  - [identifier and version identify different things](#identifier-and-version-identify-different-things)
  - [Polymorphism of PropertyValue](#polymorphism-of-propertyvalue)
- [Namespaces](#namespaces)  
- [Base Class DataSet](#base-class-dataset)
- [Required Properties from cdif Core profile](#required-properties-from-cdif-core-profile)  - [Action](#action)
- [Optional Dataset properties](#optional-dataset-properties)- 
- [Other Classes used for CDIF Core](#other-classes-used-for-cdif-core)
  - [Data Download](#data-download)
  - [DataCatalog](#datacatalog)
  - [MonetaryGrant](#monetarygrant)
  - [Organization](#organization)
  - [Person](#person)
  - [PropertyValueSpecification](#propertyvaluespecification)  
  - [Web API](#web-api)
- [Data types used for CDIF Core](#data-types-used-for-cdif-core)
  - [ContactPoint](#contactpoint)
  - [Contributor](#contributor)
  - [dcat:CatalogRecord](#dcatcatalogrecord)
  - [Defined Term](#defined-term)
  - [EntryPoint](#entrypoint)
  - [Labeled Link](#labeled-link)
  - [LinkRole](#linkrole)
  - [PropertyValue-(identifier)](#propertyvalue-identifier)
  - [spdx:Checksum](#spdxchecksum)

# Model

[↑ Back to TOC](#table-of-contents)

The information model for the base core profile is defined in the [cdifBook](https://cross-domain-interoperability-framework.github.io/cdifbook/preview-2026-05/metadata/core/#information-model). This section outlines a logical model focused on a JSON-LD implementation of the content items defined in the information model. All classes and properties are implemented with schema.org types and attributes unless there is a prefix indicating use of elements from other vocabularies. See the context section for prefixes used and their mapping to URIs.

# Notes on schema.org implementation

## JSON-LD @type

JSON-LD every graph node has a \@type property that specifies the rdf:type for the node. This type has implications for the properties expected to be found in the content of the node, and should convey the intention of the kind of thing the node is intended to represent. In the CDIF JSON-LD implementation, most of the \@types are taken from the schema.org vocabulary, but there are a few exceptions for content items that do not map to the schema.org vocabulary. The \@type is always serialized as an array [JSON list] to allow for extensions that add additional typing.

## Object reference

[↑ Back to TOC](#table-of-contents)

Linked data is implemented in rdf using URIs to reference objects that might be located in other parts of a graph, or remotely and accessed online. In the JSON-LD implementation, simply using a URI string as the value of a property does not create such a link—the value is simply a string, not the object reference by the URI. An "object ref" is always a string containing the id of the referenced object. Thus

*"schema:funder": "<https://ror.org/021nxhr62>"*

Does not create a link. Object references are implemented in JSON-LD as object that have a single node identifier as it property.

*"schema:funder": { "@id": "https://ror.org/021nxhr62" }*

Is the correct syntax to implemenat an object reference. Throughout this document, if 'object reference' is included as a value type for a property, be aware that instance documents might simply have this kind of object as the property value.

## URI-shape values in `propertyID` and `additionalType`

[↑ Back to TOC](#table-of-contents)

Building on the object-reference convention above, CDIF profiles enforce a JSON-LD semantic-clarity rule for two schema.org properties whose values often name external concepts: `schema:propertyID` and `schema:additionalType`. If the value's lexical form matches a URI or CURIE (`scheme:localname`, e.g. `wd:Q3099911`, `dcat:CatalogRecord`, `https://orcid.org/0000-...`), it MUST be serialized as an IRI reference — `{"@id": "..."}` — not as a bare string literal. A URI-shape string serialized as a literal does not participate in RDF entailment as a resource reference, which defeats the interoperability the URI was intended to provide.

Free-label strings (`"temperature"`, `"MaterialSample"`) remain valid as string values. `schema:DefinedTerm` objects (with or without an explicit `@type: schema:DefinedTerm`) also satisfy the rule.

The rule is enforced by two SHACL shapes in the aggregated `*Rules.shacl`:

- `cdifd:PropertyIDUriShouldBeIRIShape` — targets objects of `schema:propertyID`.
- `cdifd:AdditionalTypeUriShouldBeIRIShape` — targets objects of `schema:additionalType`.

Both fire at `sh:Violation` severity when a URI-shape string literal appears; documents that fail either shape are not conformant.

Correct:

```json
"schema:additionalType": [
    { "@id": "dcat:CatalogRecord" },
    "material-sample"
],
"schema:propertyID": [
    { "@id": "https://qudt.org/vocab/quantitykind/Temperature" }
]
```

Incorrect (SHACL will flag):

```json
"schema:additionalType": ["dcat:CatalogRecord"],
"schema:propertyID": ["https://qudt.org/vocab/quantitykind/Temperature"]
```

## Repeating values

[↑ Back to TOC](#table-of-contents)

Any property with a 1..\* or 0..\* cardinality has values that are always implemented as arrays. This makes client processing easier because tests for single or array values are not necessary. If a property is ‘repeatable’, then assume the implementation is an array (JSON list).

## Namespace prefixes and JSON validation.

[↑ Back to TOC](#table-of-contents)

Namespace prefixes are explicitly used in the example documents so that the JSON schema can validate instance documents. JSON Schema validates the literal JSON structure -- property names, nesting, value types. Several features of JSON-LD can cause a semantically correct document to fail JSON Schema checks. The same property can appear as "schema:name", "name", or "http://schema.org/name" depending on the \@context. A JSON Schema that checks for "schema:name" will reject a document that uses "name", even though both mean the same thing. See [Validating CDIF Profile Metadata](https://github.com/Cross-Domain-Interoperability-Framework/validation/blob/main/docs/CDIF-profiles-metadata-validation.md) for a detailed discussion of validation processes for CDIF metadata, and the use of framing to validate JSON-LD instances using different [JSON-LD forms](https://www.w3.org/TR/json-ld11/#forms-of-json-ld) or custom context documents.

The JSON Schema validates **one metadata record at a time**: the document root must be a single `schema:Dataset` node with the mandatory properties at the top level. A multi-record bundle packaged as `{ "@graph": [ ... ] }` -- the natural output of many harvesters and federated catalogs -- will fail JSON Schema validation immediately, because `@graph` is not in the schema's property list. This is purely a packaging mismatch: the bundled records may be perfectly valid individually, and SHACL (which operates on the RDF graph rather than the JSON tree) will still pass them. Before JSON-Schema validation, extract each `schema:Dataset` record from the graph using JSON-LD framing. The `FrameAndValidate.py` script and `cdifCore-frame.jsonld` frame in this repository do exactly this -- framing a document against the CDIF frame and extracting the dataset node out of `@graph` -- and can be run with `--validate` to frame and JSON-Schema-validate in one step.

## Use of dcat:CatalogRecord

[↑ Back to TOC](#table-of-contents)

In a harvesting/federated catalog system some metadata about the metadata is useful to keep track of where metadata came from, what format/profile it uses (harvesters need this to process), and update dates. Unambiguous expression of this information requires making statements about a metadata record distinct from the thing in the world that the metadata describes. In an RDF framework, this requires a distinct identifier for the metadata record object that will serve as the subject for these triples.

In the RDF serialization, [Schema.org](http://schema.org/) metadata records are [JSON-LD node objects](https://www.w3.org/TR/json-ld/#node-objects), and include an "@id" keyword with a value that identifies the node, analogous to a primary key in a relational database. This identifier can be interpreted to represent a thing in the world that the metadata record (the 'node') is about, or to represent the metadata record (a JSON object) itself.

To avoid this ambiguity, CDIF adopts the convention that the [schema.org](http://schema.org/) identifier property is used to identify a thing in the world that is the subject of the JSON-LD node. The identified thing might be physical, imaginary, abstract, or a digital object. The JSON-LD \@id property identifies a node in a graph, which is an abstract object. As a URI the \@id URI is expected to dereference to produce a JSON-LD object containing the properties that are attached to the graph node.

Given this convention, when the metadata record is processed, the processor should use the schema:identifier as subject of triples about the subject of the metadata record to avoid ambiguity. In addition, this convention would suggest that if a schema:identifier property is present, the \@id property should be interpreted to identify the JSON object that is the representation of the node in the knowledge graph. In practice JSON-LD processors use the \@id as the subject of triples generated from a JSON-LD object. A 'purist' approach would require a level of indirection to assert that the \@id is about the thing identified by the schema:identifier. JSON-LD processors don't do this, so standard practice is to make the \@id the identifier for the described resource, requiring understanding that it identifies two things—the rdf object at that graph node, and the thing in the world described by the content of that node. This has worked for the most part because metadata providers have been quite lax in providing information about the provenance of the metadata node, and in particular the conformance criteria that were followed in generating the content of that node.

To address this issue, CDIF recommends that statements about the metadata record (the JSON object) as a distinct entity should be made using a separate identified node object. This node object is typed as a schema:Dataset, with additionalType [dcat:CatalogRecord](https://www.w3.org/TR/vocab-dcat-3/#Class:Catalog_Record) recognizing that the DCAT v3 specification uses that element to address this precise issue. This node can be embedded in the Dataset metadata using the subjectOf property, and approach used in the accompanying JSON schema and examples, or implemented as a separate free standing graph node linked to the dataset object via the 'about' [object reference](#object-reference).

```json
{
  "@context": [
    "https://schema.org",
    {
      "dcterms": "http://purl.org/dc/terms/",
      "ex": "https://example.com/99152/"
    }
  ],
  "@id": "ex:URIforNode1",
  "@type": "appropriate schema.org type",
  "identifier": "ex:URIforDescribedResource",
  "name": "unique title for the resource",
  "description": "Description of the resource",
  "subjectOf": {
    "@id": "ex:URIforNode2",
    "@type": "Dataset",
    "additionalType": "dcat:CatalogRecord",
    "sdDatePublished": "2017-05-23",
    "about": { "@id": "ex:URIforNode1" },
    "description": "metadata about documentation for ex:URIforDescribedResource",
    "dcterms:conformsTo": { "@id": "https://w3id.org/cdif/core/1.1" }
  }
}
```

*Example: Instance with dcat CatalogRecord content (mapped to schema.org properties)*

## identifier and version identify different things

[↑ Back to TOC](#table-of-contents)

`schema:identifier` and `schema:version` are not two granularities of the same name. They answer different questions, and a record needs both because a consumer has both.

**`schema:identifier` is stable across versions.** It names the *intellectual intention* of the dataset — what it is for: its subject, its content type, its data model, its format. It is what a reader cites when they mean "the Indian Ocean radiocarbon dataset", and it should keep resolving as the data is corrected, reprocessed and reissued, because none of that changes what the dataset is *for*. A citation using it stays current without being rewritten.

**`schema:version` designates a particular version, used in a particular context.** It names the exact content that went into some analysis, and therefore bears on that analysis's conclusions. Where `schema:identifier` resolves to whatever is current, a version designation must stay attached to one immutable state — if the content it names can change, a result cited against it is not reproducible.

Conflating the two costs one of them:

- Record only the stable identifier, and the work is not reproducible: the content moved underneath the citation, and a later reader cannot recover what was actually used.
- Record only the version, and the reference rots: it points at a superseded snapshot when the reader wanted the dataset.

So a record describing a specific revision should carry **both** — `schema:identifier` for the thing that persists, `schema:version` for the state in hand — and a consumer resolving one should not assume it has the other.

Where a version needs its own *resolvable* identifier rather than a version string, give that version its own metadata record with its own `schema:identifier`, and relate the two records with `prov:wasDerivedFrom` (see the Provenance profile).

CDIF does not currently define a property that marks one identifier as stable and another as version-specific, so **the distinction is carried by which property is used**, and should be stated in prose wherever a consumer might mistake one for the other.

For comparison, DataONE makes the same distinction structural: every object has an immutable per-version PID, plus a `seriesId` that resolves to the current head. The DOI — the identifier humans cite — is attached to the series, not to the snapshot.

## Polymorphism of PropertyValue

[↑ Back to TOC](#table-of-contents)

The schema.org PropertyValue type is used in several different contexts in the implementation of CDIF metadata. This is a result of how the expected values for some important properties are defined in schema.org. In the Discovery profile, PropertyValue is an allowed value type for variableMeasured and for identifier. In some more advanced profiles, PropertyValue is also an allowed value for additionalProperty.

The following table compared the properties and requirements for this schema.org type in these different contexts.

<table>
<colgroup>
<col style="width: 28%" />
<col style="width: 25%" />
<col style="width: 12%" />
<col style="width: 17%" />
<col style="width: 17%" />
</colgroup>
<thead>
<tr>
<th><strong>Property</strong></th>
<th><strong>Description</strong></th>
<th><strong>identifier</strong></th>
<th><strong>variableMeasured</strong></th>
<th><strong>additionalProperty</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td><strong>@type</strong></td>
<td>Type declaration (must contain schema:PropertyValue)</td>
<td><strong>1..*<br />
required</strong><br />
contains: PropertyValue</td>
<td><strong>1..*<br />
required</strong><br />
contains: PropertyValue</td>
<td><strong>1..*<br />
required</strong><br />
contains: PropertyValue</td>
</tr>
<tr>
<td><strong>@id</strong></td>
<td>URI identifier for this node</td>
<td>-</td>
<td>0..1<br />
string</td>
<td>-</td>
</tr>
<tr>
<td><strong>schema:name</strong></td>
<td>Human-readable label</td>
<td>-</td>
<td><strong>1<br />
required</strong><br />
string</td>
<td><strong>1<br />
required</strong><br />
string</td>
</tr>
<tr>
<td><strong>schema:description</strong></td>
<td>Textual description</td>
<td>-</td>
<td>0..1<br />
string, default: "missing"</td>
<td>-</td>
</tr>
<tr>
<td><strong>schema:alternateName</strong></td>
<td>Alternative names</td>
<td>-</td>
<td>0..*<br />
array of strings</td>
<td>-</td>
</tr>
<tr>
<td><strong>schema:propertyID</strong></td>
<td>Identifier for the property concept</td>
<td>0..1<br />
string<br />
(identifier scheme name)</td>
<td>0..*<br />
array of: string | {@id} | DefinedTerm</td>
<td><strong>1..*<br />
required</strong><br />
array of: string | {@id} | DefinedTerm;<br />
minItems: 1</td>
</tr>
<tr>
<td><strong>schema:value</strong></td>
<td>The property value</td>
<td>0..1<br />
(conditional)<br />
string;<br />
required if no schema:url</td>
<td>-</td>
<td><strong>1<br />
required</strong><br />
string | number | boolean | object</td>
</tr>
<tr>
<td><strong>schema:url</strong></td>
<td>Web-resolvable URL</td>
<td>0..1<br />
(conditional)<br />
string (uri format);<br />
required if no schema:value</td>
<td>0..1<br />
string (uri) | LabeledLink</td>
<td>-</td>
</tr>
<tr>
<td><strong>schema:unitText</strong></td>
<td>Unit of measurement as text</td>
<td>-</td>
<td>0..1<br />
string</td>
<td>0..1<br />
string</td>
</tr>
<tr>
<td><strong>schema:unitCode</strong></td>
<td>URI or code for unit of measure</td>
<td>-</td>
<td>0..1<br />
string | {@id} | DefinedTerm</td>
<td>0..1<br />
string | DefinedTerm</td>
</tr>
<tr>
<td><strong>schema:measurementTechnique</strong></td>
<td>How values were obtained</td>
<td>-</td>
<td>0..1<br />
string | {@id} | DefinedTerm</td>
<td>-</td>
</tr>
<tr>
<td><strong>schema:minValue</strong></td>
<td>Minimum numeric value</td>
<td>-</td>
<td>0..1<br />
number</td>
<td>-</td>
</tr>
<tr>
<td><strong>schema:maxValue</strong></td>
<td>Maximum numeric value</td>
<td>-</td>
<td>0..1<br />
number</td>
<td>-</td>
</tr>
</tbody>
</table>

# Namespaces

[↑ Back to TOC](#table-of-contents)

-Namespace prefixes use in CDIF Core schema.org JSON-LD objects are specified by this JSON-LD context, which must be declared in every instance document. Note that the correct namespace URI for schema.org is ‘http’, not ‘https’. The https://schema.org/ uri identifies the schema.org context document, not the namespace. This example context includes all the namespaces used in any cdif profile:

```json
"@context": {
  "schema": "http://schema.org/",
  "dcterms": "http://purl.org/dc/terms/",
  "spdx": "http://spdx.org/rdf/terms#",
  "prov": "http://www.w3.org/ns/prov#",
  "dcat": "http://www.w3.org/ns/dcat#",
  "ex": "https://example.org/",
  "xsd": "http://www.w3.org/2001/XMLSchema#"
}
```

# Base Class DataSet

[↑ Back to TOC](#table-of-contents)

-This profile applies to description of resources that can be described using the properties defined in the CDIF core information model. For implementation using the schema.org vocabulary, these are typed as schema:Dataset.

# Required Properties from cdif Core profile

[↑ Back to TOC](#table-of-contents)

### **\@id**

- **Cardinality:** Required
- **Content:** string
- **Description:** This is an identifier for this node in an rdf graph. JSON-LD key is @id.

### **\@type**

- **Cardinality:** Required – "Dataset", Repeatable
- **Content:** string.uri
- **Description:** The type property specifies the rdf:type classification. For this implementation, the type is represented with the JSON-LD \@type property, and must include 'Dataset'. JSON-LD key is \@type. Type assertions here should be understood to imply the usage of properties associated with the identified type, whether from schema.org or other vocabularies that might define the type.

### **name**

- **Cardinality:** Required
- **Content:** string
- **Description:** A descriptive name of a dataset (e.g., 'Snow depth in Northern Hemisphere'). The name should uniquely identify the described resource for human use, in the scope of the metadata catalog containing this metadata record. Schema.org property, in namespace 'http://schema.org/'.

### **identifier**

- **Cardinality:** Required
- **Content:** string.uri or [PropertyValue-(identifier)](#propertyvalue-identifier)
- **Description:** The primary identifier for the described resource; other identifiers should be listed in the sameAs field. CDIF recommends that if the identifier is a resolvable URI, use the string option; if the identifier is a string that is not a resolvable URI, use the schema:PropertyValue class to provide context for interpreting the identifier. Schema.org property, in namespace 'http://schema.org/'. **This identifier is stable across versions** — it names the dataset's intellectual intention, not the particular revision in hand; see [identifier and version identify different things](#identifier-and-version-identify-different-things).

### **dateModified**

- **Cardinality:** Required
- **Content:** string, ISO 8601 format
- **Description:** ISO8601 formatted date (and optional time if relevant) when Dataset was last updated

CHOICE at least one of two options:

### **conditionsOfAccess**

- **Cardinality:** Required if no license, Repeatable
- **Content:** string, [object reference](#object-reference), or [LabeledLink](#labeled-link)
- **Description:** Text statement of conditions for use and access; if an online resource documents the restrictions or a URI is used to identify the conditions, recommend using the LabeledLink option, implemented as schema:CreativeWork, to provide a label (name) and an identifier (URI or URL).

### **license**

- **Cardinality:** Required if no conditionsOfAccess
- **Content:** string, [object reference](#object-reference), or [LabeledLink](#labeled-link)
- **Description:** Legal statement of conditions for use and access; recommend using the [LabeledLink](#labeled-link) option, implemented by schema:CreativeWork to provide a label (name) for the license, and an identifier. Sources of license identifiers: https://opensource.org/licenses/, https://creativecommons.org/about/cclicenses/, https://spdx.org/licenses/, http://cor.esipfed.org/ont/earthcube/swl. If only a string is provided, it should be recognizable name for the license. If resolvable URI is available, use the object reference.

CHOICE at least one of two options:

### **url**

- **Cardinality:** Required if no distribution
- **Content:** string.uri
- **Description:** Web Location of a page describing the dataset (landing page), typically providing links or instructions to get the actual resource content; analogous to dcat:landingPage. If a direct link is available to get the data, put in distribution/DataDownload/contentUrl, that is equivalent to dcat:accessURL

### **distribution**

- **Cardinality:** Required if no url
- **Content:** [DataDownload](#data-download) or [WebAPI](#web-api)
- **Description:** specifies how to download the data in a specific format or access via a web API. This property describes where to get the data and in what format by using the schema:DataDownload type. If user must access data through a landing page, provide link to landing page in the 'url' property for the dataset, not a distribution contentUrl.

### **subjectOf**

- **Cardinality:** Required
- **Content:** [Dataset/dcat:CatalogRecord](#dcatcatalogrecord)
- **Description:** This property contains information about the metadata record itself, as opposed to the resource the record describes. See Uses of dcat:CatalogRecord and https://github.com/Cross-Domain-Interoperability-Framework/Discovery/issues/13 for discussion on how to make assertion about the metadata record distinct from statements about the described resource. Use the dcat:CatalogRecord as additionalType to distinguish this schema:Dataset from the schema:Dataset about a described external resource. see <https://cross-domain-interoperability-framework.github.io/cdifbook/metadata/contentmodel.html#properties-for-metadata-management>. Introduction of this is novel for schema.org implementations.

## Action

[↑ Back to TOC](#table-of-contents)

### **@type**

- **Cardinality:** Required, Repeatable
- **Content:** string.uri
- **Description:** The rdf type default is 'Action', but any of these schema.org actions will validate: {Action, AssessAction, ConsumeAction, ControlAction, CreateAction, DeleteAction, FindAction, InteractAction, MoveAction, PlayAction, SearchAction, TransferAction, UpdateAction}

### **name**

- **Cardinality:** Required
- **Content:** string
- **Description:** text label for the action

### **target**

- **Cardinality:** Required
- **Content:** [EntryPoint](#entrypoint)
- **Description:** specifies the request target location and request syntax

### **result**

- **Cardinality:** Optional
- **Content:** [DataDownload](#data-download)
- **Description:** specifies the serialization scheme (encoding format, information model) for expected representation of the data

### **object**

- **Cardinality:** Optional
- **Content:** Thing
- **Description:** Specifies the resource that is the object (input) of the action. The value is an open ended class (Thing can be anything…) for general description of Actions. When schema:Action (or a subclass) is used to descibe operations for a WebAPI distribution (the normal CDIF usage), the object is implicitly the resource that is the subject of the containing metadata record, so this property would be superfluous.

### **query-input**

- **Cardinality:** Optional, Repeatable
- **Content:** [PropertyValueSpecification](#propertyvaluespecification)
- **Description:** set of explanations of the parameters in the URL template for the target EntryPoint.

# Optional Dataset properties

[↑ Back to TOC](#table-of-contents)

### **description**

- **Cardinality:** Optional
- **Content:** string
- **Description:** Abstract describing the content, format, origin, quality or any other aspects of the resource that might be useful to future users evaluating the resource for usage.

### **additionalType**

- **Cardinality:** Optional, Repeatable
- **Content:** string or [DefinedTerm](#defined-term)
- **Description:** Use to assert semantics for the JSON object using concepts from other vocabularies. Type assertions here are purely for semantic information, and do not imply presence of properties assigned to a class in some other vocabulary.

### **sameAs**

- **Cardinality:** Optional, Repeatable
- **Content:** string, [object reference](#object-reference), or [PropertyValue](#propertyvalue-identifier)
- **Description:** Other identifiers for the dataset, as IRI references, literal strings, or structured identifiers using schema:PropertyValue.

### **version**

- **Cardinality:** Optional
- **Content:** string or number
- **Description:** The version number or identifier for this dataset (text or numeric). The values should sort from oldest to newest using an alphanumeric sort on version strings. **Unlike schema:identifier, this designates one particular version** — the state that was actually used in some context, and therefore the one a reproducible result must cite; see [identifier and version identify different things](#identifier-and-version-identify-different-things).

### **inLanguage**

- **Cardinality:** Optional
- **Content:** string
- **Description:** The language of the dataset content. Use [ISO 639 code](https://www.loc.gov/standards/iso639-2/php/code_list.php) for language or language:locale

### **datePublished**

- **Cardinality:** Optional
- **Content:** string, [ISO 8601 format](https://en.wikipedia.org/wiki/ISO_8601)
- **Description:** ISO8601 formatted date (and optional time if relevant) when Dataset was made public.

### **relatedLink**

- **Cardinality:** Optional, Repeatable
- **Content:** [LinkRole](#linkrole)
- **Description:** links to related resources; linkRelationship specifies how the resource is related. Use schema.org LinkRole type for values, with a linkRelationship and target that documents the url and encoding format of the linked content.

### **publishingPrinciples**

- **Cardinality:** Optional, Repeatable
- **Content:** string, [object reference](#object-reference), or [LabeledLink](#labeled-link)
- **Description:** Policies related to maintenance, update, expected time to live, e.g. FDOF digitalObjectMutability, RDA digitalObjectPolicy, FDOF PersistencyPolicy. If an online resource documents the policies or a URI is used to identify the conditions, recommend using [LabeledLink](#labeled-link), implemented as schema:CreativeWork to provide a label (name) and an identifier (URI or URL).

### **keywords**

- **Cardinality:** Optional, Repeatable
- **Content:** string or [DefinedTerm](#defined-term)
- **Description:** Keywords are an array of strings, an array of schema:DefinedTerms, or some combination of these. If you have information about a controlled vocabulary from which keywords come from, use schema:DefinedTerm to descibe that keyword. This allowed variability complicates parsing the metadata record; recommend using DefinedTerm for all keywords if any of them are from a known vocabulary, otherwise an array of strings.

### **creator**

- **Cardinality:** Optional, Repeatable
- **Content:** List of [object reference](#object-reference), [Person](#person), or [Organization](#organization)
- **Description:** Author or orginator of intellectual content of dataset. Use the JSON-LD @list construct to preserve author order. Use contributor with the Role property to specify other roles related to creation or stewardship of the resource.

### **contributor**

- **Cardinality:** Optional, Repeatable
- **Content:** [object reference](#object-reference), [Person](#person), or [Organization](#organization)
- **Description:** Other parties who played a role in production of dataset

### **publisher**

- **Cardinality:** Optional
- **Content:** [object reference](#object-reference), [Person](#person), or [Organization](#organization)
- **Description:** Identify Party who made the dataset publicly available

### **provider**

- **Cardinality:** Optional, Repeatable
- **Content:** [object reference](#object-reference), [Person](#person), or [Organization](#organization)
- **Description:** Party who maintains the distribution options for the dataset (i.e. the hosting web server). If there are multiple distributions from different providers, use the provider property on distribution/DataDownload. Contact information for the provider is important if there are malfunctions in the data access workflow.

### **funding**

- **Cardinality:** Optional, Repeatable
- **Content:** [MonetaryGrant](#monetarygrant)
- **Description:** Acknowledgement for sources of financial or other material resources important for the creation of the described resource. Allows identification of specific funding instruments (grants, contracts, scholarships…) or institutions providing resources.

### **prov: wasGeneratedBy**

- **Cardinality:** Optional, Repeatable
- **Content:** prov:Activity/prov:used
- **Description:** For Discovery profile provide brief information about instruments, software or experimental protocols used, using the value of prov:used either a string or [object reference](#object-reference).

### **prov: wasDerivedFrom**

- **Cardinality:** Optional, Repeatable
- **Content:** string, [object reference](#object-reference), [LabeledLink](#labeled-link)
- **Description:** Brief information about sources of data used in aggregate datasets. String bibliographic citations, URIs as object references, or LabeledLink, implemented as schema:CreativeWork, to provide a title, description and URL.

# Other Classes used for CDIF Core

[↑ Back to TOC](#table-of-contents)

## Data Download

[↑ Back to TOC](#table-of-contents)

- file-based access to a resource via URL; the DataDownload object provides a link to get the resource content, along with information about the serialization format and conventions used.

### **@id**

- **Cardinality:** Optional
- **Content:** string:uri
- **Description:** Graph node identifiers are only necessary if the node content will be referenced in other places

### **@type**

- **Cardinality:** Required – 'schema:DataDownload', other types optional
- **Content:** string.uri
- **Description:** This is the rdf:type.

### **contentUrl**

- **Cardinality:** Required
- **Content:** string.uri
- **Description:** Expected to be an http uri that will directly GET the content of the resource described by this metadata record, in the format specified by the encodingFormat property, and conforming to any specifications identified in the dcterms:conformsTo property. Equivalent to dcat:accessURL.  A landing page url (dcat:landingPage) should be placed in the Dataset/schema:url property.

### **name**

- **Cardinality:** Optional
- **Content:** string
- **Description:** String to identify this download option in user interface

### **description**

- **Cardinality:** Optional
- **Content:** string
- **Description:** string providing information to document this download option in user interface

### **encodingFormat**

- **Cardinality:** Optional, Repeatable
- **Content:** string:MIME Type
- **Description:** Identifier for format from a registry

### **spdx:checksum**

- **Cardinality:** Optional
- **Content:** [spdx:Checksum](#spdxchecksum)
- **Description:** Checksum string that is 'footprint' of the described file to enable testing for file modification. Algorithm used is specified by spdx:algorithm property.

### **dcterms:conformsTo**

- **Cardinality:** Optional, Repeatable
- **Content:** [object reference](#object-reference)
- **Description:** An identifier for a specification that the distribution conforms to. Recommended to enable machine-actionable data access. The target download might conform to more that one profile specification.

### **provider**

- **Cardinality:** Optional, Repeatable
- **Content:** [object reference](#object-reference), [Person](#person), or [Organization](#organization)
- **Description:** The agent responsible for acces to the described resource. Use contact for this agent to report access problems.

## DataCatalog

[↑ Back to TOC](#table-of-contents)

- An accessible collection of data. The data might be metadata (about other resources) or datasets.

### **@type**

- **Cardinality:** Required – 'DataCatalog', Repeatable
- **Content:** string.uri

### **@id**

- **Cardinality:** Optional
- **Content:** string.uri
- **Description:** identifier for graph node.

### **name**

- **Cardinality:** Optional
- **Content:** string
- **Description:** Label for the data catalog.

### **url**

- **Cardinality:** Optional
- **Content:** string.url
- **Description:** Url to access catalog landing page.

### **identifier**

- **Cardinality:** Optional
- **Content:** string.uri or [PropertyValue-(identifier)](#propertyvalue-identifier)
- **Description:** Identifier for the data catalog.

## MonetaryGrant

[↑ Back to TOC](#table-of-contents)

### **@type**

- **Cardinality:** Required -- 'MonetaryGrant', Repeatable
- **Content:** string.uri

CHOICE (at least one of identifier, name, or funder

### **identifier**

- **Cardinality:** Required if no name or funder
- **Content:** string.uri or [PropertyValue-(identifier)](#propertyvalue-identifier)
- **Description:** identifier for a particular grant

### **name**

- **Cardinality:** Required if no identifer or funder
- **Content:** string
- **Description:** title of the grant

### **funder**

- **Cardinality:** Required if no identifier or name
- **Content:** [object reference](#object-reference), [Person](#person), or [Organization](#organization)

### **description**

- **Cardinality:** Optional
- **Content:** string
- **Description:** description of the funding or grant

## Organization

[↑ Back to TOC](#table-of-contents)

### **@id**

- **Cardinality:** Optional
- **Content:** string.uri
- **Description:** Identifier for this graph node. Useful to reference this Organization using object references if they appear more that once in the metadata record.

### **@type**

- **Cardinality:** Required – 'Organization', Repeatable
- **Content:** string.uri
- **Description:** rdf:type, list must include Organization, but other schema.org types can be added for more precision: FundingAgency, Consortium, Corporation, EducationalOrganization, FundingScheme, GovernmentOrganization, NGO, Project, ResearchOrganization, defined by enumeration in the schema.

### **name**

- **Cardinality:** Required if no identifier
- **Content:** string
- **Description:** Label for the Organization

### **identifier**

- **Cardinality:** Required if no name
- **Content:** string.uri or [PropertyValue-(identifier)](#propertyvalue-identifier)
- **Description:** identifier for organization

### **additionalType**

- **Cardinality:** Optional, Repeatable
- **Content:** string or [DefinedTerm](#defined-term)

### **alternateName**

- **Cardinality:** Optional
- **Content:** string
- **Description:** other labels by which the organization might be known

### **description**

- **Cardinality:** Optional
- **Content:** string

### **sameAs**

- **Cardinality:** Optional, Repeatable
- **Content:** string, [object reference](#object-reference)
- **Description:** other identifiers for the organization

## Person

[↑ Back to TOC](#table-of-contents)

- Object representing a person.

### **@id**

- **Cardinality:** Optional
- **Content:** string.uri
- **Description:** Identifier for this graph node. Useful to reference this Person using object references if they appear more that once in the metadata record.

### **@type**

- **Cardinality:** Required – 'Person', Repeatable
- **Content:** string.uri
- **Description:** rdf:type for this JSON-LD object.

### **name**

- **Cardinality:** Required if no identifier
- **Content:** string
- **Description:** Label for person that is meaningful for human users, should format consistently. Recommend 'Family Name, Given Name' format.

### **identifier**

- **Cardinality:** Required if no name
- **Content:** string.uri or [PropertyValue-(identifier)](#propertyvalue-identifier)
- **Description:** identifier for person, recommend ORCID

### **description**

- **Cardinality:** Optional
- **Content:** string
- **Description:** Other useful information about the person.

### **alternateName**

- **Cardinality:** Optional
- **Content:** string
- **Description:** Other names by which the person is known.

### **affiliation**

- **Cardinality:** Optional
- **Content:** [Organization](#organization)
- **Description:** Organization that the person is associated with.

### **contactPoint**

- **Cardinality:** Optional
- **Content:** [ContactPoint](#contactpoint)
- **Description:** email is required property if a contactPoint is included. Schema.org allows telephone and postal contacts as well.

### **sameAs**

- **Cardinality:** Optional, Repeatable
- **Content:** string, [object reference](#object-reference)
- **Description:** other identifiers for the person

## PropertyValueSpecification

[↑ Back to TOC](#table-of-contents)

- Description of the kind of value expected for a parameter value.

### **@type**

- **Cardinality:** Required – 'PropertyValueSpecification', repeatable

### **valueName**

- **Cardinality:** Required
- **Content:** string
- **Description:** This will be used to match the specification to parameters in a template string used to construct a query.

### **description**

- **Cardinality:** Required
- **Content:** string
- **Description:** Explanation of the purpose of the parameter, its range of values, datatype, etc.

### **valueRequired**

- **Cardinality:** Optional
- **Content:** boolean
- **Description:** Default is true. False if the specified parameter is not required to fill the template.

### **valuePattern**

- **Cardinality:** Optional
- **Content:** string
- **Description:** regular expression to validate values for template parameters.

## Web API

[↑ Back to TOC](#table-of-contents)

- Provides information to request data through a web accessible service endpoint. This implementation uses the schema.org Action to document url or url template and parameters. At this point, schema is set up for one action-- an HTTP Get that requests data. The url template parameters (in curly brackets '{}') specify query parameters to filter the source data, request particular output formats or other options offered by the interface.

### **serviceType**

- **Cardinality:** Required
- **Content:** string or [DefinedTerm](#defined-term)
- **Description:** Specify the kind of service. Ideally this should be a resolvable identifier. Currently there is no widely adopted registry for serviceType identifiers. Services might be defined at different levels of granularity, and classifications might focus on function, data formats, thematic content, security, or other aspects of the service definition. For interoperability, there must be an external arrangement between data providers and consumers on the strings that will be used to specify service types.

### **termsOfService**

- **Cardinality:** Required, Repeatable
- **Content:** string or [LabeledLink](#labeled-link)
- **Description:** Description of access privileges required to use the API, e.g. registration, licensing, payments. Note that access constraints applying to all distributions of the resource should be specified in the access constraints for the resource description as a whole.

### **documentation**

- **Cardinality:** Optional
- **Content:** string or [LabeledLink](#labeled-link)
- **Description:** A machine-actionable description of a service instance. Examples include OpenAPI documents, OGC Capabilities documents. Software designed to utilise a particular service type will typically include functionality to parse such a description document and engage with the service endpoint. If such a document is available for the service instance providing the resource distribution, it should be included in the distribution metadata.

### **potentialAction**

- **Cardinality:** Required, Repeatable
- **Content:** [object reference](#object-reference), [Action](#action)
- **Description:** Description of the operations offered by the interface.

# Data types used for CDIF Core

[↑ Back to TOC](#table-of-contents)

## ContactPoint

[↑ Back to TOC](#table-of-contents)

- Information about how to communicate with a person or organization. CDIF only includes e-mail in its schema.

### **@type**

- **Cardinality:** Required – 'ContactPoint', Repeatable
- **Content:** string.uri

### **email**

- **Cardinality:** Required
- **Content:** string
- **Description:** Property is required if a contactPoint property is included. Use missing@example.org if e-mail address is not available. Recommend using position-based contact point because people move around.

## Contributor

[↑ Back to TOC](#table-of-contents)

- For more granularity on how an agent contributed to a resource, use schema:Role. The schema.org documentation does not state that the Role type is an expected data type for the contributor property, but that is addressed in this blog post (http://blog.schema.org/2014/06/introducing-role.html). see also [ESIPfed Science on Schema.org roles of people note](https://github.com/ESIPFed/science-on-schema.org/blob/develop/guides/Dataset.md#roles-of-people).

### **@type**

- **Cardinality:** Required -- 'Role', Repeatable
- **Content:** string.uri
- **Description:** rdf:type

### **roleName**

- **Cardinality:** Required
- **Content:** string, [DefinedTerm](#defined-term)
- **Description:** term that specifies the relationship between the contributor and the described resource.

### **contributor**

- **Cardinality:** Required
- **Content:** [object reference](#object-reference), [Person](#person) or [Organization](#organization)

## dcat:CatalogRecord

[↑ Back to TOC](#table-of-contents)

- This is the class used to provide information about the metadata record itself.

### **@id**

- **Cardinality:** Required
- **Content:** string.uri
- **Description:** Identifier for the metadata record.

### **@type**

- **Cardinality:** Required – "Dataset", Repeatable
- **Content:** string.uri

### **additionalType**

- **Cardinality:** Required – "dcat:CatalogRecord", Repeatable
- **Content:** string

### **about**

- **Cardinality:** Required
- **Content:** [object reference](#object-reference)
- **Description:** This must be a reference to the metadata record that this node documents, using the @id of that record.

### **conformsTo**

- **Cardinality:** Required, Repeatable
- **Content:** [object reference](#object-reference)
- **Description:** Identifiers for conformance classes/profiles that the metadata record follows. For CDIF Core this must include 'https://w3id.org/cdif/core/1.1'; a record that also conforms to a higher-level profile (e.g. Discovery) lists that profile's identifier as well.

### **description**

- **Cardinality:** Optional
- **Content:** string
- **Description:** other information about the metadata record that might be useful.

### **maintainer**

- **Cardinality:** Optional
- **Content:** [Person](#person) or [Organization](#organization)
- **Description:** Identification of the agent that maintains the metadata, with contact information. Should include person name and affiliation, or position name and affiliation, or just organization name. e-mail address is preferred contact information.

### **sdDatePublished**

- **Cardinality:** Optional
- **Content:** ISO 8601 formatted date/datetime
- **Description:** date of most recent update to the metadata content

### **includedInDataCatalog**

- **Cardinality:** Optional
- **Content:** [DataCatalog](#datacatalog)
- **Description:** identify the source for the origin the metadata record

## Defined Term

[↑ Back to TOC](#table-of-contents)

### **@type**

- **Cardinality:** Required – 'DefinedTerm', Repeatable
- **Content:** string.uri

### **name**

- **Cardinality:** Required if no identifier or termCode
- **Content:** string
- **Description:** label for the term

### **identifier**

- **Cardinality:** Required if no name or termCode
- **Content:** string.uri or [PropertyValue-(identifier)](#propertyvalue-identifier)

### **termCode**

- **Cardinality:** Required if no name or identifier
- **Content:** string
- **Description:** A representative code for this keyword in the controlled vocabulary. Analogous to skos:Notation

### **inDefinedTermSet**

- **Cardinality:** Optional
- **Content:** string
- **Description:** Name for the controlled vocabulary responsible for this keyword.

## EntryPoint

[↑ Back to TOC](#table-of-contents)

- Use to document the URL that is the target for invoking an action, or that is the target object of a link relationship.

### **@type**

- **Cardinality:** Required – "EntryPoint", Repeatable
- **Content:** string. Uri

### **encodingFormat**

- **Cardinality:** Optional
- **Content:** string, MIME Type
- **Description:** MIME type / media type identifier for the representation returned from the URL.

### **name**

- **Cardinality:** Optional
- **Content:** string
- **Description:** Label for the resource located by the URL

### **url**

- **Cardinality:** Required
- **Content:** string.url
- **Description:** Locator that can be used to retrieve the target resource on the Web.

## Labeled Link

[↑ Back to TOC](#table-of-contents)

### **@type**

- **Cardinality:** Required – 'CreativeWork', Repeatable
- **Content:** string.uri

### **url**

- **Cardinality:** Required
- **Content:** string:uri
- **Description:** URL for web location to GET the resource

### **name**

- **Cardinality:** Optional
- **Content:** string
- **Description:** Label for the linked resource

### **description**

- **Cardinality:** Optional
- **Content:** string
- **Description:** Text description of the linked resource.

## LinkRole

[↑ Back to TOC](#table-of-contents)

- This is the type used for links that have an associated semantic conveyed by the linkRelationship.

### **@type**

- **Cardinality:** Required – 'LinkRole', Repeatable
- **Content:** string.uri

### **linkRelationship**

- **Cardinality:** Required
- **Content:** string or [DefinedTerm](#defined-term)
- **Description:** Term that specifies the relationship between the source and target of the link.

### **target**

- **Cardinality:** Required
- **Content:** [EntryPoint](#entrypoint)
- **Description:** URL for link target, along with a label and encoding format for the target resource.

## PropertyValue-(identifier)

[↑ Back to TOC](#table-of-contents)

### **@type**

- **Cardinality:** Required – 'PropertyValue', Repeatable
- **Content:** string.uri

### **value**

- **Cardinality:** Required if no url
- **Content:** string
- **Description:** the identifier string. E.g. 10.5066/F7VX0DMQ

### **url**

- **Cardinality:** Required if no value
- **Content:** string.url
- **Description:** web-resolveable string for the identifier; host name part is location of a resolver that will return some representation for the given identifier value. E.g. https://doi.org/10.5066/F7VX0DMQ

### **propertyID**

- **Cardinality:** Optional
- **Content:** string:uri
- **Description:** In this context for the schema:PropertyValue, this field is an identifier for the identifier schema, e.g. DOI, ARK. Get values from https://registry.identifiers.org/registry/ for interoperability

## spdx:Checksum

[↑ Back to TOC](#table-of-contents)

### **spdx:algorithm**

- **Cardinality:** Required
- **Content:** string
- **Description:** Name or identifier for the algorithm used to calculate the checksum.

### **spdx: checksumValue**

- **Cardinality:** Required
- **Content:** string
- **Description:** the checksum string.
