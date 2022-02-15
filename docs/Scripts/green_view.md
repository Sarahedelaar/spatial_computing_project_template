# View on greenery

### Initalization
Importing the packages
``` python linenums="1"
import os
import topogenesis as tg
import pyvista as pv
import trimesh as tm
import numpy as np
import pickle
from ladybug.sunpath import Sunpath
from scipy.interpolate import RegularGridInterpolator
from pathlib import Path

import resources.functions as f

# convert trimesh to pv_mesh
def tri_to_pv(tri_mesh):
    faces = np.pad(tri_mesh.faces, ((0, 0),(1,0)), 'constant', constant_values=3)
    pv_mesh = pv.PolyData(tri_mesh.vertices, faces)
    return pv_mesh
```

Importing meshes
``` python linenums="1"
envelope_path = os.path.relpath("../data/new_envelope.obj")
context_path = os.path.relpath("../data/immediate_context.obj")

# load the mesh from file
envelope_mesh = tm.load(envelope_path)
context_mesh = tm.load(context_path)

# Check if the mesh is watertight
print(envelope_mesh.is_watertight)
```

Importing envelope lattice
``` python linenums="1"
# loading the lattice from csv
lattice_path = os.path.relpath("../data/envelope_lowres.csv")
envelope_lattice = tg.lattice_from_csv(lattice_path)

# creating the full lattice
full_lattice = envelope_lattice * 0 + 1
```
Importing greenery mesh and points
``` python linenums="1"
green_path = os.path.relpath("../data/green_agniesbuurt_rotatedpopgeo_nostumps.obj")
green_points_path = os.path.relpath("../data/green_points.csv")

# load the mesh from file
green_mesh = tm.load(green_path)
green_points = tg.cloud_from_csv(green_points_path)
```

### Visualize greenery
Visualize green meshes
``` python linenums="1"
pv.set_plot_theme("document")

# initiating the plotter
p = pv.Plotter(notebook=True)

# adding the meshes
p.add_mesh(tri_to_pv(envelope_mesh), color='#abd8ff')
p.add_mesh(tri_to_pv(context_mesh), opacity=0.1, style='wireframe')

#trying the green
p.add_mesh(tri_to_pv(green_mesh), color="#8bbe8f", style='wireframe')

# plotting
#p.show(use_ipyvtk=True)
```

Visualize green points
``` python linenums="1"
# initiating the plotter
p = pv.Plotter(notebook=True)
pv.set_plot_theme("document")

# fast visualization of the lattice
full_lattice.fast_vis(p)

# adding the context mesh
p.add_mesh(tri_to_pv(context_mesh), opacity=0.1, style='wireframe')

# add the green points
p.add_mesh(green_points, color="#8bbe8f", show_edges=False, lighting = False)

#p.show(use_ipyvtk=True)
```

### Compute rays from green points to envelope
Preparing the list of ray directions and origins
``` python linenums="1"
# Creating list of directions and rays

converted_green_vecs = green_points.astype('float16')
green_dirs2 = np.array(converted_green_vecs)
vox_cens = full_lattice.centroids

ray_dir3 = []
ray_src3 = []

for v_cen in vox_cens:
    for gp in converted_green_vecs:
        ray_src3.append(v_cen)
        ray_dir3.append(gp - v_cen)
            
# converting the list of directions and sources to numpy array
ray_dir3 = np.array(ray_dir3)
ray_src3 = np.array(ray_src3)

# checking how many calculations to do
print("number of voxels to shoot rays from :",vox_cens.shape)
print("number of rays per each voxel :",green_dirs2.shape)
print("number of rays to be shot :",ray_src3.shape)
```

Import the pickle files with calculations
``` python linenums="1"
# the calculation is too big for a normal computer
# the calculation has been done on a better computer and imported as pickle files
bs = 1e5
d = '../data/sims'
ray_id3 = []
int_loc3 = []
for i in range(int(len(ray_src3) / bs) + 1):
    # s = int(i * bs)
    # e = int((i + 1) * bs)
    # e = e if e < len(ray_src3) else len(ray_src3) - 1
    with open(f'{d}/green_{i}.pickle', 'rb') as file:
        _,r,l = pickle.load(file)
        ray_id3.append(r + int(i * bs))
        int_loc3.append(l)
ray_id3 = np.concatenate(ray_id3)
int_loc3 = np.concatenate(int_loc3)
```

Reshape the information
``` python linenums="1"
# initializing the hits list full of zeros
hits3 = [0]*len(ray_dir3)
hit_loc3 = [[0,0,0]] * len(ray_dir3)

for i, id in enumerate(ray_id3):
    hits3[id] = 1 
    hit_loc3[id] = int_loc3[i]
        
gp_count = len(green_dirs2)
vox_count = len(vox_cens)

# initiating the list of ratio
vox_grn_vis = []

# iterate over the voxels
for v_id in range(vox_count):
    # counter for the intersection
    int_count = 0
    # iterate over the sun rays
    for s_id in range(gp_count):
        # computing the ray id from voxel id and sun id
        r_id = s_id + v_id * gp_count

        if hits3[r_id]:
            # retrieve the ray origin
            ray_orig = ray_src3[r_id]
            # retrieve the intersection point
            ray_dest = hit_loc3[r_id]
            # retrieve the ray vector
            ray_vec = ray_dir3[r_id]
            # the distance from origin to intersection point
            int_length = np.sum((ray_dest - ray_orig)**2)**0.5
            # the distance from the origin to the green point
            ray_length = np.sum(ray_vec**2)**0.5
            # if the ray length is greater than the intersection length, it means that the view toward the green point is blocked
            if ray_length > int_length:
                # summing the intersections
                int_count += 1

    # computing the percentage of the rays that DID NOT have 
    # an intersection (aka could see the skydome)
    grn_vis = 1.0 - int_count / gp_count

    # add the ratio to list
    vox_grn_vis.append(grn_vis)
    
# converting the list of directions and sources to numpy array
vox_grn_vis = np.array(vox_grn_vis)
```

Store green view information in a lattice
``` python linenums="1"
# getting the condition of all voxels: are they inside the envelop or not
env_all_vox = full_lattice.flatten()

# all voxels green view
all_vox_grn_vis = []

# v_id: voxel id in the list of only interior voxels
v_id = 0

# for all the voxels, place the interiority condition of each voxel in "vox_in"
for vox_in in env_all_vox:
    # if the voxel was outside...
    if vox_in == True:
        # read its value of green view and append it to the list of all voxel green view
        all_vox_grn_vis.append(vox_grn_vis[v_id])
        # add one to the voxel id so the next time we read the next voxel
        v_id += 1
    # if the voxel was not inside... 
    else:
        # add 0.0
        all_vox_grn_vis.append(0.0)

        
# convert to array
grn_vis_array = np.array(all_vox_grn_vis)

# Further info: for vectorized version of this code check: https://github.com/shervinazadi/spatial_computing_workshops/blob/master/notebooks/w2_solar_envelope.ipynb

# reshape to lattice shape
grn_vis_array = grn_vis_array.reshape(full_lattice.shape)

# convert to lattice
grn_vis_lattice = tg.to_lattice(grn_vis_array, full_lattice)
```

Visualize the view on greenery field
``` python linenums="1"
f.visualize(grn_vis_lattice, "Green visibility", "../data/green_view_lowres")
```

Save lowres field to csv
``` python linenums="1"
# save the SVF latice to csv
csv_path = os.path.relpath('../data/green_view_lowres.csv')
grn_vis_lattice.to_csv(csv_path)
```

### Interpolate lowres lattices to highres
Import highres lattice
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

Interpolate greenery lattice
``` python linenums="1"
highres_grn = interpolate(grn_vis_lattice)
highres_grn_lattice = highres_grn * highres_lattice
```

Visualize highres green view field
``` python linenums="1"
f.visualize(highres_grn, "Green visibility", "../data/green_view_highres.png")
```

Save highres field into a csv
``` python linenums="1"
# save the SVF latice to csv
csv_path = os.path.relpath('../data/green_view.csv')
highres_grn_lattice.to_csv(csv_path)
```

### Credits
``` python linenums="1"
__author__ = "Shervin Azadi and Pirouz Nourian"
__license__ = "MIT"
__version__ = "1.0"
__url__ = "https://github.com/shervinazadi/spatial_computing_workshops"
__summary__ = "Spatial Computing Design Studio Workshop on Solar Envelope"
```