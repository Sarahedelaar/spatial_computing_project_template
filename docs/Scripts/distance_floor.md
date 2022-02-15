# Distance floor
Import packages
``` python linenums="1"
import os
import topogenesis as tg
import pyvista as pv
import trimesh as tm
import numpy as np
import networkx as nx
from scipy.interpolate import RegularGridInterpolator
import copy

import resources.functions as f
```

Import lattice
``` python linenums="1"
lattice_path = os.path.relpath("../data/voxelized_envelope_lowres.csv")
envelope_lattice = tg.lattice_from_csv(lattice_path)
full_lattice = envelope_lattice * 0 + 1
```

### Create distance lattice
Creat vertical adjacency matrix
``` python linenums="1"
# get the amount of voxels in the z direction
vox_count = len(full_lattice[0][0])

# initialize an adjacency matrix
adj_mtrx = np.zeros((vox_count,vox_count))

# set the connecting values of the adjacency matrix to 1
for i in range(vox_count):
    if i == 0:
        adj_mtrx[i, i + 1] = 1
        continue
    if i == vox_count - 1:
        adj_mtrx[i, i - 1] = 1
        continue
    else:
        adj_mtrx[i, i + 1] = 1
        adj_mtrx[i, i - 1] = 1
```

Turning into networkx datastructure and calculating distances
``` python linenums="1"
g = nx.from_numpy_array(adj_mtrx)
dist_mtrx_vertical = nx.floyd_warshall_numpy(g)
```

Selecting floor level and mapping distances between 0 and 1
``` python linenums="1"
floor_dist = dist_mtrx_vertical[0-4] # specified floor level

max_valid = np.ma.masked_invalid(floor_dist).max() # find max distance

floor_dist_mapped = 1 - (floor_dist / max_valid) # map values between 0 and 1 with max distance
```

Mapping the values to the full latice
``` python linenums="1"
floor_dist_mapped_full = []
height = len(floor_dist_mapped)

# mapping the colomn of values to the full lattice
for i in range(len(full_lattice)*len(full_lattice[0])*len(full_lattice[0][0])):
    floor_dist_mapped_full.append(floor_dist_mapped[i % height])

# turning it into an np array
floor_dist_mapped_full = np.array(floor_dist_mapped_full)

# reshaping the array
floor_dist_lattice_full = floor_dist_mapped_full.reshape(full_lattice.shape)

# forming the lattice to the solar envelope
floor_dist_lattice = floor_dist_lattice_full * full_lattice 
```

Visualize the floor level lattice
``` python linenums="1"
base_lattice = floor_dist_lattice * envelope_lattice
f.visualize(base_lattice, "Ground floor closeness", "../data/ground_dist_lowres.png")
```

Export to csv
``` python linenums="1"
floor_dist = floor_dist_lattice * envelope_lattice

# save the lowres distance field to csv
csv_path = os.path.relpath('../data/ground_field_lowres.csv')
floor_dist.to_csv(csv_path)
```

### Interpolate lowres to highres
Define interpolation function
``` python linenums="1"
def interpolate(lowres_field):
    # loading highres lattice
    highres_lattice_path = os.path.relpath('../data/envelope_highres.csv')
    highres_lattice = tg.lattice_from_csv(highres_lattice_path)
    
    # line spaces
    x_space = np.linspace(lowres_field.minbound[0], lowres_field.maxbound[0],lowres_field.shape[0])
    y_space = np.linspace(lowres_field.minbound[1], lowres_field.maxbound[1],lowres_field.shape[1])
    z_space = np.linspace(lowres_field.minbound[2], lowres_field.maxbound[2],lowres_field.shape[2])

    # interpolation function
    interpolating_function = RegularGridInterpolator((x_space, y_space, z_space), lowres_field, bounds_error=False, fill_value=None)

    # high_res lattice
    full_lattice = highres_lattice + 1

    # sample point
    sample_points = full_lattice.centroids

    # interpolation
    interpolated_values = interpolating_function(sample_points)

    # lattice construction
    interpolated_lattice = tg.to_lattice(interpolated_values.reshape(highres_lattice.shape), highres_lattice)

    # nulling the unavailable cells
    interpolated_lattice *= highres_lattice

    return interpolated_lattice
```

Interpolate closeness lattice
``` python linenums="1"
# Import the highres envelope
highres_lattice_path = os.path.relpath('../data/envelope_highres.csv')
highres_lattice = tg.lattice_from_csv(highres_lattice_path)

# Interpolate from lowres to highres
highres_ground_lattice = interpolate(floor_dist_lattice)
# multiply by original highres_lattice to filter out the voxels outside the envelope
highres_ground = highres_ground_lattice * highres_lattice
```

Save interpolated field to csv
``` python linenums="1"
# save the interpolated distance field to csv
csv_path = os.path.relpath("../data/ground.csv")
highres_ground.to_csv(csv_path)
```

Visualize highres field
``` python linenums="1"
f.visualize(highres_ground, "Ground floor closeness", "../data/ground_acc_highres.png")
```

### Credits
``` python linenums="1"
__author__ = "Shervin Azadi"
__license__ = "MIT"
__version__ = "1.0"
__url__ = "https://github.com/shervinazadi/earthy_workshops"
__summary__ = "Earthy Design Studio"

Made with floor closeness notebook from group CUB3D
```