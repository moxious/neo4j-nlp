# GraphAware Natural Language Processing

[![Build Status](https://travis-ci.org/graphaware/neo4j-nlp.svg?branch=master)](https://travis-ci.org/graphaware/neo4j-nlp)
This [Neo4j](https://neo4j.com) plugin offers Graph Based Natural Language Processing capabilities.

The main module, this module, provide a common interface for underlying text processors as well as a
**Domain Specific Language** built atop stored procedures and functions making your Natural Language Processing
workflow developer friendly.

It comes in 2 versions, Community (open-sourced) and Enterprise with the following NLP features :

## Feature Matrix

| | Community Edition | Enterprise Edition |
| --- | :---: | :---: |
| Text information Extraction | ✔ | ✔ |
| ConceptNet5 Enricher | ✔ | ✔ |
| Keyword Extraction | ✔ | ✔ |
| Topics Extraction | ✔ | ✔ |
| Similarity Computation | ✔ | ✔ |
| Apache Spark Binding for Distributed Algorithms | | ✔ |
| User Interface | | ✔ |
| ML Prediction capalities | | ✔ |
| Entity Merging | | ✔ |
| Questions generator | | ✔ |
| Conversational Features | | ✔ |

Two NLP processor implementations are available, respectively [OpenNLP](https://github.com/graphaware/neo4j-nlp-opennlp) and
[Stanford NLP](https://github.com/graphaware/neo4j-nlp-stanfordnlp).


## Installation

From the [GraphAware plugins directory](https://products.graphaware.com), download the following `jar` files :

* `neo4j-framework` (the JAR for this is labeled "graphaware-server-enterprise-all")
* `neo4j-nlp`
* `neo4j-nlp-stanfordnlp` or `neo4j-nlp-opennlp` or both

and copy them in the `plugins` directory of Neo4j.

*Take care that the version numbers of the framework you are using match with the version of Neo4J
you are using*.  This is a common setup problem.  For example, if you are using Neo4j 3.3.0, all
of the JARs you download should contain 3.3 in their version number.

`plugins/` directory example :

```
-rw-r--r--  1 abc  staff   6108799 May 16 11:27 graphaware-nlp-1.0-SNAPSHOT.jar
-rw-r--r--@ 1 abc  staff  13391931 May  5 09:34 graphaware-server-enterprise-all-3.1.3.47.jar
-rw-r--r--  1 abc  staff  46678477 May 16 14:59 nlp-opennlp-1.0-SNAPSHOT.jar
```

Append the following configuration in the `neo4j.conf` file in the `config/` directory:

```
  dbms.unmanaged_extension_classes=com.graphaware.server=/graphaware
  com.graphaware.runtime.enabled=true
  com.graphaware.module.NLP.1=com.graphaware.nlp.module.NLPBootstrapper
  dbms.security.procedures.whitelist=ga.nlp.*
```

Start or restart your Neo4j database.

Note: both concrete text processors are quite greedy - you will need to dedicate sufficient memory for to Neo4j heap space.

Additionally, the following indexes and constraints are suggested to speed performance:

```
CREATE CONSTRAINT ON (a:Tag) ASSERT a.id IS UNIQUE;
CREATE INDEX ON :Tag(a.value);
```

## Quick Documentation in Neo4j Browser

Once the extension is loaded, you can see basic documentation on all available procedures by running
this Cypher query:

```
CALL dbms.procedures() YIELD name, signature, description
WHERE name =~ 'ga.nlp.*'
RETURN name, signature, description ORDER BY name asc;
```

## Getting Started

### Text extraction

#### Pipelines and components

The text extraction phase is done with a Natural Language Processing pipeline, each pipeline has a list of enabled components.

For example, the basic `tokenizer` pipeline has the following components :


* Sentence Segmentation
* Tokenization
* StopWords Removal
* Stemming
* Part Of Speech Tagging
* Named Entity Recognition


##### Example

Let's take the following text as example :

```
Scores of people were already lying dead or injured inside a crowded Orlando nightclub,
and the police had spent hours trying to connect with the gunman and end the situation without further violence.
But when Omar Mateen threatened to set off explosives, the police decided to act, and pushed their way through a
wall to end the bloody standoff.
```

**Simulate your original corpus**

Create a node with the text, this node will represent your original corpus or knowledge graph :

```
CREATE (n:News)
SET n.text = "Scores of people were already lying dead or injured inside a crowded Orlando nightclub,
and the police had spent hours trying to connect with the gunman and end the situation without further violence.
But when Omar Mateen threatened to set off explosives, the police decided to act, and pushed their way through a
wall to end the bloody standoff.";
```

**Perform the text information extraction**

The extraction is done via the `annotate` procedure which is the entry point to text information extraction

```
MATCH (n:News)
CALL ga.nlp.annotate({text: n.text, id: id(n)})
YIELD result
MERGE (n)-[:HAS_ANNOTATED_TEXT]->(result)
RETURN result
```

This procedure will create many linked nodes attached to your `:News` node, breaking down the language
into words, parts of speech, and functions.  This analysis of the text acts as a starting point for the
later steps.

### Enrich your original knowledge

As of now, a single enricher is available, making use of the ConceptNet5 API.

This enricher will extend the meaning of tokens (Tag nodes) in the graph.

```
MATCH (n:Tag)
CALL ga.nlp.enrich.concept({tag: n, depth:2, admittedRelationships:["IsA","PartOf"]})
YIELD result
RETURN result
```

Tags have now a `IS_RELATED_TO` relationships to other enriched concepts.

List of procedures available:

### Sentiment Detection

You can also determine whether the text presented is positive, negative, or neutral.  This procedure
requires an AnnotatedText node, which is produced by `ga.nlp.annotate` above.

```
MATCH (t:MyNode)-[]-(a:AnnotatedText) 
CALL ga.nlp.sentiment(a) YIELD result 
RETURN result;
```

This procedure will simply return "SUCCESS" when it is successful, but it will apply the `:POSITIVE`, 
`:NEUTRAL` or `:NEGATIVE` label to each Sentence.  As a result, when sentiment detection is complete,
you can query for the sentiment of sentences as such:

```
MATCH (s:Sentence)
RETURN s.text, labels(s)
```

**5. Language Detection**

```
CALL ga.nlp.detectLanguage("What language is this in?") 
YIELD result return result
```

**6. NLP based filter**

```
CALL ga.nlp.filter({text:'On 8 May 2013,
    one week before the Pakistani election, the third author,
    in his keynote address at the Sentiment Analysis Symposium, 
    forecast the winner of the Pakistani election. The chart
    in Figure 1 shows varying sentiment on the candidates for 
    prime minister of Pakistan in that election. The next day, 
    the BBC’s Owen Bennett Jones, reporting from Islamabad, wrote 
    an article titled Pakistan Elections: Five Reasons Why the 
    Vote is Unpredictable, in which he claimed that the election 
    was too close to call. It was not, and despite his being in Pakistan, 
    the outcome of the election was exactly as we predicted.', filter: 'Owen Bennett Jones/PERSON, BBC, Pakistan/LOCATION'}) YIELD result 
return result
```

**7. Cosine similarity computation**

Once tags are extracted from all the news or other nodes containing some text, it is possible to compute similarities between them using content based similarity. 
During this process, each annotated text is described using the TF-IDF encoding format. TF-IDF is an established technique from the field of information retrieval and stands for Term Frequency-Inverse Document Frequency. 
Text documents can be TF-IDF encoded as vectors in a multidimensional Euclidean space. The space dimensions correspond to the tags, previously extracted from the documents. The coordinates of a given document in each dimension (i.e., for each tag) are calculated as a product of two sub-measures: term frequency and inverse document frequency.

```
CALL ga.nlp.ml.cosine.compute({}) YIELD result
```

