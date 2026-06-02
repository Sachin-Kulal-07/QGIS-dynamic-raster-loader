# QGIS On-Demand Dynamic Raster Loader

A high-performance QGIS workflow tool designed to eliminate software lag when managing massive imagery datasets. By leveraging custom Python-backed QGIS Attribute Actions embedded within a lightweight vector index layer, users can dynamically load and unload high-resolution raster tiles on demand with a single click.

---

## 💡 The Problem & The Solution

### The Challenge
When working with large-scale geographic projects (e.g., thousands of high-resolution aerial orthophotos, drone surveys, or continuous 360° panoramic tile networks), loading all layers simultaneously severely degrades GIS performance, exhausts RAM, and slows canvas rendering to a crawl. 

### The Game-Changing Solution
Instead of loading the heavy imagery directly, this tool uses a lightweight vector tile index (polygon boundary grid) as a proxy. The underlying `.qml` style embeds two robust Python-based actions directly into the QGIS user interface:
1. **Load Image:** Dynamically locates the file path from the feature attributes, creates a valid `QgsRasterLayer`, bundles it into a dedicated `"PS group"` layer tree, and loads it into the map canvas on demand.
2. **Remove Image:** Instantly unloads the specific raster layer from both the workspace tree and memory caching when visualization is no longer required.

---

## 🛠️ Key Technical Features

* **Memory-Efficient Workflows:** Keeps the QGIS map canvas lightning-fast by ensuring only actively scrutinized tiles are loaded into memory.
* **Smart Layer Tree Structuring:** Automatically checks for, creates, and organizes loaded tiles into a standardized layer group (`"PS group"`), keeping the project sidebar clean.
* **Stellar Error Handling:** Employs explicit validation routines via `QgsRasterLayer.isValid()` and leverages the native QGIS `iface.messageBar()` to provide clear success or critical error alerts to the user.
* **Auto-Focus Interaction:** Automatically switches active layer focus back to the index grid after loading a tile, allowing seamless, continuous click-to-load interactions without manual navigation.

---

## 📖 Setup & Usage Guide
* **Prepare Your Data:** Ensure you have a vector index layer (e.g., a shapefile or GeoPackage grid) where each polygon represents a raster tile boundary, and contains an attribute field named location storing the absolute or relative file path to the corresponding image.

* **Apply the Style:** Right-click your vector index layer in QGIS, go to Properties > Symbology, click the Style dropdown menu at the bottom, select Load Style..., and choose ps_loader_v3.qml.

* **Run the Actions:** Select the Run Feature Action tool from the main QGIS toolbar (or press A).

* **Click** any tile boundary polygon on your map canvas to trigger Load Image or Remove Image.

## 💻 Tech Stack
* GIS Platform: QGIS 3.x
* Language: Python (PyQGIS framework)
* Configuration Format: QML (QGIS Layer Style)

## 🚀 How It Works (Under the Hood)

The core automation is built into custom QGIS Attribute Actions triggered via the **Run Feature Action** tool.

### 1. Dynamic Insertion Logic
When a feature is clicked using the action tool, it reads the file path from the `[%location%]` attribute field and evaluates the workspace:

```python
import os
from qgis.utils import iface
from qgis.core import QgsProject, QgsRasterLayer

root = QgsProject.instance().layerTreeRoot()
group_name = "PS group"
ps_group = None

# Locate or dynamically generate the destination group
for group in root.findGroups():
    if group.name() == group_name:
        ps_group = group
        break

if not ps_group:
    ps_group = root.addGroup(group_name)

path = r'[%location%]'
raster_layer = QgsRasterLayer(path, os.path.basename(path))

if not raster_layer.isValid():
    iface.messageBar().pushCritical('Error', f'Raster tile {os.path.basename(path)} could not be loaded')
else:
    QgsProject.instance().addMapLayer(raster_layer, False)
    ps_group.addLayer(raster_layer)
    iface.messageBar().pushSuccess('Success', f'Raster tile {os.path.basename(path)} loaded')

# Ensure user keeps focus on the index layer for uninterrupted workflows
layer_id = '[%@layer_id%]'
layer = QgsProject.instance().mapLayer(layer_id)
iface.setActiveLayer(layer)
