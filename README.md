# Knowledge Graph with LangChain and Neo4j

A demonstration of building and querying knowledge graphs using LangChain, Neo4j graph database, and LLMs. This project shows how to convert unstructured text into structured graph representations and query them using natural language.

## Overview

This project demonstrates two approaches to working with knowledge graphs:

1. **Text-to-Graph Transformation**: Converting unstructured text into graph documents using LLM-powered extraction
2. **Natural Language Querying**: Using GraphCypherQAChain to query Neo4j graphs with natural language questions

Knowledge graphs enable semantic understanding and relationship mapping, making them ideal for complex data relationships, question-answering systems, and recommendation engines.

## Features

- Convert unstructured text to structured graph documents using LLMGraphTransformer
- Automatic entity and relationship extraction from text
- Neo4j graph database integration
- Natural language to Cypher query conversion
- Movie dataset example with actors, directors, and genres
- Question-answering over graph data

## Prerequisites

- Python 3.8+
- Neo4j database instance (local or cloud)
- Groq API key (or compatible LLM API)
- Basic understanding of graph databases and Cypher query language

## Installation

1. Clone the repository:
```bash
git clone <your-repo-url>
cd <repo-name>
```

2. Create a virtual environment:
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

3. Install dependencies:
```bash
pip install -r requirements.txt
```

4. Create a `.env` file in the project root:
```env
API_KEY=your_groq_api_key_here
NEO4J_URI=neo4j+s://your-neo4j-instance
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your_neo4j_password
```

## Neo4j Setup

### Option 1: Neo4j AuraDB (Cloud - Recommended for beginners)
1. Go to [neo4j.com/cloud/aura](https://neo4j.com/cloud/aura/)
2. Sign up for a free account
3. Create a new instance
4. Save the connection URI, username, and password

### Option 2: Local Neo4j
1. Download Neo4j Desktop from [neo4j.com/download](https://neo4j.com/download/)
2. Install and create a local database
3. Start the database and note the connection details

## Project Structure

```
.
├── knowledgegraph.ipynb    # Main Jupyter notebook
├── requirements.txt        # Python dependencies
├── .env                    # Environment variables (not tracked)
└── README.md              # This file
```

## Usage

### Running the Notebook

Open and run `knowledgegraph.ipynb` in Jupyter:

```bash
jupyter notebook knowledgegraph.ipynb
```

### Key Components

**1. Initialize Neo4j Graph Connection**
```python
from langchain_community.graphs import Neo4jGraph

graph = Neo4jGraph(
    url=NEO4J_URI,
    username=NEO4J_USERNAME,
    password=NEO4J_PASSWORD
)
```

**2. Initialize LLM**
```python
from langchain_groq import ChatGroq

llm = ChatGroq(groq_api_key=API_KEY, model_name="gemma2-9b-it")
```

**3. Convert Text to Graph Documents**
```python
from langchain_experimental.graph_transformers import LLMGraphTransformer
from langchain_core.documents import Document

# Create document from text
text = "Elon Musk is the CEO of Tesla and founder of SpaceX..."
documents = [Document(page_content=text)]

# Transform to graph
llm_transformer = LLMGraphTransformer(llm=llm)
graph_documents = llm_transformer.convert_to_graph_documents(documents)

# Extract nodes and relationships
nodes = graph_documents[0].nodes
relationships = graph_documents[0].relationships
```

**4. Load Data into Neo4j**
```python
# Example: Loading movie dataset
movie_query = """
LOAD CSV WITH HEADERS FROM
'https://raw.githubusercontent.com/tomasonjo/blog-datasets/main/movies/movies_small.csv' 
as row
MERGE(m:Movie{id:row.movieId})
SET m.released = date(row.released),
    m.title = row.title,
    m.imdbRating = toFloat(row.imdbRating)
...
"""

graph.query(movie_query)
```

**5. Query with Natural Language**
```python
from langchain.chains import GraphCypherQAChain

chain = GraphCypherQAChain.from_llm(
    llm=llm,
    graph=graph,
    allow_dangerous_requests=True,
    verbose=True
)

response = chain.invoke({"query": "List the actors in Toy Story?"})
print(response['result'])
# Output: Jim Varney, Tim Allen, Tom Hanks, Don Rickles
```

## How It Works

### Text-to-Graph Transformation

The LLMGraphTransformer uses an LLM to:
1. **Identify entities** (People, Companies, Universities, etc.)
2. **Extract relationships** (CEO_OF, FOUNDED, ATTENDED, etc.)
3. **Create structured graph** with nodes and edges

Example transformation:
```
Text: "Elon Musk is the CEO of Tesla"
↓
Node: Person(name="Elon Musk")
Node: Company(name="Tesla")
Relationship: (Elon Musk)-[CEO]->(Tesla)
```

### Natural Language to Cypher

The GraphCypherQAChain:
1. Takes a natural language question
2. Converts it to Cypher query using the LLM
3. Executes the query against Neo4j
4. Formats the results into a natural language answer

Example flow:
```
Question: "Who acted in Toy Story?"
↓
Cypher: MATCH (m:Movie {title: "Toy Story"})<-[:ACTED_IN]-(p:Person) RETURN p.name
↓
Results: [Jim Varney, Tim Allen, Tom Hanks, Don Rickles]
↓
Answer: "Jim Varney, Tim Allen, Tom Hanks, Don Rickles acted in Toy Story."
```

## Graph Schema

The movie dataset creates the following schema:

**Nodes:**
- `Person`: actors and directors with `name` and `born` properties
- `Movie`: films with `id`, `title`, `released`, and `imdbRating` properties
- `Genre`: movie genres with `name` property
- `Role`: job roles with `jobtitle` and `appointed` properties

**Relationships:**
- `(:Person)-[:DIRECTED]->(:Movie)`
- `(:Person)-[:ACTED_IN]->(:Movie)`
- `(:Movie)-[:IN_GENRE]->(:Genre)`
- `(:Person)-[:WORKS_AS]->(:Role)`
- `(:Person)-[:SUPERVISOR]->(:Person)`

## Example Queries

Try these natural language queries:

```python
# Find actors in a movie
chain.invoke({"query": "Who acted in The Matrix?"})

# Find directors
chain.invoke({"query": "Who directed Toy Story?"})

# Find movies by genre
chain.invoke({"query": "List movies in the Action genre"})

# Find movie ratings
chain.invoke({"query": "What is the IMDB rating of Inception?"})
```

## Dependencies

```
langchain              # LangChain framework
langchain-community    # Community integrations
langchain-groq        # Groq LLM integration
langchain-experimental # Experimental features (LLMGraphTransformer)
neo4j                 # Neo4j Python driver
python-dotenv         # Environment variable management
```

## Configuration

### LLMGraphTransformer Options

You can customize entity and relationship extraction:

```python
llm_transformer = LLMGraphTransformer(
    llm=llm,
    allowed_nodes=["Person", "Company", "Location"],  # Limit node types
    allowed_relationships=["WORKS_AT", "LOCATED_IN"],  # Limit relationships
)
```

### GraphCypherQAChain Options

```python
chain = GraphCypherQAChain.from_llm(
    llm=llm,
    graph=graph,
    verbose=True,              # Show generated Cypher queries
    return_intermediate_steps=True,  # Return query steps
    allow_dangerous_requests=True,   # Required for write operations
)
```

## Security Considerations

⚠️ **Important**: The `allow_dangerous_requests=True` flag allows the LLM to generate any Cypher query, including write operations. In production:

- Use read-only database users for querying
- Implement query validation and sanitization
- Consider using `GraphCypherQAChain` with restricted permissions
- Monitor and log all generated queries

## Troubleshooting

**Connection issues to Neo4j:**
- Verify your Neo4j instance is running
- Check the URI format (should include protocol: `neo4j://` or `neo4j+s://`)
- Ensure firewall rules allow the connection
- Verify credentials are correct

**LLMGraphTransformer not extracting entities:**
- The text may be too short or ambiguous
- Try providing more context in the text
- Experiment with different LLM models
- Check if the LLM has rate limits

**Cypher query errors:**
- Enable `verbose=True` to see generated queries
- Verify the graph schema matches expectations using `graph.schema`
- Check Neo4j query logs for detailed error messages

**Deprecation warnings:**
- The `langchain_community.graphs.Neo4jGraph` is deprecated
- Consider migrating to `langchain-neo4j` package when available
- Current implementation still works but may need updates in future versions

## Advanced Usage

### Custom Graph Transformations

```python
# Add metadata to nodes
from langchain_experimental.graph_transformers import LLMGraphTransformer

transformer = LLMGraphTransformer(
    llm=llm,
    node_properties=["description", "category"],
    relationship_properties=["strength", "since"]
)
```

### Querying with Filters

```python
# Complex query with conditions
response = chain.invoke({
    "query": "List action movies released after 2020 with rating above 7.0"
})
```

## Best Practices

1. **Chunk large texts** before transformation to avoid token limits
2. **Validate graph structure** after transformation
3. **Use specific entity types** for better relationship extraction
4. **Index frequently queried properties** in Neo4j for performance
5. **Test Cypher queries manually** before deploying to production

## Resources

- [Neo4j Graph Database](https://neo4j.com/)
- [LangChain Documentation](https://python.langchain.com/)
- [Cypher Query Language Guide](https://neo4j.com/docs/cypher-manual/)
- [Graph Data Science](https://neo4j.com/docs/graph-data-science/)

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Acknowledgments

- [LangChain](https://python.langchain.com/) for the graph transformation and QA framework
- [Neo4j](https://neo4j.com/) for the graph database
- [Groq](https://groq.com/) for fast LLM inference
- Movie dataset from [tomasonjo/blog-datasets](https://github.com/tomasonjo/blog-datasets)