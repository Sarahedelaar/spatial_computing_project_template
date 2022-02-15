# Voxelization
### Initialization
Visualize highres field
``` python linenums="1"
#!conda env update -f ../environment.yml

import os
import topogenesis as tg
import trimesh as tm
import numpy as np
import pyvista as pv

# convert trimesh to pv_mesh
def tri_to_pv(tri_mesh):
    faces = np.pad(tri_mesh.faces, ((0, 0),(1,0)), 'constant', constant_values=3)
    pv_mesh = pv.PolyData(tri_mesh.vertices, faces)
    return pv_mesh

# set the size of the voxels
vs = 3.6
unit = [vs, vs, vs]
mesh_path = os.path.relpath('../data/new_envelope.obj')
```

### Input mesh
``` python linenums="1"
# load the mesh from file
mesh = tm.load(mesh_path)
# Check if the mesh is watertight
print(mesh.is_watertight)

# loading the context mesh
context_path = os.path.relpath('../data/immediate_context.obj')
context_mesh = tm.load(context_path)

# initiating the plotter
p = pv.Plotter()
pv.set_plot_theme("document")

# adding the base mesh: light blue
p.add_mesh(tri_to_pv(mesh), color='#abd8ff')
# adding the meshes
p.add_mesh(tri_to_pv(context_mesh), opacity=0.1, style='wireframe')

# plotting
#p.show(screenshot="plot.png")
```

### Voxelize the mesh
``` python linenums="1"
# initialize the base lattice
base_lattice = tg.lattice(mesh.bounds, unit=unit, default_value=1, dtype=int)

# check which voxel centroids is inside the mesh
interior_condition = mesh.contains(base_lattice.centroids)

# reshape the interior condition to the shape of the base_lattice
interior_array = interior_condition.reshape(base_lattice.shape)

# convert the interior array into a lattice
interior_lattice = tg.to_lattice(interior_array, base_lattice.minbound, base_lattice.unit)

interior_lattice.shape

# initiating the plotter
p = pv.Plotter()
pv.set_plot_theme("document")

# fast visualization of the lattice
interior_lattice.fast_vis(p)

# adding the base mesh: light blue
p.add_mesh(tri_to_pv(mesh), color='#abd8ff', opacity=0.1)

# adding the meshes
p.add_mesh(tri_to_pv(context_mesh), opacity=0.1, style='wireframe')

# plotting
# p.show(screenshot="highres_lattice.png")
```

### Saving the lattice to csv
``` python linenums="1"
csv_path = os.path.relpath('../data/envelope_highres.csv')
interior_lattice.to_csv(csv_path)
```

### Credits
``` python linenums="1"
__author__ = "Shervin Azadi"
__license__ = "MIT"
__version__ = "1.0"
__url__ = "https://github.com/shervinazadi/spatial_computing_workshops"
__summary__ = "Spatial Computing Design Studio Workshop on Voxelization"
```