# Noise

### Initialization
Load required libraries
``` python linenums="1"
import os
import topogenesis as tg
import pyvista as pv
import trimesh as tm
import numpy as np
import scipy as sp
from scipy.interpolate import RegularGridInterpolator

import resources.functions as f

# convert mesh to pv_mesh
def tri_to_pv(tri_mesh):
    faces = np.pad(tri_mesh.faces, ((0, 0),(1,0)), 'constant', constant_values=3)
    pv_mesh = pv.PolyData(tri_mesh.vertices, faces)
    return pv_mesh
```

Load envelope lattice as the availability lattice
``` python linenums="1"
# loading the lattice from csv
lattice_path = os.path.relpath('../data/envelope_lowres.csv')
avail_lattice = tg.lattice_from_csv(lattice_path)
init_avail_lattice = tg.to_lattice(np.copy(avail_lattice), avail_lattice)
init_avail_lattice_one = init_avail_lattice * 0 + 1

# loading the context mesh
context_path = os.path.relpath('../data/immediate_context.obj')
context_mesh = tm.load(context_path)
```

Load noise sources
``` python linenums="1"
# loading noise source points from CSV
noise_source_path = os.path.relpath('../data/streetpoints.xyz.txt')
noise_sources = np.genfromtxt(noise_source_path, delimiter=',')
```

Visualize noise source points
``` python linenums="1"
pv.set_plot_theme("Document")
p = pv.Plotter(notebook=True)

# adding the avilability lattice
init_avail_lattice.fast_vis(p)

# adding axes
p.add_axes()

# add the meshes
p.add_mesh(noise_sources, point_size=8)
p.add_mesh(tri_to_pv(context_mesh), opacity=0.1, style='wireframe')

#p.show(use_ipyvtk=True)
```

### Creation of noise field
Load noise sources
``` python linenums="1"
# loading noise source points from CSV
noise_source_path = os.path.relpath('../data/streetpoints.xyz.txt')
noise_sources = np.genfromtxt(noise_source_path, delimiter=',')
```

Visualize the noise lattices
``` python linenums="1"
f.visualize(agg_noise_lat, "Noise", "../data/noise_lowres.png")
```

### Interpolate lowres lattice to highres
Visualize the noise lattices
``` python linenums="1"
# loading highres lattice
highres_lattice_path = os.path.relpath('../data/envelope_highres.csv')
highres_lattice = tg.lattice_from_csv(highres_lattice_path)
```

Define interpolation function
``` python linenums="1"
def interpolate(lowres_field):
    # loading highres lattice
    highres_lattice_path = os.path.relpath('../data/envelope_highres.csv')
    highres_lat = tg.lattice_from_csv(highres_lattice_path)
    highres_lattice = highres_lat * 0 + 1
    
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
highres_noise = interpolate(agg_noise_lat)
highres_noise_field = highres_noise * highres_lattice
```

Save interpolated field to csv
``` python linenums="1"
# save the interpolated distance field to csv
csv_path = os.path.relpath("../data/noise.csv")
highres_noise_field.to_csv(csv_path)
```

Visualize highres field
``` python linenums="1"
f.visualize(highres_noise, "Noise", "../data/noise_highres.png")
```

### Credits
Visualize highres field
``` python linenums="1"
__author__ = "Shervin Azadi"
__license__ = "MIT"
__version__ = "1.0"
__url__ = "https://github.com/shervinazadi/spatial_computing_workshops"
__summary__ = "Spatial Computing Design Studio Workshop on Noise Fields"
```