# Context for Spatial Memory System Development

## Project Overview
This notebook is part of a project to develop a spatial memory system for physical AI (robots). The system combines Discrete Global Grid Systems (DGGS), multimodal Large Language Models (LLMs), and Knowledge Graphs (KGs) to create a comprehensive spatial understanding and memory framework for robots operating in physical environments.

## Key Components

### 1. DGGS for Spatial Indexing
We're using the S2 Geometry library, an implementation of Discrete Global Grid Systems. S2 provides hierarchical spatial indexing through a quadtree-based approach that divides the Earth's surface into cells of varying resolution. This gives us:
- A consistent spatial reference framework
- Multi-resolution capabilities for different granularity needs
- Efficient spatial querying and neighborhood operations
- Stable cell identifiers that can serve as spatial indexes

### 2. Semantic Trajectories
We're implementing the Semantic Trajectory Ontology Design Pattern to structure how the robot records its movements through space. This allows us to:
- Record meaningful "fixes" (points with timestamps)
- Connect fixes with segments containing semantic attributes
- Attach observations and context to specific points in the trajectory
- Query past movements and locations efficiently

### 3. Multimodal LLM Integration
We're using multimodal LLMs (particularly Claude) to process visual information from the robot's camera and extract:
- Scene descriptions and classifications
- Object detection and identification
- Spatial relationships between objects
- Semantic context for locations

### 4. Knowledge Graph Representation
We're using RDFlib to create a knowledge graph that connects:
- Spatial locations (via S2 cells)
- Trajectory data (fixes, segments)
- Visual observations
- Extracted semantic information
- Vector embeddings of observations

### 5. Vector Storage with Polars and Pandas
We're using Polars and Pandas DataFrames instead of SQLite-vec to store vector embeddings and related metadata. This provides:
- Fast, in-memory processing
- Easy integration with data science tools
- Flexible querying capabilities
- Efficient serialization to disk (via Parquet)
- KGLab integration for storing RDFlib graphs as Parquet files

## Technical Approach
We're adopting a literate, exploratory programming approach similar to fast.ai's methodology:
- Start small with working pieces and gradually integrate
- Test each component thoroughly with real examples
- Build incrementally with clear explanations of code and reasoning
- Document as we go for better understanding and future reference
- Package the final system using nbdev for easy distribution

## Project Goals
- Create a reusable framework for spatial memory in robotics
- Enable robots to remember and reason about places they've visited
- Support both exact (spatial) and approximate (similarity-based) memory retrieval
- Facilitate semantic understanding of environments
- Create a foundation for more advanced spatial AI capabilities

## Tools and Libraries
- S2sphere: Python binding for S2 Geometry
- RDFlib: Python library for working with RDF and knowledge graphs
- KGLab: For storing RDFlib graphs as Parquet files
- Anthropic API: For accessing Claude's vision capabilities
- Polars/Pandas: For DataFrame-based vector storage and processing
- Sentence-Transformers: For generating embeddings
- Folium: For visualization of spatial data
- nbdev: For turning notebooks into Python packages with documentation

This notebook is part of an incremental exploration to build the system step by step, with each stage working toward a complete integration of all components.