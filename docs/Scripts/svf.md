# Sky view factor
Import
``` python linenums="1"
import os
import topogenesis as tg
import pyvista as pv
import trimesh as tm
import numpy as np
import pickle
from scipy.interpolate import RegularGridInterpolator

import resources.functions as f

# convert mesh to pv_mesh
def tri_to_pv(tri_mesh):
    faces = np.pad(tri_mesh.faces, ((0, 0),(1,0)), 'constant', constant_values=3)
    pv_mesh = pv.PolyData(tri_mesh.vertices, faces)
    return pv_mesh
```

Import meshes
``` python linenums="1"
envelope_path = os.path.relpath("../data/new_envelope.obj")
context_path = os.path.relpath("../data/immediate_context.obj")

# load the mesh from file
envelope_mesh = tm.load(envelope_path)
context_mesh = tm.load(context_path)

# Check if the mesh is watertight
print(envelope_mesh.is_watertight)
```

Importing the envelope lattice
``` python linenums="1"
# loading the lattice from csv
lattice_path = os.path.relpath("../data/envelope_lowres.csv")
envelope_lattice = tg.lattice_from_csv(lattice_path)

# creating the full lattice
full_lattice = envelope_lattice * 0 + 1
```

Importing the skydome
``` python linenums="1"
# loading the skydome
skydome_path = os.path.relpath("../data/skydome.obj")
skydome_mesh = tm.load(skydome_path)
sky_sphere = tm.primitives.Sphere(radius=1, center=[0,0,0], subdivisions=1)
```

### Sun vectors
Computing SVF vectors with skydome
``` python linenums="1"
#adding normals of the dome to a numpy list
svf_vectors = np.array(sky_sphere.face_normals)
svf_vectors = svf_vectors[svf_vectors[:,2] > 0]
#checking how many rays to calculate
print(len(svf_vectors))
```

Visualize the skydome
``` python linenums="1"
# initiating the plotter
p = pv.Plotter(notebook=True)

# fast visualization of the lattice
full_lattice.fast_vis(p)

# adding the meshes
p.add_mesh(tri_to_pv(context_mesh), opacity=0.1, style='wireframe')

# add the sun locations, color orange
for loc in (svf_vectors * 250):
    p.add_mesh(pv.Sphere(radius= 2, center = loc), color="#ffa500", show_edges=False, lighting = False)

# plotting
#p.show(use_ipyvtk=True)
```

### Compute intersection of sun ray with context mesh
Computing SVF vectors with skydome
``` python linenums="1"
# Creating list of directions and rays for the SVF

converted_svf_vectors = svf_vectors.astype('float16')
sun_dirs2 = np.array(converted_svf_vectors)
vox_cens = full_lattice.centroids

ray_dir3 = []
ray_src3 = []

for v_cen in vox_cens:
        for s_dir in sun_dirs2:
            ray_src3.append(v_cen)
            ray_dir3.append(s_dir)
            
# converting the list of directions and sources to numpy array
ray_dir3 = np.array(ray_dir3)
ray_src3 = np.array(ray_src3)

# checking how many calculations to do for SVF
print("number of voxels to shoot rays from :",vox_cens.shape)
print("number of rays per each voxel :",sun_dirs2.shape)
print("number of rays to be shot :",ray_src3.shape)
```

Computing intersection
``` python linenums="1"
# computing the intersections of rays with the context mesh for the SVF
tri_id3, ray_id3 = context_mesh.ray.intersects_id(ray_origins=ray_src3, ray_directions=ray_dir3, multiple_hits=False)
```

### Aggregate simulation result in the sun access lattice
Compute percentage of time that each voxel sees sun
``` python linenums="1"
# Same calculations, but for the SVF
# initializing the hits list full of zeros

hits4 = [0]*len(ray_dir3)

for id in ray_id3:
    hits4[id] = 1 
        
sun_count = len(sun_dirs2)
vox_count = len(vox_cens)

# initiating the list of ratio
vox_sky_acc = []

# iterate over the voxels
for v_id in range(vox_count):
    # counter for the intersection
    int_count = 0
    # iterate over the sun rays
    for s_id in range(sun_count):
        # computing the ray id from voxel id and sun id
        r_id = s_id + v_id * sun_count

        # summing the intersections
        int_count += hits4[r_id]

    # computing the percentage of the rays that DID NOT have 
    # an intersection (aka could see the skydome)
    sky_access = 1.0 - int_count / sun_count

    # add the ratio to list
    vox_sky_acc.append(sky_access)
    
# converting the list of directions and sources to numpy array
vox_sky_acc = np.array(vox_sky_acc)
```

Store sun acces information in a lattice
``` python linenums="1"
# Same calculation, but for SVF
# getting the condition of all voxels: are they inside the envelop or not
env_all_vox = full_lattice.flatten()

# all voxels sky access
all_vox_sky_acc = []

# v_id: voxel id in the list of only interior voxels
v_id = 0

# for all the voxels, place the interiority condition of each voxel in "vox_in"
for vox_in in env_all_vox:
    # if the voxel was outside...
    if vox_in == True:
        # read its value of sky acces and append it to the list of all voxel sky access
        all_vox_sky_acc.append(vox_sky_acc[v_id])
        # add one to the voxel id so the next time we read the next voxel
        v_id += 1
    # if the voxel was not inside... 
    else:
        # add 0.0 for its sun access
        all_vox_sky_acc.append(0.0)

        
# convert to array
sky_acc_array = np.array(all_vox_sky_acc)

# Further info: for vectorized version of this code check: https://github.com/shervinazadi/spatial_computing_workshops/blob/master/notebooks/w2_solar_envelope.ipynb

# reshape to lattice shape
sky_acc_array = sky_acc_array.reshape(full_lattice.shape)

# convert to lattice
sky_acc_lattice = tg.to_lattice(sky_acc_array, full_lattice)
```

Computing intersection
``` python linenums="1"
f.visualize(sky_acc_lattice, "Sky view", "../data/sky_view_lowres.png")
```

Write sky view field to csv
``` python linenums="1"
# save the SVF latice to csv
csv_path = os.path.relpath('../data/sky_view_lowres.csv')
sky_acc_lattice.to_csv(csv_path)
```

### Interpolate lowres lattices to highres
Computing intersection
``` python linenums="1"
# loading the lattice from csv
lattice_path = os.path.relpath('../data/envelope_highres.csv')
highres_lattice = tg.lattice_from_csv(lattice_path)
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
sky_acc_highres = interpolate(sky_acc_lattice)

# multiply with the original lattice to exclude the voxels outside the envelope
sky_acc_highres_lattice = sky_acc_highres * highres_lattice
```

Save interpolated field to csv
``` python linenums="1"
# save the SVF latice to csv
csv_path = os.path.relpath('../data/sky_view.csv')
sky_acc_highres_lattice.to_csv(csv_path)
```

Visualize highres field
``` python linenums="1"
f.visualize(sky_acc_highres, "Sky view", "../data/sky_view_highres.png")
```

### Credits
``` python linenums="1"
__author__ = "Shervin Azadi and Pirouz Nourian"
__license__ = "MIT"
__version__ = "1.0"
__url__ = "https://github.com/shervinazadi/spatial_computing_workshops"
__summary__ = "Spatial Computing Design Studio Workshop on Solar Envelope"
```