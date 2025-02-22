# Prerequisites

To run the code in this repo, you need to have a working Python
installation with the following packages installed (with pip in this
case):

``` {bash}
#| eval: false
pip install matplotlib pandas shapely geopandas osmnx networkx scipy folium mapclassify
```

# Data Preparation and Processing

The process is divided into several stages, each with a specific
purpose, collectively contributing to the creation of a streamlined
representation of the road network that emphasises essential
cycling-related information.

The process requires two main input datasets. The first is a GeoJSON
file named “rnet_princes_street.geojson,” read into the system as
input_detailed. This dataset represents the more intricate and detailed
spatial information related to cycle commuting (stored in
input_detailed\[‘value’\]). The second is another GeoJSON file named
“Edc_Roadlink.geojson,” read as input_simple, serving as the simplified
road network onto which the detailed information will be projected.

## 1. Preprocessing on input_detailed

The preprocessing stage focuses on cleaning and restructuring the
input_detailed data to set a unique index for each road. It involves
merging spatially connected lines within the dataset, grouped by ‘Value’
and ‘Quietness’ attributes. A custom function, merge_directly_connected,
is implemented for this purpose. The function uses a while loop that
continuously joins lines if the start and end coordinates match,
resulting in a set of merged lines. The purpose of this action is to
eliminate redundancy and ensure that each road segment has a unique
identifier, facilitating further analysis.

## 2. Segmentation of input_detailed

After preprocessing, the processed input_detailed data is then segmented
using the split_line_at_angles function. This function computes the
direction of each vector within a line and identifies indices where the
angle between consecutive vectors exceeds a given threshold. The line is
then split at these points into individual segments. The purpose of this
action is to ensure that the following geospatial data analysis can
produce more refined results by recognising and preserving significant
changes in direction.

Together, these stages transform the detailed spatial data into a format
suitable for merging with the simplified road network.

# Projecting cycle commuting data from input_detailed to input_simple

The process of projecting cycle commuting data from the more detailed
spatial information (input_detailed) to a simplified road network
(input_simple) is carried out through a series of well-defined stages:

## 1. Creating a Buffer around input_simple

A new GeoDataFrame gdf_buffer is created around the geometries in
input_simple using a predefined buffer size. This buffer serves as a
zone within which lines from input_detailed will be analyzed and matched
to the corresponding roads in input_simple.

## 2. Identifying Lines within the Buffer and Processing ‘value’ Attribute

The first major step in this stage is to find the lines from
input_detailed that fall within the buffer. The ‘value’ attribute
associated with these lines is then processed according to three
specific requirements:

Requirement 1: If multiple lines with the same ‘Ori_index’ are within
the same buffer, the mean ‘value’ of these lines is added to the buffer.

Requirement 2: If lines with the same ‘Ori_index’ are within different
buffers, the lengths of the lines are compared, and the ‘value’ is added
to the buffer with the longer lines.

Requirement 3: The ‘value’ of each line is added only once to ensure
accuracy.

In this section, instead of using standard geopandas functions like
gpd.sjoin, the code implements a series of loops and dictionary
structures. This custom approach meets the specific requirements,
storing the index of lines within the buffer in
gdf_buffered\[‘Line_index_from_gdf_Within’\] and ensuring that each
‘value’ is added only once.

## 3. Identifying Lines that Intersect with the Buffer

Lines from input_detailed that intersect with the buffer are also
identified. This stage involves calculating the angle between
intersecting lines and the corresponding lines in input_simple. The
functions get_vector and calculate_angle are utilised to compute the
vectors and angles, respectively.

If the angle is less than 25 degrees, the intersecting lines are
considered to be in the same direction as the corresponding line in
input_simple, and their ‘value’ is added to the attribute
gdf_buffered\[‘value_sum’\]. The index of intersecting lines is recorded
in gdf_buffered\[‘Line_index_from_gdf_Intersect’\].

## 4. Final Data Aggregation

In the final stage of the process, the attributes from gdf_buffered are
matched to input_simple using the index. A join operation then updates
input_simple with the attributes value_sum, effectively merging the
processed cycle commuting data onto the simplified road network.

``` python
import matplotlib.pyplot as plt
import pandas as pd
import math
from typing import List, Tuple
from shapely.geometry import LineString
import geopandas as gpd
import osmnx as ox
import numpy as np
from scipy.spatial.distance import pdist, squareform
from shapely.geometry import Point, LineString, Polygon, MultiLineString
import networkx as nx
from pyproj import CRS
import folium
import os
from numpy.linalg import norm
from collections import defaultdict
from shapely.ops import unary_union

# Calculate total length of linestrings in a GeoDataFrame 
def calculate_total_length(gdf, crs="EPSG:4326"):
    # Copy the GeoDataFrame
    gdf_projected = gdf.copy()

    # Change the CRS to a UTM zone for more accurate length calculation
    gdf_projected = gdf_projected.to_crs(crs)

    # Calculate the length of each line
    gdf_projected["length"] = gdf_projected.length

    # Calculate the total length
    total_length = gdf_projected["length"].sum()

    return total_length

# Plot GeoDataFrame and label each feature with its index
def plot_geodataframe_with_labels(gdf, gdf_name):

    # Create a new figure
    fig, ax = plt.subplots(figsize=(10, 10))

    # Plot the GeoDataFrame
    gdf.plot(ax=ax)

    # Add labels for each line with its index
    for x, y, label in zip(gdf.geometry.centroid.x, gdf.geometry.centroid.y, gdf.index):
        ax.text(x, y, str(label), fontsize=12)
    plt.savefig(f"pics/{gdf_name}.jpg")
    # Display the plot
    plt.show()

# Create interactive map from one or more GeoDataFrames  
def plot_geodataframes(*args, colors=['red', 'blue', 'green'], line_widths=[3.5, 2.5,1.5], marker_sizes=[10, 10, 10], map_type="OpenStreetMap"):
    """
    Args:
        *args: One or more (name, GeoDataFrame) tuples 
        colors: List of colors for lines
        line_widths: List of line widths
        map_type: Folium map type
    Returns:
        Folium map object
    """
    # Prepare gdfs and their names
    gdfs = [arg[1] for arg in args]
    names = [arg[0] for arg in args]

    # Initialize the map to the first point of the first geodataframe
    start_point = gdfs[0].iloc[0].geometry.centroid.coords[0]
    m = folium.Map(location=[start_point[1], start_point[0]], zoom_start=15)

    if map_type == "Esri Satellite":
        esri = folium.TileLayer(
            tiles="https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}",
            attr="Esri",
            name="Esri Satellite",
            control=True
        )
        esri.add_to(m)

    # Create feature groups for each geodataframe and add them to the map
    for i, gdf in enumerate(gdfs):
        fg = folium.FeatureGroup(name=names[i], show=False)
        for _, row in gdf.iterrows():
            if row.geometry.geom_type == 'Point':
                marker = folium.Marker(location=[row.geometry.y, row.geometry.x], 
                                       icon=folium.Icon(color=colors[i % len(colors)]))
                fg.add_child(marker)
            elif row.geometry.geom_type == 'LineString':
                line = folium.vector_layers.PolyLine(locations=[[p[1], p[0]] for p in list(row.geometry.coords)],
                                                      color=colors[i % len(colors)],
                                                      weight=line_widths[i % len(line_widths)])
                fg.add_child(line)

            elif row.geometry.geom_type == 'Polygon':
                polygon = folium.vector_layers.Polygon(locations=[[p[1], p[0]] for p in list(row.geometry.exterior.coords)],
                                                       color=colors[i % len(colors)],
                                                       fill=True)
                fg.add_child(polygon)
            elif row.geometry.geom_type == 'MultiLineString':
                for line_geom in row.geometry.geoms:
                    coordinates = [list(p) for p in line_geom.coords]
                    line = folium.vector_layers.PolyLine(locations=[[p[1], p[0]] for p in coordinates],
                                                        color=colors[i % len(colors)],
                                                        weight=line_widths[i % len(line_widths)])
                    fg.add_child(line)                
        m.add_child(fg)

    # Add layer control to the map
    folium.LayerControl().add_to(m)

    return m

# Filter GeoDataFrame based on column conditions  
def filter_data(gdf, conditions):
    """
    Filter a GeoDataFrame based on multiple conditions.

    """
    for column, condition in conditions.items():
        gdf = gdf[condition(gdf[column])]
    return gdf

# Buffer input geometries by specified distance
def create_buffer(gdf, buffer_size = 0.00002):
    gdf_buffered = gdf.copy()
    gdf_buffered.geometry = gdf.geometry.buffer(buffer_size)
    # gdf_buffered.to_file("data/gdf_buffered.geojson", driver='GeoJSON')
    return  gdf_buffered

# Get start and end points from (Multi)LineString  
def get_vector(line):
    if isinstance(line, LineString):
        start, end = line.coords[:2]
    else:  # for MultiLineStrings, just use the first line
        start, end = line.geoms[0].coords[:2]
    return [end[0] - start[0], end[1] - start[1]]

# Calculate angle between two vectors 
def calculate_angle(vector1, vector2):
    dot_product = vector1[0] * vector2[0] + vector1[1] * vector2[1]
    magnitude_product = math.sqrt(vector1[0]**2 + vector1[1]**2) * math.sqrt(vector2[0]**2 + vector2[1]**2)
    cos_angle = dot_product / magnitude_product
    angle = math.degrees(math.acos(cos_angle))
    return angle

# Modified function to split a line into segments based on an angle threshold and retain the 'value', 'Quietness', and original index
def split_line_at_angles(line, value, quietness, original_index, threshold=30):
    segments = []
    if isinstance(line, LineString):
        coords = np.array(line.coords)
        # Compute the direction of each vector
        vectors = np.diff(coords, axis=0)
        directions = np.arctan2(vectors[:, 1], vectors[:, 0])
        # Compute the angle between each pair of vectors
        angles = np.diff(directions)
        # Convert the angles to degrees and take absolute values
        angles = np.abs(np.degrees(angles))
        # Identify the indices where the angle exceeds the threshold
        split_indices = np.where(angles > threshold)[0] + 1
        # Split the line at the points corresponding to the split indices
        last_index = 0
        for index in split_indices:
            segment = LineString(coords[last_index:index + 1])
            segments.append((segment, value, quietness, original_index))
            last_index = index
        # Include all remaining parts of the line after the last split point
        segment = LineString(coords[last_index:])
        segments.append((segment, value, quietness, original_index))
    elif isinstance(line, MultiLineString):
        # Handle each LineString in the MultiLineString separately
        for geom in line.geoms:
            segments.extend(split_line_at_angles(geom, value, quietness, original_index, threshold))
    else:
        raise ValueError(f"Unexpected geometry type: {type(line)}")

    return segments

# Calculate angle between two line segments 
def calculate_angle(line1, line2):
    # Define the vectors
    vector1 = np.array(line1[1]) - np.array(line1[0])
    vector2 = np.array(line2[1]) - np.array(line2[0])
    
    # Compute the dot product
    dot_product = np.dot(vector1, vector2)
    
    # Compute the magnitudes of the vectors
    magnitude1 = np.sqrt(np.dot(vector1, vector1))
    magnitude2 = np.sqrt(np.dot(vector2, vector2))
    
    # Compute the angle between the vectors in radians
    angle_rad = np.arccos(dot_product / (magnitude1 * magnitude2))
    
    # Convert the angle to degrees
    angle_deg = np.degrees(angle_rad)
    
    return angle_deg

# Concat parallel linestrings into single geometry
def filter_parallel_lines_concat(gdf, name, angle_tolerance=25):
    # Filter the GeoDataFrame by the 'name' column
    filtered_gdf = gdf[gdf['name'] == name]

    # Create a list to store the parallel lines
    parallel_lines = []

    # Iterate through each pair of lines
    for i in range(len(filtered_gdf)):
        for j in range(i+1, len(filtered_gdf)):
            # Get the lines
            line1 = list(filtered_gdf.iloc[i].geometry.coords)
            line2 = list(filtered_gdf.iloc[j].geometry.coords)

            # Calculate the angle between the lines
            angle = calculate_angle(line1, line2)

            # If the angle is close to 0 or 180 degrees, add the lines to the list
            if abs(angle) <= angle_tolerance or abs(angle - 180) <= angle_tolerance:
                parallel_lines.append(filtered_gdf.iloc[i:i+1])
                parallel_lines.append(filtered_gdf.iloc[j:j+1])

    # Combine the lines into a new GeoDataFrame using pd.concat
    parallel_gdf = pd.concat(parallel_lines).drop_duplicates()

    return parallel_gdf

# Distance between two points
def calculate_distance(point1, point2):
    return Point(point1).distance(Point(point2))

# Plot results of line within or intersect with buffer
def plot_buffer_with_lines(gdf_buffered, gdf, buffer_index='all', relation='intersect'):
    column_name = 'Line_index_from_gdf_Within' if relation == 'within' else 'Line_index_from_gdf_Intersect'

    fig, ax = plt.subplots(figsize=(12, 8))

    def plot_single_buffer(buffer_idx, buffered_geom):
        indices_related_buffer = buffered_geom[column_name]
        gpd.GeoSeries([buffered_geom['geometry']]).plot(ax=ax, edgecolor='blue', facecolor='none')

        if indices_related_buffer:
            gdf.loc[indices_related_buffer].plot(ax=ax, color='red')
            for idx in indices_related_buffer:
                line_value = gdf.loc[idx, 'value']
                annotation_text = f"{idx} - {line_value}"
                centroid = gdf.loc[idx, 'geometry'].centroid.coords[0]
                plt.text(centroid[0], centroid[1] + 0.00005, annotation_text, fontsize=9, color='red', ha='center')
        else:
            plt.text(buffered_geom['geometry'].centroid.x, buffered_geom['geometry'].centroid.y - 0.00015, "No lines " + relation, fontsize=9, color='blue', ha='center')

        plt.text(buffered_geom['geometry'].centroid.x, buffered_geom['geometry'].centroid.y - 0.00035, f"Buffer Index: {buffer_idx}", fontsize=9, color='blue', ha='center')

    if buffer_index == 'all':
        for buffer_idx, buffered_geom in gdf_buffered.iterrows():
            plot_single_buffer(buffer_idx, buffered_geom)
        plt.title(f'Buffered Geometries (Blue) with {relation.capitalize()} Lines (Red)')
    elif isinstance(buffer_index, int):
        plot_single_buffer(buffer_index, gdf_buffered.loc[buffer_index])
        plt.title(f'Buffered Geometry (Blue) with {relation.capitalize()} Lines (Red) for Buffer Index {buffer_index}')

    plt.xlabel('Longitude')
    plt.ylabel('Latitude')
    plt.show()

# Function to merge directly connected line segments within a group
def merge_directly_connected(group):
    lines = list(group['geometry'])
    merged_lines = []
    while lines:
        current_line = lines.pop(0)
        changed = True
        while changed:
            changed = False
            for i, line in enumerate(lines):
                if line.coords[0] == current_line.coords[-1]:
                    current_line = LineString(list(current_line.coords) + list(line.coords[1:]))
                    lines.pop(i)
                    changed = True
                    break
                elif line.coords[-1] == current_line.coords[0]:
                    current_line = LineString(list(line.coords[:-1]) + list(current_line.coords))
                    lines.pop(i)
                    changed = True
                    break
        merged_lines.append(current_line)
    return merged_lines
```

``` python
# # Define the centre point of AoT from OSM
# point = (55.952227 , -3.1959271)
# distance = 1300  # in meters

# # Only download if the data/edges.shp file does not exist:
# if not os.path.exists("data/edges.shp"):
#     # Download the road network data
#     graph = ox.graph_from_point(point, dist=distance, network_type='all')

#     # Save the road network as a shapefile
#     ox.save_graph_shapefile(graph, filepath=r'data/')

# Read in data from CycleStreets 
gdf = gpd.read_file("data/rnet_princes_street.geojson")
gdf = gdf.rename(columns={'commute_fastest_bicycle_go_dutch': 'value'})

# Group the GeoDataFrame by 'value' and 'Quietness' columns
grouped_gdf = gdf.groupby(['value', 'Quietness'])

# Iterate through the groups and merge directly connected lines
merged_lines_final = []
for (value, quietness), group in grouped_gdf:
    connected_lines = merge_directly_connected(group)
    for line in connected_lines:
        merged_lines_final.append({'value': value, 'Quietness': quietness, 'geometry': line})

# Create a new GeoDataFrame with the merged and directly connected lines
gdf_merged_directly_connected_final = gpd.GeoDataFrame(merged_lines_final, geometry='geometry')
gdf_merged_directly_connected_final['length'] = gdf_merged_directly_connected_final['geometry'].length

# gdf_merged_directly_connected_final.to_file("data/gdf_merged_directly_connected_final.geojson", driver='GeoJSON')

gdf = gdf_merged_directly_connected_final


# TODO: check total length after network simplification
total_distance_traveled = round(sum(gdf['value'] * gdf['length']))
total_distance_traveled


# Applying the split function to the gdf GeoDataFrame
segments_list_modified = []
for index, row in gdf.iterrows():
    segments = split_line_at_angles(row['geometry'], row['value'], row['Quietness'], index)
    segments_list_modified.extend(segments)

# Creating a new GeoDataFrame with the individual segments and additional attributes
segments_gdf_modified = gpd.GeoDataFrame(segments_list_modified, columns=['geometry', 'value', 'Quietness', 'Ori_index'])
segments_gdf_modified['value'].sum(), segments_gdf_modified.shape, segments_gdf_modified.head()
segments_gdf_modified.shape
segments_gdf_modified.to_file("data/segments_gdf_modified.geojson", driver='GeoJSON')

# # Filter the data with conditions
# conditions = {
#     'value': lambda x: x != 0
# }

# filtered_gdf = filter_data(gdf, conditions)

# Read in simplified OS Road map data 
gdf_road_simplified = gpd.read_file("data/Edc_Roadlink.geojson")
gdf_road_simplified = gdf_road_simplified[['identifier', 'geometry']]
gdf_road_simplified.crs = "EPSG:4326"


# Plotting the geodataframes
map = plot_geodataframes(('gdf', gdf), ('gdf_road_simplified', gdf_road_simplified),('segments_gdf_modified', segments_gdf_modified),('gdf_merged_directly_connected_final', gdf_merged_directly_connected_final),
                          colors=['blue', 'red', 'green','black'], line_widths=(3.0, 2.0, 1.0,0.6), map_type="Esri Satellite")
# map
```

:::
{#join-attribute-(commute_fastest_bicycle_go_dutch-from-gdf-to-gdf_road_simplified
.cell execution_count=3}

``` python
gdf = segments_gdf_modified
all_lines = gdf.index.tolist()

# Step 1: Create a buffer around the geometries in gdf_road_simplified
gdf_buffered = create_buffer(gdf_road_simplified, buffer_size=0.00022)

# Step 2.1: identifying all lines in gdf that are within the buffer
# Initializing the required columns

gdf_buffered['value_sum'] = 0
gdf_buffered['Line_index_from_gdf_Within'] = None
all_lines_within_buffer = []

# Dictionary to store lines across different buffers grouped by 'Ori_index'
# Iterate through the buffered geometries to find lines within the buffer and sum their 'value'

lines_across_buffers_by_ori_index = defaultdict(list)

for buffer_index, buffered_geom in gdf_buffered.iterrows():
    # Dictionary to store lines grouped by 'Ori_index' within the current buffer
    lines_within_buffer_by_ori_index = defaultdict(list)

    # Find lines within the buffer and group them by 'Ori_index'
    indices_within_buffer = []
    for line_index, line_geom in gdf.iterrows():
        if line_geom['geometry'].within(buffered_geom['geometry']):
            lines_within_buffer_by_ori_index[line_geom['Ori_index']].append((line_index, line_geom['value'], line_geom['geometry'].length))
            indices_within_buffer.append(line_index)

    # Store the lines grouped by 'Ori_index' for comparison across buffers
    for ori_index, lines in lines_within_buffer_by_ori_index.items():
        lines_across_buffers_by_ori_index[ori_index].append((buffer_index, lines))

    # Placeholder for value sum; actual sum will be calculated later
    gdf_buffered.at[buffer_index, 'Line_index_from_gdf_Within'] = indices_within_buffer
    all_lines_within_buffer.extend(indices_within_buffer)

# Set to keep track of processed lines
processed_lines = set()

# Iterate through lines grouped by 'Ori_index' across buffers
for ori_index, buffers_with_lines in lines_across_buffers_by_ori_index.items():
    if len(buffers_with_lines) == 1:
        # Requirement 1: All lines with the same 'Ori_index' are within the same buffer
        buffer_index, lines = buffers_with_lines[0]
        mean_value = sum(line[1] for line in lines if line[0] not in processed_lines) / len(lines)
        gdf_buffered.at[buffer_index, 'value_sum'] += mean_value
        processed_lines.update(line[0] for line in lines)
    else:
        # Requirement 2: Lines with the same 'Ori_index' are within different buffers
        max_length_buffer = max(buffers_with_lines, key=lambda x: sum(line[2] for line in x[1] if line[0] not in processed_lines))
        mean_value = sum(line[1] for line in max_length_buffer[1] if line[0] not in processed_lines) / len(max_length_buffer[1])
        gdf_buffered.at[max_length_buffer[0], 'value_sum'] += mean_value
        processed_lines.update(line[0] for line in max_length_buffer[1])


plot_buffer_with_lines(gdf_buffered, gdf, buffer_index=1477, relation='within')


# Helper function to find examples that meet Requirement 1 and Requirement 2
# Requirement 1: If there are multiple lines with the same 'Ori_index' value within the same buffer, the mean value of those lines should be added to the buffer.
# Requirement 2: If there are lines with the same 'Ori_index' value but within different buffers, compare the lengths of the lines. If the length of lines with the same 'Ori_index' in one buffer is shorter than the lines in another buffer, only add the value of the lines in the buffer with the longer lines.

def find_examples():
    # Dictionary to store examples
    examples = {"requirement_1": [], "requirement_2": []}

    # Iterate through buffers to find examples for Requirement 1
    for buffer_index, buffer_row in gdf_buffered.iterrows():
        lines_within_buffer = gdf.loc[buffer_row['Line_index_from_gdf_Within']]
        grouped_by_ori_index = lines_within_buffer.groupby('Ori_index')
        for ori_index, group in grouped_by_ori_index:
            if len(group) > 1:
                examples["requirement_1"].append((buffer_index, ori_index, group))

    # Iterate through unique 'Ori_index' values to find examples for Requirement 2
    unique_ori_indexes = gdf['Ori_index'].unique()
    for ori_index in unique_ori_indexes:
        lines_with_same_ori_index = gdf[gdf['Ori_index'] == ori_index]
        buffers_with_same_ori_index = gdf_buffered[gdf_buffered['Line_index_from_gdf_Within'].apply(lambda x: any(idx in x for idx in lines_with_same_ori_index.index))]
        if len(buffers_with_same_ori_index) > 1:
            examples["requirement_2"].append((ori_index, buffers_with_same_ori_index))

    return examples

# Finding examples for the specific requirements
examples = find_examples()

# Displaying the first example for Requirement 1
example_req1 = examples["requirement_1"][0] if examples["requirement_1"] else None
example_req1_buffer_index, example_req1_ori_index, example_req1_group = example_req1 if example_req1 else (None, None, None)

# Displaying the first example for Requirement 2
example_req2 = examples["requirement_2"][0] if examples["requirement_2"] else None
example_req2_ori_index, example_req2_buffers = example_req2 if example_req2 else (None, None)

example_req1
# plot_buffer_with_lines(gdf_buffered, gdf, buffer_index=405, relation='within')
example_req2

plot_buffer_with_lines(gdf_buffered, gdf, buffer_index=405, relation='within')

# Examples
plot_buffer_with_lines(gdf_buffered, gdf, buffer_index=614, relation='within')
gdf_buffered.iloc[614]['value_sum']
plot_buffer_with_lines(gdf_buffered, gdf, buffer_index=463, relation='within')
plot_buffer_with_lines(gdf_buffered, gdf, buffer_index=1476, relation='within')
gdf_buffered.iloc[463]['value_sum']

##############################################################
# Step 2.2: identifying all lines in gdf that intersect with the buffer
all_lines_intersect_buffer = []
gdf_buffered['Line_index_from_gdf_Intersect'] = None
for buffer_index, buffered_geom in gdf_buffered.iterrows():
    indices_intersecting_buffer = []  # Indices of lines intersecting with the buffer
    value_sum = gdf_buffered.loc[buffer_index, 'value_sum']  # Existing value_sum for the buffer
    corresponding_road_line = gdf_road_simplified.loc[buffer_index]['geometry']  # Corresponding road line
    for line_index, line_geom in gdf.iterrows():
        # Skip lines that are already within any buffer
        if line_index in all_lines_within_buffer:
            continue
        if line_geom['geometry'].intersects(buffered_geom['geometry']):
            vector1 = get_vector(line_geom['geometry'])
            vector2 = get_vector(corresponding_road_line)
            angle = calculate_angle(vector1, vector2)
            if angle < 25:
                value_sum += line_geom['value']
            indices_intersecting_buffer.append(line_index)
    gdf_buffered.at[buffer_index, 'value_sum'] = value_sum
    gdf_buffered.at[buffer_index, 'Line_index_from_gdf_Intersect'] = indices_intersecting_buffer
    all_lines_intersect_buffer.extend(indices_intersecting_buffer)

plot_buffer_with_lines(gdf_buffered, gdf, buffer_index=614, relation='intersect')

gdf_buffered.iloc[1477]['value_sum']

# Finding the missed lines using buffer methods
missing_lines = set(all_lines) - set(all_lines_within_buffer) - set(all_lines_intersect_buffer)
len(missing_lines)
Missed_gdf = gdf.loc[list(missing_lines)]

map = plot_geodataframes(('gdf', gdf), ('gdf_buffered', gdf_buffered),('Missed_gdf', Missed_gdf),
                          colors=['blue', 'red', 'black'], line_widths=(3.0, 2.5, 5), map_type="Esri Satellite")
map

##############################################################

# Extracting the details for the provided example
example_buffer_index = 959
example_line_index_gdf = 962
example_line_index_road_simplified = 962

# Getting the geometries for the example
example_buffer_geom = gdf_buffered.loc[example_buffer_index]['geometry']
example_line_gdf = gdf.loc[example_line_index_gdf]['geometry']
example_line_road_simplified = gdf_road_simplified.loc[example_line_index_road_simplified]['geometry']

# Calculating the vector and angle for the example
vector_gdf = get_vector(example_line_gdf)
vector_road_simplified = get_vector(example_line_road_simplified)
example_angle = calculate_angle(vector_gdf, vector_road_simplified)

# Checking the condition for adding the 'value'
add_value_condition = example_angle < 25

# Getting the 'value_sum' and 'Line_index_from_gdf_Intersect' for the corresponding buffer
example_value_sum = gdf_buffered.loc[example_buffer_index]['value_sum']
example_line_indices_intersect = gdf_buffered.loc[example_buffer_index]['Line_index_from_gdf_Intersect']

example_angle, add_value_condition, example_value_sum, example_line_indices_intersect
##############################################################

# Step 3: Match the attributes from gdf_buffered to gdf_road_simplified using the index, and then update gdf_road_simplified with these attributes.
# Joining gdf_buffered with gdf_road_simplified on the index to combine attributes
gdf_road_simplified_updated = gdf_road_simplified.join(gdf_buffered[['value_sum', 'Line_index_from_gdf_Within', 'Line_index_from_gdf_Intersect']])

# Displaying the first few rows of the updated gdf_road_simplified DataFrame
gdf_road_simplified_updated.head()

total_distance_traveled = round(sum(gdf_road_simplified_updated['value_sum'] * gdf_road_simplified_updated['geometry'].length))
total_distance_traveled
```

:::

The code above generates output that has values from the detailed
network merged onto the simplified network. The output is saved as a
GeoJSON file.

To recap, code below reads the input and output GeoJSON files and
displays the first few rows of the output.

``` python
input_detailed = gpd.read_file("data/rnet_princes_street.geojson")
input_simple = gpd.read_file("data/Edc_Roadlink.geojson")
input_detailed = input_detailed.to_crs('EPSG:27700')
input_simple = input_simple.to_crs('EPSG:27700')
# Union of input_simple:
input_detailed_union = input_detailed['geometry'].unary_union
# Buffer of 10 m around the union:
input_detailed_buffer = input_detailed_union.buffer(30)
# Convert to GeoDataFrame:
input_detailed_buffer = gpd.GeoDataFrame([input_detailed_buffer])
input_detailed_buffer.set_geometry(0, inplace=True)
# Set CRS:
input_detailed_buffer.crs = input_detailed.crs
# the intersection of the simple network with the detailed buffered network:
input_intersection = gpd.overlay(input_simple, input_detailed_buffer, how='intersection')
# Plot both input and output networks with values represented by colour:
input_detailed.plot(linewidth=input_detailed['commute_fastest_bicycle_go_dutch']/1000, cmap='Blues', legend=True);
input_intersection.plot(color='red', linewidth=2);
```

![](buffer-aggregate_files/figure-commonmark/cell-6-output-1.png)

![](buffer-aggregate_files/figure-commonmark/cell-6-output-2.png)

The output is as follows:

``` python
gdf_output = gpd.read_file("data/gdf_road_simplified_updated.geojson")
gdf_output.head()
gdf_output[['value_sum']].mean()
gdf_output_projected = gdf_output.to_crs('EPSG:27700')
# calculate the length of each segment:
gdf_output_projected['length'] = gdf_output_projected['geometry'].length
```

The total distance travelled on the input and output networks are as
follows:

``` python
total_distance_traveled_input = round(sum(input_detailed['commute_fastest_bicycle_go_dutch'] * input_detailed['geometry'].length))
round(total_distance_traveled_input / 1000)
```

    17159

And on the output network:

``` python
total_distance_traveled_output = round(sum(gdf_output_projected['value_sum'] * gdf_output_projected['geometry'].length))
round(total_distance_traveled_output / 1000)
```

    43926
