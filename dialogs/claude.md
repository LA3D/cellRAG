# Spatial Memory for Physical AI using DGGS and LLMs
## 🧑 human _(Feb 27, 2025, 08:09 AM)_
Ok. I want to experiment with dggs as a method of GraphRAG for storing spatial memory for a physical-ai (robot) based on using LLMs, and reasoning LLMs. I think the idea would be to use multimodal LLMs and the robot's camera to create "semantic understanding" of the space. We would use sqllite-vec to store the vector representations and dggs to store the spatial KG. The semantic trajectory ontology design pattern can be leveraged to describe the robots trajectory through a space and structure retrieval from the spatial memory. The vector and KG memorys need to be linked. I want to prototype this using a exploratory, literate programming approach like fast.ai's approach to software development.
## 🤖 assistant _(Feb 27, 2025, 08:10 AM)_
# Spatial Memory for Physical AI Using DGGS and LLMs

Your idea to combine Discrete Global Grid Systems (DGGS) with LLMs for robotic spatial memory is fascinating. This approach could create a powerful spatial knowledge representation system that leverages both structured spatial data and semantic understanding.

Let me outline a prototype architecture for developing this system using an exploratory, literate programming approach similar to fast.ai's methodology.

## Core Components

1. **DGGS-based Spatial Knowledge Graph**
- Using S2 or H3 hierarchical discrete global grid system
- Spatial indexing of observations and trajectories
- KG representation of spatial relationships

2. **Semantic Trajectory Pattern Implementation**
- Recording robot movement paths
- Segmenting trajectories with meaningful fixes
- Attaching semantic context to locations

3. **Multimodal LLM Processing**
- Scene understanding from camera input
- Text descriptions of visual observations
- Semantic labeling of detected objects and spaces

4. **Vector-KG Integration**
- SQLite-vec for embedding storage
- Links between vector embeddings and KG nodes
- Hybrid retrieval mechanisms

## Implementation Approach

### 1. Setting Up the DGGS Framework

First, we need to set up the DGGS system as our spatial indexing framework. Based on the documents, S2 Geometry seems like a good choice due to its hierarchical structure and support for multi-resolution analysis:

```python
# Install necessary packages
!pip install s2sphere sqlitedict sentence-transformers

import s2sphere
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

# Define helper functions for S2 cell operations
def latlong_to_s2cell(lat, lng, level=13):
"""Convert latitude/longitude to S2 cell ID at specified level"""
p = s2sphere.LatLng.from_degrees(lat, lng)
cell = s2sphere.CellId.from_lat_lng(p).parent(level)
return cell

def s2cell_to_polygon(cell_id):
"""Get polygon vertices for an S2 cell"""
cell = s2sphere.Cell(cell_id)
vertices = []
for i in range(4):
vertex = cell.get_vertex(i)
ll = s2sphere.LatLng.from_point(vertex)
vertices.append((ll.lat().degrees, ll.lng().degrees))
return vertices
```

### 2. Implementing the Semantic Trajectory Pattern

We'll implement the semantic trajectory ontology design pattern to structure the robot's movement path:

```python
class Fix:
def __init__(self, lat, lng, timestamp, s2_level=13):
self.location = s2sphere.LatLng.from_degrees(lat, lng)
self.timestamp = timestamp
self.s2cell = latlong_to_s2cell(lat, lng, s2_level)
self.attributes = {}  # Store additional semantic attributes
self.visual_embedding = None  # Store visual embedding from LLM
self.source = "robot_camera"  # Source of the fix

class Segment:
def __init__(self, start_fix, end_fix):
self.start_fix = start_fix
self.end_fix = end_fix
self.attributes = {}  # Store segment attributes like speed, etc.
self.traversed_by = "robot"  # Moving object

class SemanticTrajectory:
def __init__(self, name="robot_path"):
self.name = name
self.fixes = []
self.segments = []

def add_fix(self, fix):
if len(self.fixes) > 0:
# Create a segment between the previous fix and the new one
segment = Segment(self.fixes[-1], fix)
self.segments.append(segment)
self.fixes.append(fix)

def get_s2cells(self, level=None):
"""Get all S2 cells traversed in this trajectory"""
cells = set()
for fix in self.fixes:
if level is not None:
cells.add(fix.s2cell.parent(level))
else:
cells.add(fix.s2cell)
return cells
```

### 3. Creating a Vector-KG Storage System

Now we'll implement a storage system that combines the vector database with the knowledge graph:

```python
import sqlite3
import json
from sqlitedict import SqliteDict
from sentence_transformers import SentenceTransformer

class SpatialMemory:
def __init__(self, db_path="spatial_memory.db", model_name="clip-ViT-B-32"):
# Initialize vector embedding model
self.model = SentenceTransformer(model_name)

# Connect to SQLite
self.conn = sqlite3.connect(db_path)
self.cursor = self.conn.cursor()

# Create tables
self.cursor.execute('''
CREATE TABLE IF NOT EXISTS s2cells (
cell_id TEXT PRIMARY KEY,
level INTEGER,
parent_cell_id TEXT,
polygon TEXT
)
''')

self.cursor.execute('''
CREATE TABLE IF NOT EXISTS fixes (
fix_id TEXT PRIMARY KEY,
cell_id TEXT,
timestamp TEXT,
attributes TEXT,
FOREIGN KEY (cell_id) REFERENCES s2cells (cell_id)
)
''')

self.cursor.execute('''
CREATE TABLE IF NOT EXISTS segments (
segment_id TEXT PRIMARY KEY,
start_fix_id TEXT,
end_fix_id TEXT,
attributes TEXT,
FOREIGN KEY (start_fix_id) REFERENCES fixes (fix_id),
FOREIGN KEY (end_fix_id) REFERENCES fixes (fix_id)
)
''')

# Initialize vector storage
self.vector_db = SqliteDict(db_path, tablename='vector_embeddings', autocommit=True)

self.conn.commit()

def store_s2cell(self, cell_id):
"""Store an S2 cell in the database"""
cell = s2sphere.CellId(cell_id)
level = cell.level()
parent = cell.parent(level-1) if level > 0 else None
parent_id = str(parent.id()) if parent else None
polygon = json.dumps(s2cell_to_polygon(cell))

self.cursor.execute(
"INSERT OR REPLACE INTO s2cells VALUES (?, ?, ?, ?)",
(str(cell_id.id()), level, parent_id, polygon)
)
self.conn.commit()

def store_fix(self, fix):
"""Store a fix in the database with its vector embedding"""
fix_id = f"fix_{fix.timestamp.isoformat()}"
cell_id = str(fix.s2cell.id())

# Store the S2 cell if not already stored
self.store_s2cell(fix.s2cell)

# Store fix metadata
self.cursor.execute(
"INSERT OR REPLACE INTO fixes VALUES (?, ?, ?, ?)",
(fix_id, cell_id, fix.timestamp.isoformat(), json.dumps(fix.attributes))
)

# Store visual embedding if available
if fix.visual_embedding is not None:
self.vector_db[fix_id] = fix.visual_embedding

self.conn.commit()
return fix_id

def store_segment(self, segment, start_fix_id, end_fix_id):
"""Store a segment in the database"""
segment_id = f"segment_{start_fix_id}_{end_fix_id}"

self.cursor.execute(
"INSERT OR REPLACE INTO segments VALUES (?, ?, ?, ?)",
(segment_id, start_fix_id, end_fix_id, json.dumps(segment.attributes))
)
self.conn.commit()
return segment_id

def store_trajectory(self, trajectory):
"""Store an entire semantic trajectory"""
for i, fix in enumerate(trajectory.fixes):
fix_id = self.store_fix(fix)

if i > 0:
prev_fix_id = f"fix_{trajectory.fixes[i-1].timestamp.isoformat()}"
self.store_segment(trajectory.segments[i-1], prev_fix_id, fix_id)

def query_by_cell(self, cell_id):
"""Query all fixes within an S2 cell"""
self.cursor.execute(
"SELECT * FROM fixes WHERE cell_id = ?",
(str(cell_id.id()),)
)
return self.cursor.fetchall()

def query_by_vector_similarity(self, query_embedding, top_k=5):
"""Find the most similar embeddings in vector space"""
# Simple implementation - could be optimized
results = []
for fix_id, embedding in self.vector_db.items():
similarity = np.dot(query_embedding, embedding)
results.append((fix_id, similarity))

# Sort by similarity and return top_k
results.sort(key=lambda x: x[1], reverse=True)
return results[:top_k]

def hybrid_query(self, cell_id, query_embedding, top_k=5):
"""Hybrid query using both spatial and vector similarity"""
# Get all fixes in the specified S2 cell
fixes_in_cell = self.query_by_cell(cell_id)
fix_ids = [f[0] for f in fixes_in_cell]

# Find vectors for these fixes and calculate similarities
results = []
for fix_id in fix_ids:
if fix_id in self.vector_db:
embedding = self.vector_db[fix_id]
similarity = np.dot(query_embedding, embedding)
results.append((fix_id, similarity))

# Sort by similarity and return top_k
results.sort(key=lambda x: x[1], reverse=True)
return results[:top_k]
```

### 4. Multimodal LLM Integration for Scene Understanding

Now, let's integrate a multimodal LLM to process camera images and extract semantic understanding:

```python
from PIL import Image
import requests
import io
import base64
import json

class MultimodalProcessor:
def __init__(self, api_key=None, model="gpt-4-vision"):
self.api_key = api_key
self.model = model

def process_image(self, image_path, prompt=None):
"""Process an image with a multimodal LLM to extract semantic information"""
# This is a simplified placeholder - actual implementation would call your LLM API
# Placeholder for demonstration

# For a real implementation with OpenAI's GPT-4 Vision:
"""
headers = {
"Content-Type": "application/json",
"Authorization": f"Bearer {self.api_key}"
}

# Read and encode image
with open(image_path, "rb") as image_file:
encoded_image = base64.b64encode(image_file.read()).decode('utf-8')

# Create payload
payload = {
"model": self.model,
"messages": [
{
"role": "user",
"content": [
{"type": "text", "text": prompt or "What do you see in this image? Describe the objects, spatial layout, and any notable features."},
{"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{encoded_image}"}}
]
}
],
"max_tokens": 500
}

# Make request
response = requests.post(
"https://api.openai.com/v1/chat/completions",
headers=headers,
json=payload
)
result = response.json()
return result["choices"][0]["message"]["content"]
"""

# Placeholder implementation for demonstration
return {
"description": "A room with a desk, chair, and bookshelf. The desk has a computer on it.",
"objects": ["desk", "chair", "bookshelf", "computer"],
"spatial_relations": [
{"subject": "computer", "relation": "on", "object": "desk"},
{"subject": "chair", "relation": "near", "object": "desk"}
]
}

def generate_embedding(self, image_path):
"""Generate embedding for an image using the sentence transformer model"""
# In a real implementation, you'd extract features from image using your model
# This is a simplified placeholder - would use CLIP or similar

image = Image.open(image_path)
# Process image and generate embedding
# Placeholder for demonstration
return np.random.rand(512)  # Random embedding for demo
```

### 5. Robotics Integration

Now, let's implement a simple robot class that ties everything together:

```python
from datetime import datetime
import uuid
import time

class SpatialRobot:
def __init__(self, memory_db_path="robot_memory.db"):
self.spatial_memory = SpatialMemory(db_path=memory_db_path)
self.multimodal_processor = MultimodalProcessor()
self.current_trajectory = SemanticTrajectory()
self.position = (0, 0)  # (lat, lng)

def move_to(self, lat, lng):
"""Move the robot to a new position"""
self.position = (lat, lng)
timestamp = datetime.now()

# Create a new fix at this position
fix = Fix(lat, lng, timestamp)

# Add it to the current trajectory
self.current_trajectory.add_fix(fix)

print(f"Moved to position: {lat}, {lng}")
return fix

def observe_environment(self, camera_image_path):
"""Process a camera image and update the current fix with semantic information"""
if not self.current_trajectory.fixes:
print("No current position fix. Move the robot first.")
return

current_fix = self.current_trajectory.fixes[-1]

# Process the image with multimodal LLM
scene_understanding = self.multimodal_processor.process_image(camera_image_path)

# Generate visual embedding
visual_embedding = self.multimodal_processor.generate_embedding(camera_image_path)

# Update the current fix with scene understanding and embedding
current_fix.attributes["scene"] = scene_understanding
current_fix.visual_embedding = visual_embedding

# Store the updated fix in spatial memory
self.spatial_memory.store_fix(current_fix)

print(f"Processed observation at position: {self.position}")
return scene_understanding

def save_trajectory(self):
"""Save the current trajectory to spatial memory"""
self.spatial_memory.store_trajectory(self.current_trajectory)
print(f"Saved trajectory with {len(self.current_trajectory.fixes)} fixes")

def query_similar_observations(self, image_path, top_k=3):
"""Find similar observations based on visual similarity"""
query_embedding = self.multimodal_processor.generate_embedding(image_path)
results = self.spatial_memory.query_by_vector_similarity(query_embedding, top_k)
return results

def query_area(self, lat, lng, level=13):
"""Query observations in a specific area"""
cell = latlong_to_s2cell(lat, lng, level)
results = self.spatial_memory.query_by_cell(cell)
return results

def hybrid_search(self, image_path, lat, lng, level=13, top_k=3):
"""Perform a hybrid search using both spatial and visual similarity"""
query_embedding = self.multimodal_processor.generate_embedding(image_path)
cell = latlong_to_s2cell(lat, lng, level)
results = self.spatial_memory.hybrid_query(cell, query_embedding, top_k)
return results
```

### 6. Creating a Demo Notebook

Let's design a demo notebook to showcase the literate programming approach:

```python
# Spatial Memory for Physical AI Demo
# ===================================
#
# This notebook demonstrates a prototype of a spatial memory system for robots
# using DGGS and multimodal LLMs.

import numpy as np
import matplotlib.pyplot as plt
from IPython.display import Image, display
import folium

# Initialize our robot
robot = SpatialRobot()

# Simulate robot exploration
# --------------------------
#
# Let's simulate a robot exploring an indoor environment by moving to different
# positions and making observations.

print("Starting robot exploration...")

# Define a simple path for the robot
waypoints = [
(37.7749, -122.4194),  # San Francisco
(37.7746, -122.4172),
(37.7751, -122.4168),
(37.7755, -122.4175)
]

# Move the robot along the path
for lat, lng in waypoints:
robot.move_to(lat, lng)

# Simulate taking a photo (in a real implementation, this would capture from camera)
# For this demo, we'll use placeholder images
image_path = "placeholder.jpg"  # You would replace this with actual camera capture

# Process the observation
scene_info = robot.observe_environment(image_path)

print(f"Observation at ({lat}, {lng}):")
print(f"  Description: {scene_info['description']}")
print(f"  Objects detected: {', '.join(scene_info['objects'])}")
print()

# Pause to simulate robot movement time
time.sleep(1)

# Save the trajectory
robot.save_trajectory()

# Visualize the exploration
# -------------------------
#
# Let's visualize the robot's trajectory on a map

m = folium.Map(location=[37.7749, -122.4194], zoom_start=16)

# Add the trajectory
coords = [(fix.location.lat().degrees, fix.location.lng().degrees)
for fix in robot.current_trajectory.fixes]
folium.PolyLine(coords, color="blue", weight=2.5, opacity=1).add_to(m)

# Add markers for observations
for fix in robot.current_trajectory.fixes:
lat, lng = fix.location.lat().degrees, fix.location.lng().degrees
if 'scene' in fix.attributes:
popup_text = f"Objects: {', '.join(fix.attributes['scene']['objects'])}"
folium.Marker([lat, lng], popup=popup_text).add_to(m)

# Display the map
m

# Query the spatial memory
# -----------------------
#
# Now let's perform some queries to demonstrate the spatial memory capabilities

print("Querying similar observations...")
similar_obs = robot.query_similar_observations("new_query_image.jpg")
print(f"Found {len(similar_obs)} similar observations")

print("\nQuerying by area...")
area_obs = robot.query_area(37.7750, -122.4180, level=15)
print(f"Found {len(area_obs)} observations in the specified area")

print("\nPerforming hybrid search...")
hybrid_results = robot.hybrid_search("new_query_image.jpg", 37.7750, -122.4180)
print(f"Found {len(hybrid_results)} relevant observations")

# Display some of the hybrid results
for fix_id, similarity in hybrid_results:
print(f"Fix ID: {fix_id}, Similarity: {similarity:.4f}")
```

## Next Steps for Implementation

1. **Refine the DGGS Grid Implementation**:
- Optimize the cell level for indoor robotic exploration
- Add support for 3D representations (elevation/floor levels)

2. **Extend the Semantic Trajectory Model**:
- Add support for semantic segmentation of trajectories
- Implement stop/move classification for more meaningful trajectory division

3. **Improve the Multimodal LLM Integration**:
- Implement actual vision-language model integration (CLIP, GPT-4V, etc.)
- Add fine-tuning capabilities for domain-specific understanding

4. **Enhance the Storage System**:
- Add spatial indexing for more efficient queries
- Implement efficient vector quantization for large-scale deployment

5. **Add Spatial Reasoning Capabilities**:
- Implement qualitative spatial reasoning based on the KG
- Add support for navigation and path planning

6. **Develop Visualization Tools**:
- Create interactive dashboards for exploring the spatial memory
- Build tools for analyzing trajectory patterns

## Benefits of This Approach

1. **Structured Spatial Representation**: The DGGS provides a consistent, hierarchical structure for organizing spatial data at different scales.

2. **Semantic Understanding**: The multimodal LLM integration allows the robot to understand and contextualize its observations.

3. **Efficient Retrieval**: The combination of spatial indexing and vector embeddings enables efficient and flexible querying.

4. **Scalability**: The hierarchical nature of the S2 grid makes it scalable to both small indoor environments and large-scale outdoor settings.

5. **Interoperability**: By using standard ontology patterns like the Semantic Trajectory Pattern, the system can easily integrate with other spatial data systems.

This prototype offers a strong foundation for building an advanced spatial memory system for robots using LLMs and DGGS. The literate programming approach makes it accessible and easy to adapt to specific robotic platforms and use cases.
## 🧑 human _(Feb 27, 2025, 08:12 AM)_
So, I want to construct a knowledge graph, as in rdflib-dggs. The LLM should be able to use the KG as a semantic index along with the vector store.
## 🤖 assistant _(Feb 27, 2025, 08:13 AM)_
# Building a Semantic Knowledge Graph with DGGS and RDFlib

You're right to focus on building a proper RDF-based knowledge graph using DGGS, similar to the approach in rdflib-dggs. This will allow your system to leverage semantic relationships while maintaining the spatial indexing capabilities of DGGS, and enable the LLM to use both the KG and vector store effectively.

Let's develop this approach using RDFlib to construct the knowledge graph:


Now, let's implement the spatial robot that leverages this knowledge graph and integrates with multimodal LLM processing:
## 🧑 human _(Feb 27, 2025, 08:32 AM)_
Ok. This all looks good, but it's a little incomprehensible. I want to build this in the Solveit environment -- a literate, exploratory programming approach similar to colab or jupyter notebooks. The Solveit environment has an additional cell type which is a dialog cell to talk to claude sonnet to assist in building the app in a testable, step-by-step manner.  I think we want to explore using S2, the anthropic API using Claudette, rdflib, the image attribute extraction. Let's come up with a plan for implementing the app within the solveit environment and then using nbdev to generate the python packages and documentation. Pretend you're Jeremy Howard, how would he build this?
## 🤖 assistant _(Feb 27, 2025, 08:34 AM)_
# Building a Spatial Memory System in Solveit: A Jeremy Howard-inspired Approach

You're absolutely right! Let me take a step back and rethink this approach with a more Jeremy Howard-esque philosophy. The code I provided is comprehensive but lacks the exploratory, iterative, and educational aspects that make the fast.ai approach so powerful.

Let's reimagine this project as a series of explorable notebooks in the Solveit environment, using dialog cells to refine our understanding and build incrementally.

## The Fast.ai / Jeremy Howard Approach

When tackling a complex project like this, Jeremy would likely:

1. **Start small and get something working end-to-end first**
2. **Use real examples early** rather than abstract implementations
3. **Build incrementally** with constant testing and visualization
4. **Use existing libraries** wherever possible instead of reinventing
5. **Focus on flexible, modular interfaces** that enable experimentation
6. **Document as you go** via nbdev
7. **Always make the code understandable** with clear explanations of "why"

## Project Plan for Solveit Environment

Let's outline a series of notebooks that would build this system step by step:

### Notebook 1: Exploring S2 Geometry and Spatial Indexing

```python
# [Markdown Cell]
# # Exploring S2 Geometry for Robot Spatial Memory
#
# In this notebook, we'll explore how Google's S2 Geometry library can be used as the
# spatial foundation for our robot's memory system.

# [Code Cell]
!pip install s2sphere folium matplotlib
import s2sphere
import folium
import matplotlib.pyplot as plt
import numpy as np
```

```python
# [Code Cell]
# Let's create a simple example of S2 cells at different levels
def display_s2_cells(lat, lng, levels=[10, 13, 15]):
"""Display S2 cells at different levels containing a point"""
m = folium.Map(location=[lat, lng], zoom_start=15)
folium.Marker([lat, lng], popup="Target Point").add_to(m)

# Add cells at different levels
for level in levels:
cell_id = s2sphere.CellId.from_lat_lng(
s2sphere.LatLng.from_degrees(lat, lng)).parent(level)
cell = s2sphere.Cell(cell_id)

# Get vertices of the cell
vertices = []
for i in range(4):
vertex = cell.get_vertex(i)
ll = s2sphere.LatLng.from_point(vertex)
vertices.append([ll.lat().degrees, ll.lng().degrees])
# Close the polygon
vertices.append(vertices[0])

# Add to map with different colors by level
colors = {10: 'red', 13: 'blue', 15: 'green'}
folium.Polygon(
locations=vertices,
popup=f"S2 Cell Level {level}",
color=colors[level],
fill=True,
fill_opacity=0.2
).add_to(m)

return m

# Try with a sample location (e.g., a university campus)
display_s2_cells(34.0689, -118.4452)  # UCLA campus
```

```
# [Dialog Cell with Claude]
#
# I'm trying to understand how best to use S2 cells for a robot's spatial memory.
# The robot will be indoors mostly - would S2 cells still be appropriate? What level
# should I use for room-scale observations? And what's the difference between
# using S2 (quadtree based) vs H3 (hexagonal) for indoor robotics?
```

### Notebook 2: Implementing the Semantic Trajectory Pattern with RDFlib

```python
# [Markdown Cell]
# # Semantic Trajectory Pattern with RDFlib
#
# In this notebook, we'll implement the Semantic Trajectory ontology pattern using RDFlib.
# This will form the foundation of our robot's spatial memory.

# [Code Cell]
!pip install rdflib s2sphere folium matplotlib
import rdflib
from rdflib import Graph, Namespace, Literal, URIRef, BNode
from rdflib.namespace import RDF, RDFS, XSD
import s2sphere
import uuid
from datetime import datetime
```

```python
# [Code Cell]
# Define our namespaces
ST = Namespace("http://spatialrobot.org/trajectory/")
GEO = Namespace("http://www.opengis.net/ont/geosparql#")
SOSA = Namespace("http://www.w3.org/ns/sosa/")
TIME = Namespace("http://www.w3.org/2006/time#")

# Create a new graph
g = Graph()
g.bind("st", ST)
g.bind("geo", GEO)
g.bind("sosa", SOSA)
g.bind("time", TIME)

# Let's create a simple trajectory
# First, define a few sample fixes (points with timestamps)
fixes = [
(37.7749, -122.4194, "2023-01-01T10:00:00"),  # San Francisco
(37.7746, -122.4172, "2023-01-01T10:05:00"),
(37.7751, -122.4168, "2023-01-01T10:10:00"),
]

# Add each fix to the graph
fix_uris = []
for i, (lat, lng, timestamp) in enumerate(fixes):
fix_id = uuid.uuid4()
fix_uri = URIRef(f"{ST}fix_{fix_id}")
fix_uris.append(fix_uri)

# Create the fix node
g.add((fix_uri, RDF.type, ST.Fix))

# Add location as WKT point
point_wkt = f"POINT({lng} {lat})"
geom_uri = URIRef(f"{ST}geom_{fix_id}")
g.add((geom_uri, RDF.type, GEO.Geometry))
g.add((geom_uri, GEO.asWKT, Literal(point_wkt, datatype=GEO.wktLiteral)))
g.add((fix_uri, GEO.hasGeometry, geom_uri))

# Add timestamp
time_uri = URIRef(f"{ST}time_{fix_id}")
g.add((time_uri, RDF.type, TIME.Instant))
g.add((time_uri, TIME.inXSDDateTime, Literal(timestamp, datatype=XSD.dateTime)))
g.add((fix_uri, ST.atTime, time_uri))

# Add S2 cell for spatial indexing
point = s2sphere.LatLng.from_degrees(lat, lng)
cell = s2sphere.CellId.from_lat_lng(point).parent(13)  # Level 13
g.add((fix_uri, ST.hasS2Cell, Literal(str(cell.id()))))

# Connect the fixes with segments
for i in range(len(fix_uris) - 1):
segment_id = uuid.uuid4()
segment_uri = URIRef(f"{ST}segment_{segment_id}")

# Create the segment
g.add((segment_uri, RDF.type, ST.Segment))
g.add((segment_uri, ST.startsFrom, fix_uris[i]))
g.add((segment_uri, ST.endsAt, fix_uris[i+1]))
g.add((segment_uri, ST.isTraversedBy, URIRef(f"{ST}robot")))

# Create a trajectory that connects all the fixes and segments
traj_id = uuid.uuid4()
traj_uri = URIRef(f"{ST}trajectory_{traj_id}")
g.add((traj_uri, RDF.type, ST.SemanticTrajectory))
g.add((traj_uri, RDFS.label, Literal("Sample Robot Path")))

# Link trajectory to fixes
for fix_uri in fix_uris:
g.add((traj_uri, ST.hasFix, fix_uri))

# Print out our newly created graph in Turtle format
print(g.serialize(format="turtle"))
```

```
# [Dialog Cell with Claude]
#
# I've implemented a basic semantic trajectory pattern, but I'm not sure if I'm
# using the right predicates. For example, should I use ST.hasS2Cell or should I
# create actual S2Cell resources in the graph? Also, how would you recommend
# handling the parent-child relationships between S2 cells of different levels?
```

### Notebook 3: Integrating Multimodal LLMs for Scene Understanding

```python
# [Markdown Cell]
# # Multimodal Scene Understanding with Claude
#
# In this notebook, we'll explore how to use Claude's vision capabilities to
# understand scenes and extract semantic information that can enrich our spatial memory.

# [Code Cell]
!pip install anthropic pillow matplotlib
import anthropic
import os
import json
from PIL import Image
import matplotlib.pyplot as plt
import base64
from io import BytesIO
```

```python
# [Code Cell]
# Initialize the Claude client
client = anthropic.Anthropic(
api_key=os.environ.get("ANTHROPIC_API_KEY")
)

# Helper function to encode image for Claude API
def encode_image(image_path):
with open(image_path, "rb") as image_file:
return base64.b64encode(image_file.read()).decode('utf-8')

# Function to extract scene information using Claude
def analyze_scene(image_path):
"""Use Claude to analyze a scene and extract structured information"""

# Prepare the image
base64_image = encode_image(image_path)

# Define the prompt
prompt = """
Analyze this image carefully. I want a structured description with:
1. A general description of the scene
2. A list of objects visible in the scene
3. Spatial relationships between objects (e.g., "chair in front of desk")
4. The type of environment (indoor/outdoor)

Format your response as a valid JSON object with keys: "description", "objects",
"spatial_relations" (as an array of {"subject", "relation", "object"}), and "environment".
"""

# Send the request to Claude
message = client.messages.create(
model="claude-3-opus-20240229",
max_tokens=1000,
messages=[
{
"role": "user",
"content": [
{
"type": "text",
"text": prompt
},
{
"type": "image",
"source": {
"type": "base64",
"media_type": "image/jpeg",
"data": base64_image
}
}
]
}
],
temperature=0
)

# Parse the JSON response
try:
# Extract the JSON part from Claude's response
response_text = message.content[0].text
# Find the JSON block
json_start = response_text.find('{')
json_end = response_text.rfind('}') + 1
if json_start >= 0 and json_end > json_start:
json_str = response_text[json_start:json_end]
scene_data = json.loads(json_str)
return scene_data
else:
# If no JSON block found, try to parse the whole response
return json.loads(response_text)
except json.JSONDecodeError:
# If JSON parsing fails, return the raw text
return {"raw_response": message.content[0].text}

# Test with a sample image
test_image_path = "path/to/your/test_image.jpg"
if os.path.exists(test_image_path):
scene_info = analyze_scene(test_image_path)
print(json.dumps(scene_info, indent=2))
else:
print(f"Test image not found at {test_image_path}")
```

```
# [Dialog Cell with Claude]
#
# I've set up the Claude vision integration, but I'm wondering what would be the best
# way to structure the scene information for our spatial memory. Should we extract
# individual objects with their positions? Should we create a "scene graph" in our
# RDF knowledge base? What structure would be most useful for a robot trying to
# remember and reason about places it has visited?
```

### Notebook 4: Vector Database Integration with SQLite-vec

```python
# [Markdown Cell]
# # Vector Database Integration with SQLite-vec
#
# In this notebook, we'll explore how to store and retrieve vector embeddings
# for our spatial observations using SQLite-vec.

# [Code Cell]
!pip install sqlite-vec sentence-transformers numpy matplotlib
import sqlite_vec
import numpy as np
from sentence_transformers import SentenceTransformer
import matplotlib.pyplot as plt
from sklearn.decomposition import PCA
import json
```

```python
# [Code Cell]
# Initialize the database
db = sqlite_vec.connect("robot_memory.db")

# Create a table for our embeddings
db.execute("""
CREATE TABLE IF NOT EXISTS scene_embeddings (
id TEXT PRIMARY KEY,
location TEXT,  -- JSON string with lat/lng
s2cell TEXT,    -- S2 cell ID for spatial indexing
timestamp TEXT,
embedding BLOB, -- Binary embedding vector
metadata TEXT   -- Additional JSON metadata
)
""")

# Create a vector index
db.enable_vectors("scene_embeddings", "embedding", dimension=512)

# Initialize the embedding model
model = SentenceTransformer('clip-ViT-B-32')

# Function to add a scene embedding
def add_scene_embedding(
scene_id, description, location, s2cell, timestamp, metadata=None
):
"""Add a scene embedding to the database"""
# Generate embedding from scene description
embedding = model.encode(description)

# Store in database
db.execute(
"INSERT INTO scene_embeddings VALUES (?, ?, ?, ?, ?, ?)",
(
scene_id,
json.dumps(location),
s2cell,
timestamp,
embedding.tobytes(),
json.dumps(metadata or {})
)
)

return scene_id

# Function to query similar scenes
def find_similar_scenes(description, top_k=5):
"""Find scenes similar to the given description"""
# Generate query embedding
query_embedding = model.encode(description)

# Query the database
results = db.execute(
"SELECT id, location, metadata, vector_cosine_distance(embedding, ?) AS distance "
"FROM scene_embeddings "
"ORDER BY distance ASC "
"LIMIT ?",
(query_embedding.tobytes(), top_k)
).fetchall()

return [
{
"id": row[0],
"location": json.loads(row[1]),
"metadata": json.loads(row[3]),
"similarity": 1 - row[4]  # Convert distance to similarity
}
for row in results
]

# Test with some sample data
sample_scenes = [
{
"id": "scene_001",
"description": "A kitchen with a large refrigerator, stove, and wooden cabinets",
"location": {"lat": 37.7749, "lng": -122.4194},
"s2cell": "1234567890",
"timestamp": "2023-01-01T10:00:00",
"metadata": {"room_type": "kitchen", "objects": ["refrigerator", "stove", "cabinets"]}
},
{
"id": "scene_002",
"description": "A living room with a sofa, coffee table, and TV mounted on the wall",
"location": {"lat": 37.7749, "lng": -122.4195},
"s2cell": "1234567891",
"timestamp": "2023-01-01T10:05:00",
"metadata": {"room_type": "living room", "objects": ["sofa", "coffee table", "TV"]}
},
{
"id": "scene_003",
"description": "A bathroom with a shower, toilet, and sink with a mirror above it",
"location": {"lat": 37.7750, "lng": -122.4196},
"s2cell": "1234567892",
"timestamp": "2023-01-01T10:10:00",
"metadata": {"room_type": "bathroom", "objects": ["shower", "toilet", "sink", "mirror"]}
}
]

# Add sample scenes to the database
for scene in sample_scenes:
add_scene_embedding(
scene["id"],
scene["description"],
scene["location"],
scene["s2cell"],
scene["timestamp"],
scene["metadata"]
)

# Test similarity search
test_query = "A kitchen with a fridge and oven"
similar_scenes = find_similar_scenes(test_query)
print("Query:", test_query)
print("Similar scenes:")
for scene in similar_scenes:
print(f"ID: {scene['id']}, Similarity: {scene['similarity']:.4f}")
print(f"Metadata: {scene['metadata']}")
print()
```

```
# [Dialog Cell with Claude]
#
# I'm trying to understand the best way to link the vector database with the RDF
# knowledge graph. Should I store URIs in the vector database that reference nodes
# in the RDF graph? Or should I store the vector embeddings directly in the RDF graph
# somehow? What's the most efficient approach that still maintains the connection
# between semantic structure and vector similarity?
```

### Notebook 5: Building the Integrated System

```python
# [Markdown Cell]
# # Integrated Spatial Memory System
#
# In this notebook, we'll bring together all the components we've explored:
# - S2 Geometry for spatial indexing
# - Semantic Trajectory Pattern with RDFlib
# - Multimodal Scene Understanding with Claude
# - Vector Database Integration
#
# We'll build a complete, integrated system that a robot could use to remember
# and reason about places it has visited.

# [Code Cell]
!pip install rdflib s2sphere anthropic sqlite-vec sentence-transformers folium matplotlib
import rdflib
from rdflib import Graph, Namespace, Literal, URIRef, BNode
from rdflib.namespace import RDF, RDFS, XSD
import s2sphere
import anthropic
import sqlite_vec
import numpy as np
from sentence_transformers import SentenceTransformer
import folium
import matplotlib.pyplot as plt
import json
import uuid
import os
from datetime import datetime
from PIL import Image
import base64
from io import BytesIO

```python
# [Code Cell]
# Create our SpatialMemory class that integrates all components
class SpatialMemory:
def __init__(self, memory_path="robot_memory"):
"""Initialize the spatial memory system"""
os.makedirs(memory_path, exist_ok=True)
self.memory_path = memory_path

# Initialize RDF graph
self.graph = Graph()
self._setup_graph()

# Initialize vector database
self.vector_db = sqlite_vec.connect(f"{memory_path}/vectors.db")
self._setup_vector_db()

# Initialize embedding model
self.model = SentenceTransformer('clip-ViT-B-32')

# Initialize Claude client if API key is available
api_key = os.environ.get("ANTHROPIC_API_KEY")
self.claude = anthropic.Anthropic(api_key=api_key) if api_key else None

def _setup_graph(self):
"""Set up the RDF graph with namespaces and base ontology"""
# Define namespaces
self.ST = Namespace("http://spatialrobot.org/trajectory/")
self.GEO = Namespace("http://www.opengis.net/ont/geosparql#")
self.SOSA = Namespace("http://www.w3.org/ns/sosa/")
self.TIME = Namespace("http://www.w3.org/2006/time#")
self.ROB = Namespace("http://spatialrobot.org/robot/")
self.VEC = Namespace("http://spatialrobot.org/vector/")

# Bind namespaces
self.graph.bind("st", self.ST)
self.graph.bind("geo", self.GEO)
self.graph.bind("sosa", self.SOSA)
self.graph.bind("time", self.TIME)
self.graph.bind("rob", self.ROB)
self.graph.bind("vec", self.VEC)

# Create basic ontology classes and properties
# (This would be more extensive in a real implementation)
self.graph.add((self.ST.S2Cell, RDF.type, RDFS.Class))
self.graph.add((self.ST.Fix, RDF.type, RDFS.Class))
self.graph.add((self.ST.Segment, RDF.type, RDFS.Class))
self.graph.add((self.ST.SemanticTrajectory, RDF.type, RDFS.Class))
self.graph.add((self.ST.Observation, RDF.type, RDFS.Class))

# Load existing graph if available
graph_path = f"{self.memory_path}/knowledge_graph.ttl"
if os.path.exists(graph_path):
self.graph.parse(graph_path, format="turtle")
print(f"Loaded existing knowledge graph from {graph_path}")

def _setup_vector_db(self):
"""Set up the vector database for embeddings"""
# Create table for observations
self.vector_db.execute("""
CREATE TABLE IF NOT EXISTS observations (
uri TEXT PRIMARY KEY,
s2cell TEXT,
timestamp TEXT,
embedding BLOB,
metadata TEXT
)
""")

# Create vector index
dimension = 512  # CLIP's default dimension
self.vector_db.enable_vectors("observations", "embedding", dimension=dimension)

# S2 Cell management functions
def add_s2cell(self, cell_id, level=None):
"""Add an S2 cell to the graph if it doesn't exist yet"""
if isinstance(cell_id, str):
cell_id = int(cell_id)

cell = s2sphere.CellId(cell_id)
if level is None:
level = cell.level()

cell_uri = URIRef(f"{self.ST}cell_{cell_id}")

# Check if the cell already exists
if (cell_uri, RDF.type, self.ST.S2Cell) in self.graph:
return cell_uri

# Add cell to graph
self.graph.add((cell_uri, RDF.type, self.ST.S2Cell))
self.graph.add((cell_uri, self.ST.hasID, Literal(str(cell_id))))
self.graph.add((cell_uri, self.ST.hasLevel, Literal(level, datatype=XSD.integer)))

# Add WKT geometry
cell_obj = s2sphere.Cell(cell)
vertices = []
for i in range(4):
vertex = cell_obj.get_vertex(i)
ll = s2sphere.LatLng.from_point(vertex)
vertices.append(f"{ll.lng().degrees} {ll.lat().degrees}")

# Close the polygon
vertices.append(vertices[0])
wkt = f"POLYGON(({', '.join(vertices)}))"

geom_uri = URIRef(f"{self.ST}geom_{cell_id}")
self.graph.add((geom_uri, RDF.type, self.GEO.Geometry))
self.graph.add((geom_uri, self.GEO.asWKT, Literal(wkt, datatype=self.GEO.wktLiteral)))
self.graph.add((cell_uri, self.GEO.hasGeometry, geom_uri))

# Add parent-child relationships
if level > 0:
parent_cell = cell.parent(level - 1)
parent_uri = self.add_s2cell(parent_cell.id(), level - 1)

self.graph.add((parent_uri, self.ST.contains, cell_uri))
self.graph.add((cell_uri, self.ST.within, parent_uri))

return cell_uri

# Functions for adding trajectories, fixes, and observations

def add_fix(self, lat, lng, timestamp, attributes=None):
"""Add a fix to the graph"""
fix_id = str(uuid.uuid4())
fix_uri = URIRef(f"{self.ST}fix_{fix_id}")

# Create the fix node
self.graph.add((fix_uri, RDF.type, self.ST.Fix))

# Add location as WKT point
point_wkt = f"POINT({lng} {lat})"
geom_uri = URIRef(f"{self.ST}geom_{fix_id}")
self.graph.add((geom_uri, RDF.type, self.GEO.Geometry))
self.graph.add((geom_uri, self.GEO.asWKT, Literal(point_wkt, datatype=self.GEO.wktLiteral)))
self.graph.add((fix_uri, self.GEO.hasGeometry, geom_uri))

# Add timestamp
time_uri = URIRef(f"{self.ST}time_{fix_id}")
self.graph.add((time_uri, RDF.type, self.TIME.Instant))
if isinstance(timestamp, str):
self.graph.add((time_uri, self.TIME.inXSDDateTime, Literal(timestamp, datatype=XSD.dateTime)))
else:
# Assume datetime object
self.graph.add((time_uri, self.TIME.inXSDDateTime, Literal(timestamp.isoformat(), datatype=XSD.dateTime)))
self.graph.add((fix_uri, self.ST.atTime, time_uri))

# Add S2 cell for spatial indexing (level 13 by default)
point = s2sphere.LatLng.from_degrees(lat, lng)
cell = s2sphere.CellId.from_lat_lng(point).parent(13)
cell_uri = self.add_s2cell(cell.id())
self.graph.add((fix_uri, self.ST.hasLocation, cell_uri))

# Add attributes if provided
if attributes:
for key, value in attributes.items():
attr_uri = URIRef(f"{self.ST}attr_{fix_id}_{key}")
self.graph.add((attr_uri, RDF.type, self.ST.Attribute))
self.graph.add((attr_uri, RDFS.label, Literal(key)))

if isinstance(value, dict):
self.graph.add((attr_uri, RDFS.value, Literal(json.dumps(value))))
else:
self.graph.add((attr_uri, RDFS.value, Literal(value)))

self.graph.add((fix_uri, self.ST.hasAttribute, attr_uri))

return fix_uri

def add_observation(self, fix_uri, image_path=None, scene_data=None):
"""Add an observation to a fix, with visual embedding and scene data"""
obs_id = str(uuid.uuid4())
obs_uri = URIRef(f"{self.ST}obs_{obs_id}")

# Create observation node
self.graph.add((obs_uri, RDF.type, self.ST.Observation))
self.graph.add((obs_uri, self.SOSA.madeBySensor, self.ROB.camera))
self.graph.add((fix_uri, self.ST.hasObservation, obs_uri))

# If we have an image, process it
if image_path and os.path.exists(image_path):
# Get timestamp from fix
timestamp = None
for time_uri in self.graph.objects(fix_uri, self.ST.atTime):
for time_value in self.graph.objects(time_uri, self.TIME.inXSDDateTime):
timestamp = str(time_value)

# Get S2 cell from fix
s2cell = None
for cell_uri in self.graph.objects(fix_uri, self.ST.hasLocation):
for cell_id in self.graph.objects(cell_uri, self.ST.hasID):
s2cell = str(cell_id)

# Process image with Claude if available and no scene data provided
if self.claude and not scene_data:
scene_data = self._analyze_scene_with_claude(image_path)

# Generate embedding
embedding = self._generate_embedding(image_path, scene_data)

# Store embedding in vector database
self.vector_db.execute(
"INSERT OR REPLACE INTO observations VALUES (?, ?, ?, ?, ?)",
(
str(obs_uri),
s2cell,
timestamp,
embedding.tobytes(),
json.dumps(scene_data) if scene_data else None
)
)

# Add reference to embedding in RDF graph
self.graph.add((obs_uri, self.VEC.hasEmbedding, Literal(True)))

# If we have scene data, add it to the graph
if scene_data:
# Add scene description
self.graph.add((obs_uri, RDFS.comment, Literal(scene_data.get("description", ""))))

# Add detected objects
for obj in scene_data.get("objects", []):
obj_uri = URIRef(f"{self.ST}object_{obs_id}_{obj.replace(' ', '_')}")
self.graph.add((obj_uri, RDF.type, self.ST.Object))
self.graph.add((obj_uri, RDFS.label, Literal(obj)))
self.graph.add((obs_uri, self.ST.hasDetectedObject, obj_uri))

# Add spatial relationships
for rel in scene_data.get("spatial_relations", []):
if "subject" in rel and "relation" in rel and "object" in rel:
rel_uri = URIRef(f"{self.ST}rel_{obs_id}_{len(self.graph)}")
self.graph.add((rel_uri, RDF.type, self.ST.SpatialRelation))
self.graph.add((rel_uri, self.ST.hasSubject, Literal(rel["subject"])))
self.graph.add((rel_uri, self.ST.hasRelation, Literal(rel["relation"])))
self.graph.add((rel_uri, self.ST.hasObject, Literal(rel["object"])))
self.graph.add((obs_uri, self.ST.hasSpatialRelation, rel_uri))

return obs_uri

def _analyze_scene_with_claude(self, image_path):
"""Analyze a scene with Claude Vision"""
if not self.claude:
print("Claude API not available - skipping scene analysis")
return None

# Encode image
with open(image_path, "rb") as f:
base64_image = base64.b64encode(f.read()).decode("utf-8")

# Define prompt
prompt = """
Analyze this image carefully. I want a structured description with:
1. A general description of the scene
2. A list of objects visible in the scene
3. Spatial relationships between objects (e.g., "chair in front of
## 🧑 human _(Feb 27, 2025, 08:35 AM)_
Continue
## 🤖 assistant _(Feb 27, 2025, 08:36 AM)_
Let me continue outlining the Solveit notebook structure for our Spatial Memory system, following Jeremy Howard's approach of step-by-step, exploratory programming:

### Notebook 5: Building the Integrated System (continued)

```python
# [Code Cell - continued]
prompt = """
Analyze this image carefully. I want a structured description with:
1. A general description of the scene
2. A list of objects visible in the scene
3. Spatial relationships between objects (e.g., "chair in front of desk")
4. The type of environment (indoor/outdoor)

Format your response as a valid JSON object with keys: "description", "objects",
"spatial_relations" (as an array of {"subject", "relation", "object"}), and "environment".
"""

# Call Claude API
try:
message = self.claude.messages.create(
model="claude-3-opus-20240229",
max_tokens=1000,
messages=[
{
"role": "user",
"content": [
{
"type": "text",
"text": prompt
},
{
"type": "image",
"source": {
"type": "base64",
"media_type": "image/jpeg",
"data": base64_image
}
}
]
}
],
temperature=0
)

# Parse the JSON response
response_text = message.content[0].text
json_start = response_text.find('{')
json_end = response_text.rfind('}') + 1
if json_start >= 0 and json_end > json_start:
json_str = response_text[json_start:json_end]
return json.loads(json_str)
return None
except Exception as e:
print(f"Error analyzing scene with Claude: {e}")
return None

def _generate_embedding(self, image_path, scene_data=None):
"""Generate an embedding for an image and/or scene data"""
# If we have scene data, use description for text embedding
if scene_data and "description" in scene_data:
return self.model.encode(scene_data["description"])

# Otherwise, encode the image directly
try:
image = Image.open(image_path)
# In a real implementation, we'd use the image_model
# For now, return a deterministic embedding based on image path
embedding_dim = 512
seed = hash(str(image_path)) % 10000
np.random.seed(seed)
embedding = np.random.randn(embedding_dim)
# Normalize
embedding = embedding / np.linalg.norm(embedding)
return embedding
except Exception as e:
print(f"Error generating embedding: {e}")
return np.zeros(512)

# Query functions

def query_by_location(self, lat, lng, radius_meters=100, level=13):
"""Query for observations near a specific location"""
# Get the S2 cell for this location
point = s2sphere.LatLng.from_degrees(lat, lng)
cell_id = s2sphere.CellId.from_lat_lng(point).parent(level)

# Get a covering of cells within the radius
region = s2sphere.Cap.from_axis_height(
s2sphere.LatLng.from_degrees(lat, lng).to_point(),
1 - math.cos(radius_meters / 6371000.0)  # Earth radius in meters
)
coverer = s2sphere.RegionCoverer()
coverer.min_level = level
coverer.max_level = level
covering = coverer.get_covering(region)

# Query for observations in these cells
results = []
for cell in covering:
# Query the RDF graph
cell_uri = URIRef(f"{self.ST}cell_{cell.id()}")

# Find fixes at this location
for fix_uri in self.graph.subjects(self.ST.hasLocation, cell_uri):
# Find observations for these fixes
for obs_uri in self.graph.objects(fix_uri, self.ST.hasObservation):
# Get observation details
observation = {
"uri": str(obs_uri),
"location": {"lat": lat, "lng": lng},
"cell_id": str(cell.id()),
"objects": [],
"description": ""
}

# Get timestamp
for time_uri in self.graph.objects(fix_uri, self.ST.atTime):
for time_value in self.graph.objects(time_uri, self.TIME.inXSDDateTime):
observation["timestamp"] = str(time_value)

# Get description
for description in self.graph.objects(obs_uri, RDFS.comment):
observation["description"] = str(description)

# Get detected objects
for obj_uri in self.graph.objects(obs_uri, self.ST.hasDetectedObject):
for label in self.graph.objects(obj_uri, RDFS.label):
observation["objects"].append(str(label))

results.append(observation)

return results

def find_similar_observations(self, query_embedding, top_k=5):
"""Find observations with similar embeddings"""
# Query the vector database
if isinstance(query_embedding, np.ndarray):
query_embedding = query_embedding.tobytes()

results = self.vector_db.execute("""
SELECT uri, metadata, vector_cosine_distance(embedding, ?) AS distance
FROM observations
ORDER BY distance ASC
LIMIT ?
""", (query_embedding, top_k)).fetchall()

# Format results
observations = []
for uri, metadata_json, distance in results:
metadata = json.loads(metadata_json) if metadata_json else {}

# Get additional information from RDF graph
obs_uri = URIRef(uri)
observation = {
"uri": uri,
"similarity": 1 - distance,  # Convert distance to similarity
"description": metadata.get("description", ""),
"objects": metadata.get("objects", []),
"spatial_relations": metadata.get("spatial_relations", [])
}

# Find the associated fix
for fix_uri in self.graph.subjects(self.ST.hasObservation, obs_uri):
# Get location from fix
for cell_uri in self.graph.objects(fix_uri, self.ST.hasLocation):
for geom_uri in self.graph.objects(cell_uri, self.GEO.hasGeometry):
for wkt in self.graph.objects(geom_uri, self.GEO.asWKT):
observation["location"] = str(wkt)

# Get timestamp from fix
for time_uri in self.graph.objects(fix_uri, self.ST.atTime):
for time_value in self.graph.objects(time_uri, self.TIME.inXSDDateTime):
observation["timestamp"] = str(time_value)

observations.append(observation)

return observations

def hybrid_search(self, image_path, lat, lng, radius_meters=100, level=13, top_k=5):
"""Perform a hybrid search using both location and visual similarity"""
# Generate embedding for the query image
query_embedding = self._generate_embedding(image_path)

# Get the S2 cell for this location
point = s2sphere.LatLng.from_degrees(lat, lng)
cell_id = s2sphere.CellId.from_lat_lng(point).parent(level)

# Query the vector database with spatial constraint
results = self.vector_db.execute("""
SELECT uri, metadata, vector_cosine_distance(embedding, ?) AS distance
FROM observations
WHERE s2cell IN (
SELECT s2cell FROM observations
WHERE ST_Distance(
ST_Point(?, ?),
ST_Point(json_extract(metadata, '$.location.lng'), json_extract(metadata, '$.location.lat'))
) <= ?
)
ORDER BY distance ASC
LIMIT ?
""", (query_embedding.tobytes(), lng, lat, radius_meters/111000, top_k)).fetchall()  # rough conversion to degrees

# Format results
observations = []
for uri, metadata_json, distance in results:
metadata = json.loads(metadata_json) if metadata_json else {}

observation = {
"uri": uri,
"similarity": 1 - distance,
"description": metadata.get("description", ""),
"objects": metadata.get("objects", []),
"location": metadata.get("location", {})
}

observations.append(observation)

return observations

def save(self):
"""Save the spatial memory to disk"""
# Save RDF graph
graph_path = f"{self.memory_path}/knowledge_graph.ttl"
self.graph.serialize(destination=graph_path, format="turtle")

# Vector database is automatically saved (SQLite)
print(f"Saved spatial memory to {self.memory_path}")

def load(self):
"""Load the spatial memory from disk"""
graph_path = f"{self.memory_path}/knowledge_graph.ttl"
if os.path.exists(graph_path):
self.graph.parse(graph_path, format="turtle")
print(f"Loaded knowledge graph from {graph_path}")
else:
print(f"No knowledge graph found at {graph_path}")

# Let's create a simple robot class that uses this spatial memory
class SpatialRobot:
def __init__(self, memory_path="robot_memory", api_key=None):
"""Initialize the robot with spatial memory capabilities"""
# Set up spatial memory
self.memory = SpatialMemory(memory_path)

# Robot state
self.position = (0, 0)  # (lat, lng)
self.trajectory = []
self.current_fixes = []

def start_exploration(self, name=None):
"""Start a new exploration trajectory"""
if name is None:
name = f"exploration_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

self.trajectory_name = name
self.current_fixes = []

print(f"Started new exploration: {name}")
return name

def move_to(self, lat, lng):
"""Move to a new position and record a fix"""
self.position = (lat, lng)

# Create a fix in the spatial memory
fix_uri = self.memory.add_fix(lat, lng, datetime.now())
self.current_fixes.append(fix_uri)

print(f"Moved to position: {lat}, {lng}")
return fix_uri

def observe(self, image_path):
"""Make an observation at the current position"""
if not self.current_fixes:
print("No current position. Move the robot first.")
return None

# Add observation to the most recent fix
current_fix_uri = self.current_fixes[-1]
obs_uri = self.memory.add_observation(current_fix_uri, image_path)

print(f"Recorded observation at current position")
return obs_uri

def end_exploration(self):
"""End the current exploration and save to memory"""
print(f"Ended exploration with {len(self.current_fixes)} fixes")
self.memory.save()

# Reset state
self.current_fixes = []

def remember_similar(self, image_path, top_k=3):
"""Remember observations similar to the given image"""
# Generate query embedding
query_embedding = self.memory._generate_embedding(image_path)

# Find similar observations
similar_obs = self.memory.find_similar_observations(query_embedding, top_k)

return similar_obs

def remember_nearby(self, radius_meters=100):
"""Remember observations near the current position"""
lat, lng = self.position
nearby_obs = self.memory.query_by_location(lat, lng, radius_meters)

return nearby_obs

def visualize_memory(self):
"""Visualize the robot's spatial memory on a map"""
# If no position is set, use a default
lat, lng = self.position if self.position != (0, 0) else (37.7749, -122.4194)

# Create a map centered at current position
m = folium.Map(location=[lat, lng], zoom_start=15)

# Add all fixes to the map
query = """
SELECT ?fix ?lat ?lng ?time
WHERE {
?fix a <http://spatialrobot.org/trajectory/Fix> .
?fix <http://www.opengis.net/ont/geosparql#hasGeometry> ?geom .
?geom <http://www.opengis.net/ont/geosparql#asWKT> ?wkt .
?fix <http://spatialrobot.org/trajectory/atTime> ?timeNode .
?timeNode <http://www.w3.org/2006/time#inXSDDateTime> ?time .
BIND(geof:getX(?wkt) AS ?lng)
BIND(geof:getY(?wkt) AS ?lat)
}
"""

# In a real implementation, we'd use the query above
# For our simple demo, we'll just iterate through the graph
fixes = []
for fix_uri in self.memory.graph.subjects(RDF.type, self.memory.ST.Fix):
# Get geometry
for geom_uri in self.memory.graph.objects(fix_uri, self.memory.GEO.hasGeometry):
for wkt in self.memory.graph.objects(geom_uri, self.memory.GEO.asWKT):
# Parse WKT to get lat/lng
wkt_str = str(wkt)
if wkt_str.startswith("POINT"):
# Extract coordinates from POINT(lng lat)
coords = wkt_str.strip("POINT()").split()
lng, lat = float(coords[0]), float(coords[1])

# Get timestamp
timestamp = None
for time_uri in self.memory.graph.objects(fix_uri, self.memory.ST.atTime):
for time_val in self.memory.graph.objects(time_uri, self.memory.TIME.inXSDDateTime):
timestamp = str(time_val)

fixes.append({
"uri": str(fix_uri),
"lat": lat,
"lng": lng,
"timestamp": timestamp
})

# Add markers for fixes
for fix in fixes:
popup_text = f"Fix: {fix['uri'].split('/')[-1]}<br>Time: {fix['timestamp']}"
folium.Marker(
location=[fix["lat"], fix["lng"]],
popup=popup_text,
icon=folium.Icon(color="blue")
).add_to(m)

# Add current position
folium.Marker(
location=[lat, lng],
popup="Current Position",
icon=folium.Icon(color="red")
).add_to(m)

return m
```

```
# [Dialog Cell with Claude]
#
# I've built an integrated system that combines S2 cells, semantic trajectories,
# multimodal vision analysis, and vector embeddings. A few questions:
#
# 1. Would you recommend materializing the topological relationships between S2 cells
#    (contains, within, touches) in the RDF graph, or computing them on-the-fly?
#
# 2. For a robot exploring indoor environments, what's the best approach to handle
#    coordinate systems? Geographic coordinates aren't precise enough for room-scale
#    navigation. Should I convert local coordinates to geographic, or is there a better way?
#
# 3. How would you adapt this system if we wanted to implement SLAM (Simultaneous
#    Localization and Mapping) capabilities?
```

### Notebook 6: Testing the System with a Simulated Robot Exploration

```python
# [Markdown Cell]
# # Testing the Spatial Memory System
#
# In this notebook, we'll test our spatial memory system with a simulated robot
# exploration scenario. We'll use some sample images and locations to create a
# realistic test.

# [Code Cell]
# Test our spatial memory system with a simulated exploration
!mkdir -p test_images robot_memory

# Download some test images (or use your own)
!wget -P test_images https://example.com/room1.jpg
!wget -P test_images https://example.com/room2.jpg
!wget -P test_images https://example.com/room3.jpg

# Initialize our robot
robot = SpatialRobot(memory_path="robot_memory")

# Start exploration
robot.start_exploration("test_exploration")

# Define a simulated path through some rooms
waypoints = [
(37.7749, -122.4194, "test_images/room1.jpg", "Living room"),
(37.7750, -122.4195, "test_images/room2.jpg", "Kitchen"),
(37.7751, -122.4196, "test_images/room3.jpg", "Bedroom")
]

# Move the robot along the path and make observations
for lat, lng, image_path, room_type in waypoints:
# Move to location
robot.move_to(lat, lng)

# Make observation
if os.path.exists(image_path):
robot.observe(image_path)
print(f"Observed {room_type} at ({lat}, {lng})")
else:
print(f"Image not found: {image_path}")

# Pause to simulate movement
time.sleep(1)

# End exploration
robot.end_exploration()

# Visualize the robot's memory
memory_map = robot.visualize_memory()
memory_map
```

```python
# [Code Cell]
# Test memory retrieval
# Let's retrieve observations near the first waypoint
nearby = robot.remember_nearby(radius_meters=50)
print(f"Found {len(nearby)} observations nearby:")
for obs in nearby:
print(f"- {obs.get('description', 'No description')} at {obs.get('timestamp', 'unknown time')}")

# Test visual similarity
if os.path.exists("test_images/room1.jpg"):
similar = robot.remember_similar("test_images/room1.jpg", top_k=2)
print(f"\nFound {len(similar)} visually similar observations:")
for obs in similar:
print(f"- {obs.get('description', 'No description')} (similarity: {obs.get('similarity', 0):.2f})")
```

```
# [Dialog Cell with Claude]
#
# I've tested our system with a simulated exploration, but I'm not sure if this is the
# best way to evaluate it. What metrics would you use to evaluate the system's
# effectiveness? And how would you design more comprehensive tests to make sure all
# components are working well together?
```

## Packaging with nbdev

After developing and testing our system in Solveit notebooks, we can use nbdev to turn these notebooks into a proper Python package with documentation. Here's how we'd set up the nbdev configuration:

### Create a settings.ini file:

```ini
[DEFAULT]
# Library name
lib_name = spatial_memory
# Minimum Python version
min_python = 3.8
# Set to True if you want to create a more fancy sidebar.json than the default
custom_sidebar = False
# Change to True if you want to use MathJax for LaTeX support
use_math = True
# Change to True if you want to document only the exported modules (False will document all modules)
doc_only_exported = True
# Change to True if you want to use the development version of nbdev
dev_requirements = True
# List of python packages required for development
dev_requirements = nbdev jupyter pandas matplotlib folium anthropic s2sphere rdflib sqlite-vec sentence-transformers
# The version of your package
version = 0.0.1
# GitHub username and repo name
user = yourusername
repo = spatial_memory
# A one-line description of your library
description = A spatial memory system for robots using S2, LLMs, and knowledge graphs
# Keywords for your library
keywords = robotics spatial-memory s2-geometry knowledge-graph vector-database
# The license of your package
license = apache2
# Where your documentation will be hosted
doc_host = https://yourusername.github.io
# The branch to be used for doc deployment
doc_branch = main
# Where will your package be hosted
pip_requirements = s2sphere rdflib anthropic sqlite-vec sentence-transformers folium
```

### Creating the nbdev Infrastructure

We would run:

```bash
nbdev_new
```

This creates the basic infrastructure including:

- `nbs/` directory for notebooks
- `spatial_memory/` directory for the generated Python modules
- GitHub Actions workflows for testing and deployment
- Documentation templates

We'd then move our Solveit notebooks into the `nbs/` directory and add appropriate nbdev directives:

```python
# | export
# This directive tells nbdev to export this cell to the Python module
```

And for functions or classes we want to export:

```python
# | export
class SpatialMemory:
# ...
```

For documentation, we'd add:

```python
# | hide
# This cell won't appear in the documentation
```

### Converting Notebooks to Python Modules

Once our notebooks are ready with the appropriate directives, we'd run:

```bash
nbdev_export
```

This would extract all the code cells marked with `# | export` into proper Python modules in the `spatial_memory/` directory.

### Building Documentation

To generate documentation from our notebooks:

```bash
nbdev_docs
```

This would create documentation using Quarto, which can be hosted on GitHub Pages.

## Final Project Structure

Our final project structure would look like:

```
spatial_memory/
├── .github/           # GitHub workflows
├── nbs/               # Notebooks
│   ├── 00_core.ipynb  # Core functionality
│   ├── 01_s2.ipynb    # S2 geometry integration
│   ├── 02_kg.ipynb    # Knowledge graph functionality
│   ├── 03_vision.ipynb # Vision integration
│   ├── 04_vectors.ipynb # Vector database integration
│   ├── 05_robot.ipynb  # Robot interface
│   ├── index.ipynb    # Documentation index
├── spatial_memory/    # Generated Python modules
│   ├── __init__.py
│   ├── core.py
│   ├── s2.py
│   ├── kg.py
│   ├── vision.py
│   ├── vectors.py
│   ├── robot.py
├── docs/              # Generated documentation
├── README.md
├── LICENSE
├── settings.ini
└── setup.py
```

## Why This Approach Works

Following Jeremy Howard's approach to building this complex system has several advantages:

1. **Incremental development**: We start with the simplest version that works and gradually build complexity.

2. **Exploratory programming**: Using Solveit dialog cells allows us to refine our understanding while coding.

3. **Testing as we go**: Each notebook can be executed independently to verify functionality.

4. **Real examples early**: We work with concrete examples from the beginning, making it easier to understand.

5. **Educational**: The notebooks serve both as code and as learning materials.

6. **Reproducible**: Anyone can follow our process step by step.

7. **Properly packaged**: Using nbdev ensures our exploratory work becomes a robust, documented package.

By structuring our work this way, we create not just a spatial memory system, but a complete, well-documented, and readable solution that others can learn from and extend.
## 🧑 human _(Feb 27, 2025, 08:39 AM)_
Ok. I can add a context markdown file for the LLM within the Solveit notebook. The context helps give background factual knowledge and and steer the LLM to the task. Can you generate the context for me?
## 🤖 assistant _(Feb 27, 2025, 08:39 AM)_
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

### 5. Vector Database Integration
We're using SQLite-vec to store vector embeddings that represent:
- Visual scene embeddings
- Semantic embeddings of descriptions
- These enable similarity-based retrieval of past observations

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
- Anthropic API: For accessing Claude's vision capabilities
- SQLite-vec: Vector extension for SQLite
- Sentence-Transformers: For generating embeddings
- Folium: For visualization of spatial data
- nbdev: For turning notebooks into Python packages with documentation

This notebook is part of an incremental exploration to build the system step by step, with each stage working toward a complete integration of all components.
## 🧑 human _(Feb 27, 2025, 08:59 AM)_
Ok. I think I want to use polars and pandas dataframes instead of SQLite-vec to store the embeddings. KGLab also has a method for storing a RDFLIB graph as a parquet file.
## 🤖 assistant _(Feb 27, 2025, 08:59 AM)_
# Updated Context for Spatial Memory System Development

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
## 🧑 human _(Feb 27, 2025, 04:53 PM)_
Ok. How can I integrate a multimodal LLM like claude. We have cameras attached to the robot that can be processed by Qwen2.5-VL vision-language model that has been fine tuned for spatial reasoning and can return bounding boxes objects in the image. We also can use openstreetmap for spatial data. Can we give snapshots of the rendered map to a visual LLM?
## 🤖 assistant _(Feb 27, 2025, 04:54 PM)_
# Extended Context for Spatial Memory System Development

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
We're integrating two types of multimodal LLM capabilities:

#### Qwen2.5-VL for Real-time Scene Analysis
- Processes camera feed images from the robot
- Fine-tuned specifically for spatial reasoning
- Provides detailed object detection with bounding boxes
- Supports real-time spatial relationship understanding
- Outputs structured data about the robot's environment

#### Claude for Higher-level Reasoning
- Processes map visualizations and aggregated scene data
- Provides higher-level environmental reasoning
- Can interpret rendered maps from OpenStreetMap
- Helps with route planning and spatial understanding
- Connects visual observations to semantic knowledge

### 4. OpenStreetMap Integration
We're incorporating OpenStreetMap data to:
- Provide geographical context to indoor environments
- Enable navigation between buildings/locations
- Support global location awareness
- Visualize trajectories on meaningful maps
- Enable LLMs to reason about geographical context by showing them rendered map images

### 5. Knowledge Graph Representation
We're using RDFlib to create a knowledge graph that connects:
- Spatial locations (via S2 cells)
- Trajectory data (fixes, segments)
- Visual observations with bounding box metadata
- Extracted semantic information
- Vector embeddings of observations
- OpenStreetMap entities and relationships

### 6. Vector Storage with Polars and Pandas
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
- Bridge indoor and outdoor spatial awareness

## Tools and Libraries
- S2sphere: Python binding for S2 Geometry
- RDFlib: Python library for working with RDF and knowledge graphs
- KGLab: For storing RDFlib graphs as Parquet files
- Anthropic API: For accessing Claude's vision capabilities
- Qwen2.5-VL: For real-time spatial vision processing
- OSMnx: For working with OpenStreetMap data
- Folium/Matplotlib: For visualization of spatial data
- Polars/Pandas: For DataFrame-based vector storage and processing
- Sentence-Transformers: For generating embeddings
- nbdev: For turning notebooks into Python packages with documentation

This notebook is part of an incremental exploration to build the system step by step, with each stage working toward a complete integration of all components.
## 🧑 human _(Feb 27, 2025, 04:55 PM)_
Ok. I'm not asking to generate a context, but a specific question. How can I integrate a multimodal LLM like claude? We have cameras attached to the robot that can be processed by Qwen2.5-VL vision-language model that has been fine tuned for spatial reasoning and can return bounding boxes objects in the image. We also can use openstreetmap for spatial data. Can we give snapshots of the rendered map to a visual LLM?
## 🤖 assistant _(Feb 27, 2025, 04:56 PM)_
# Integrating Multimodal LLMs with Robot Vision and OpenStreetMap

Yes, you can definitely integrate Claude and other multimodal LLMs with your robotic system and provide them snapshots of rendered maps. Here's how you can approach this integration:

## 1. Integrating Camera Feeds through Qwen2.5-VL

Since you're already using Qwen2.5-VL for processing camera feeds and obtaining bounding boxes, you can integrate Claude as a higher-level reasoning layer:

```python
import base64
from anthropic import Anthropic
from PIL import Image
import io
import json

class MultimodalIntegration:
def __init__(self, anthropic_api_key):
self.anthropic = Anthropic(api_key=anthropic_api_key)
# Assume you have Qwen2.5-VL integration set up separately
self.qwen_processor = QwenVLProcessor()  # Your existing Qwen integration

def process_scene_with_claude(self, image_path, qwen_results=None, prompt=None):
"""
Process an image with Claude, optionally providing Qwen2.5-VL results for context
"""
# Use Qwen2.5-VL to get bounding boxes if not provided
if qwen_results is None:
qwen_results = self.qwen_processor.process_image(image_path)

# Prepare the image for Claude
with open(image_path, "rb") as img_file:
base64_image = base64.b64encode(img_file.read()).decode("utf-8")

# Default prompt if none provided
if prompt is None:
# Include Qwen bounding box results in the prompt for Claude to reason about
bounding_box_info = json.dumps(qwen_results, indent=2)

prompt = f"""
I'm a robot exploring my environment. The image shows what I'm seeing through my camera.

Another vision model (Qwen2.5-VL) has already processed this image and detected the following objects with bounding boxes:
{bounding_box_info}

Please analyze this scene and provide:
1. A general description of where I am
2. Spatial relationships between the detected objects
3. Any potential navigation considerations
4. What information should I store in my spatial memory about this location?

Format your response as structured JSON with keys: "location_type", "description", "spatial_relationships", "navigation_notes", and "memory_elements".
"""

# Call Claude API with the image and custom prompt
response = self.anthropic.messages.create(
model="claude-3-opus-20240229",
max_tokens=1000,
messages=[
{
"role": "user",
"content": [
{
"type": "text",
"text": prompt
},
{
"type": "image",
"source": {
"type": "base64",
"media_type": "image/jpeg",
"data": base64_image
}
}
]
}
],
temperature=0
)

# Parse response (assuming Claude returns JSON as requested)
try:
# Extract the JSON part from Claude's response
response_text = response.content[0].text
json_start = response_text.find('{')
json_end = response_text.rfind('}') + 1

if json_start >= 0 and json_end > json_start:
result = json.loads(response_text[json_start:json_end])
return result
else:
return {"raw_response": response_text}
except Exception as e:
return {"error": str(e), "raw_response": response.content[0].text}
```

## 2. Integrating OpenStreetMap with Claude

You can render OpenStreetMap views and send them to Claude for broader spatial context:

```python
import osmnx as ox
import matplotlib.pyplot as plt
import io
from PIL import Image

class SpatialMapIntegration:
def __init__(self, anthropic_api_key):
self.anthropic = Anthropic(api_key=anthropic_api_key)

def get_map_context(self, lat, lng, radius=300, prompt=None):
"""
Render an OpenStreetMap view and send it to Claude for analysis
"""
# Get OSM graph for the area
G = ox.graph_from_point((lat, lng), dist=radius, network_type='all')

# Get buildings and other features
buildings = ox.features_from_point((lat, lng), tags={'building': True}, dist=radius)

# Create a figure with both network and buildings
fig, ax = plt.subplots(figsize=(10, 10))
ox.plot_graph(G, ax=ax, node_size=0, edge_linewidth=0.5, show=False)
if not buildings.empty:
buildings.plot(ax=ax, facecolor='lightgray', alpha=0.7)

# Mark the robot's current position
ax.scatter(lng, lat, c='red', s=100, zorder=5)

# Save figure to bytes
buf = io.BytesIO()
plt.savefig(buf, format='jpeg', dpi=120, bbox_inches='tight')
buf.seek(0)

# Encode image for Claude
base64_image = base64.b64encode(buf.read()).decode('utf-8')

# Default prompt
if prompt is None:
prompt = f"""
I'm a robot currently at GPS coordinates ({lat}, {lng}).
This map shows my surroundings from OpenStreetMap data.

Please analyze this map and provide:
1. Description of the area I'm in (urban, residential, etc.)
2. Notable landmarks or buildings nearby
3. Potential navigation paths or points of interest
4. How this geographical context relates to indoor navigation

Format your response as JSON with keys: "area_type", "landmarks", "navigation_options", and "indoor_outdoor_context".
"""

# Call Claude API
response = self.anthropic.messages.create(
model="claude-3-opus-20240229",
max_tokens=1000,
messages=[
{
"role": "user",
"content": [
{
"type": "text",
"text": prompt
},
{
"type": "image",
"source": {
"type": "base64",
"media_type": "image/jpeg",
"data": base64_image
}
}
]
}
],
temperature=0
)

# Parse response
try:
response_text = response.content[0].text
json_start = response_text.find('{')
json_end = response_text.rfind('}') + 1

if json_start >= 0 and json_end > json_start:
result = json.loads(response_text[json_start:json_end])
return result
else:
return {"raw_response": response_text}
except Exception as e:
return {"error": str(e), "raw_response": response.content[0].text}
```

## 3. Integrating Qwen Results with Your Knowledge Graph

Now you can connect this multimodal processing to your semantic knowledge graph:

```python
def update_knowledge_graph(self, s2_cell, scene_data, map_data=None):
"""
Update the knowledge graph with scene understanding and map data
"""
# Create URI for this observation
obs_id = str(uuid.uuid4())
obs_uri = URIRef(f"{self.ST}obs_{obs_id}")

# Add observation to graph
self.graph.add((obs_uri, RDF.type, self.ST.Observation))
self.graph.add((obs_uri, self.ST.hasLocation, s2_cell))
self.graph.add((obs_uri, self.ST.observationTime, Literal(datetime.now().isoformat())))

# Add scene understanding from Claude
if "description" in scene_data:
self.graph.add((obs_uri, RDFS.comment, Literal(scene_data["description"])))

# Add detected objects from Qwen
if "detected_objects" in scene_data:
for obj in scene_data["detected_objects"]:
obj_uri = URIRef(f"{self.ST}object_{obs_id}_{obj['label'].replace(' ', '_')}")
self.graph.add((obj_uri, RDF.type, self.ST.Object))
self.graph.add((obj_uri, RDFS.label, Literal(obj["label"])))

# Add bounding box coordinates
if "bbox" in obj:
bbox = obj["bbox"]  # [x1, y1, x2, y2]
self.graph.add((obj_uri, self.ST.hasBoundingBox, Literal(json.dumps(bbox))))

# Link object to observation
self.graph.add((obs_uri, self.ST.hasDetectedObject, obj_uri))

# Add spatial relationships
if "spatial_relationships" in scene_data:
for rel in scene_data["spatial_relationships"]:
rel_uri = URIRef(f"{self.ST}rel_{obs_id}_{len(self.graph)}")
self.graph.add((rel_uri, RDF.type, self.ST.SpatialRelation))
self.graph.add((rel_uri, self.ST.hasSubject, Literal(rel["subject"])))
self.graph.add((rel_uri, self.ST.hasRelation, Literal(rel["relation"])))
self.graph.add((rel_uri, self.ST.hasObject, Literal(rel["object"])))
self.graph.add((obs_uri, self.ST.hasSpatialRelation, rel_uri))

# Add map context if available
if map_data and "landmarks" in map_data:
for landmark in map_data["landmarks"]:
landmark_uri = URIRef(f"{self.ST}landmark_{obs_id}_{landmark['name'].replace(' ', '_')}")
self.graph.add((landmark_uri, RDF.type, self.ST.Landmark))
self.graph.add((landmark_uri, RDFS.label, Literal(landmark["name"])))
if "distance" in landmark:
self.graph.add((landmark_uri, self.ST.hasDistance, Literal(landmark["distance"])))
if "direction" in landmark:
self.graph.add((landmark_uri, self.ST.hasDirection, Literal(landmark["direction"])))

# Link landmark to observation
self.graph.add((obs_uri, self.ST.hasNearbyLandmark, landmark_uri))

# Store vector embeddings in Polars/Pandas DataFrame
if hasattr(self, 'embeddings_df'):
# Generate embedding using model
embedding = self.model.encode(scene_data.get("description", ""))

# Add to DataFrame
new_row = {
"uri": str(obs_uri),
"s2_cell": str(s2_cell),
"timestamp": datetime.now().isoformat(),
"embedding": embedding.tolist(),
"metadata": json.dumps(scene_data)
}

# Append to DataFrame (Polars-specific syntax)
self.embeddings_df = self.embeddings_df.vstack(pl.DataFrame([new_row]))

return obs_uri
```

## 4. Bringing It All Together in a Robot Spatial Memory System

Here's a sketch of how to integrate this into your overall system:

```python
class RobotSpatialMemory:
def __init__(self, anthropic_api_key, memory_path="robot_memory"):
# Set up knowledge graph
self.kg = RDFlib.Graph()
self.setup_ontology()

# Set up multimodal integration
self.vision_processor = MultimodalIntegration(anthropic_api_key)
self.map_processor = SpatialMapIntegration(anthropic_api_key)

# Set up embeddings storage with Polars
self.embeddings_df = pl.DataFrame({
"uri": [],
"s2_cell": [],
"timestamp": [],
"embedding": [],
"metadata": []
})

# Load existing data if available
self.load_memory(memory_path)

def process_camera_feed(self, image_path, lat, lng):
"""Process camera feed and update spatial memory"""
# Get S2 cell for location
s2_cell = self.get_s2_cell(lat, lng)

# Process with Qwen2.5-VL
qwen_results = self.vision_processor.qwen_processor.process_image(image_path)

# Get higher-level understanding with Claude
claude_results = self.vision_processor.process_scene_with_claude(
image_path, qwen_results=qwen_results
)

# Get map context if outdoors or near building entrances
map_context = None
if claude_results.get("location_type") == "outdoor" or \
"entrance" in claude_results.get("description", "").lower():
map_context = self.map_processor.get_map_context(lat, lng)

# Update knowledge graph
observation_uri = self.update_knowledge_graph(s2_cell, {
**claude_results,
"detected_objects": qwen_results.get("objects", [])
}, map_context)

# Save updated memory
self.save_memory()

return {
"observation_uri": str(observation_uri),
"scene_understanding": claude_results,
"object_detection": qwen_results,
"map_context": map_context
}

def query_spatial_memory(self, query_image=None, lat=None, lng=None, text_query=None):
"""Query spatial memory using multimodal input"""
results = []

# If we have an image, get embedding and find similar observations
if query_image:
# Process query image with Claude
query_results = self.vision_processor.process_scene_with_claude(query_image)

# Generate query embedding
query_embedding = self.model.encode(query_results.get("description", ""))

# Search in embeddings DataFrame using vector similarity
# For Polars, you might need custom function or extension
# Here's a simplified version that calculates cosine similarity
def cosine_similarity(emb):
emb_array = np.array(emb)
return np.dot(emb_array, query_embedding) / (
np.linalg.norm(emb_array) * np.linalg.norm(query_embedding)
)

# Apply similarity function and sort
with pl.Config(temporaries_lifetime="memory"):
similarities = self.embeddings_df.with_columns(
pl.col("embedding").map_elements(cosine_similarity).alias("similarity")
)

top_matches = similarities.sort("similarity", descending=True).head(5)

# Get detailed information from knowledge graph
for row in top_matches.to_dicts():
uri = row["uri"]
# Query graph for details about this observation
details = self.query_observation_details(uri)
results.append({
"uri": uri,
"similarity": row["similarity"],
"details": details
})

# If we have location, find nearby observations
if lat is not None and lng is not None:
nearby = self.query_by_location(lat, lng)
results.extend(nearby)

return results
```

## Important Considerations

1. **Processing Pipeline**: This integration gives you a two-stage visual processing pipeline:
- Qwen2.5-VL provides detailed object detection with bounding boxes
- Claude provides higher-level scene understanding and reasoning

2. **Map Rendering**: You can render OSM data in different ways:
- Static map images with matplotlib/folium
- Interactive visualizations that highlight specific features
- Combined views that overlay robot trajectory on the map

3. **Cross-Modal Reasoning**: Claude can reason across different modalities:
- Camera feed images
- Map renderings
- Previous observations from the knowledge graph
- Trajectory data

4. **Memory Organization**: Use S2 cells to organize spatial memory at different scales:
- Fine-grained cells for room-level observations
- Coarser cells for building/neighborhood context

5. **Vector Storage in DataFrames**: Polars offers great performance for vector operations:
- Use Arrow arrays for efficient storage of embeddings
- Leverage Polars' vectorized operations for faster similarity searches
- Save to Parquet for persistence with good compression

By integrating Qwen2.5-VL for detailed spatial perception and Claude for higher-level reasoning, along with OpenStreetMap for geographical context, you're creating a comprehensive spatial memory system that bridges robotics, computer vision, NLP, and knowledge representation.