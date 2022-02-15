# Solar envelope

### Initialization
Importing the packages
``` python linenums="1"
#!pip install ipynb
import os
import topogenesis as tg
import pyvista as pv
import trimesh as tm
import numpy as np
from ladybug.sunpath import Sunpath
from scipy.interpolate import RegularGridInterpolator

import resources.functions as f


# convert mesh to pv_mesh
def tri_to_pv(tri_mesh):
    faces = np.pad(tri_mesh.faces, ((0, 0),(1,0)), 'constant', constant_values=3)
    pv_mesh = pv.PolyData(tri_mesh.vertices, faces)
    return pv_mesh
```

Importing meshes
``` python linenums="1"
envelope_path = os.path.relpath('../data/new_envelope.obj')
context_path = os.path.relpath('../data/immediate_context.obj')

# load the mesh from file
envelope_mesh = tm.load(envelope_path)
context_mesh = tm.load(context_path)

# Check if the mesh is watertight
print(envelope_mesh.is_watertight)
print(context_mesh.is_watertight)
```

Importing envelope lattice
``` python linenums="1"
# loading the lattice from csv
lattice_path = os.path.relpath('../data/envelope_lowres.csv')
envelope_lattice = tg.lattice_from_csv(lattice_path)
envelope_lattice_one = envelope_lattice * 0 + 1 
```

### Sun vectors
Importing envelope lattice
``` python linenums="1"
# initiate sunpath
sp = Sunpath(longitude=4.3571, latitude=52.0116)

# define sun hours : A list of hours of the year for each sun vector
# there are 8760 hours in a year, so the following integers refer to specific hours throughout the year
hoys = []
sun_vectors = []
day_multiples = 100
# for each day of the year ...
for d in range(365):
    # if it is one of the multiples
    if d%day_multiples==0:
        # for each hour of the day ...
        for h in range(24):
            # compute the hoy (hour of the year)
            hoy = d*24 + h
            # compute the sun object
            sun = sp.calculate_sun_from_hoy(hoy)
            # extract the sun vector (the direction that the sun ray travels toward)
            sun_vector = sun.sun_vector.to_array()
            # evidently, if the Z component of sun vector is positive, 
            # the sun is under the horizon 
            if sun_vector[2] < 0.0:
                hoys.append(hoy)
                sun_vectors.append(sun_vector)
                
sun_vectors = np.array(sun_vectors)
# compute the rotation matrix 
Rz = tm.transformations.rotation_matrix(np.radians(36.324), [0,0,1])
# Rotate the sun vectors to match the site rotation
sun_vectors = tm.transform_points(sun_vectors, Rz)
print(sun_vectors.shape)

# initiating the plotter
pv.set_plot_theme("document")
p = pv.Plotter(notebook=True)

# fast visualization of the lattice
envelope_lattice_one.fast_vis(p)

# adding the meshes
p.add_mesh(tri_to_pv(context_mesh), opacity=0.1, style='wireframe')

# add the sun locations, color orange
p.add_points( - sun_vectors * 300, color='#ffa500')

# plotting
#p.show(use_ipyvtk=True)
```

### Compute intersection of sun rays with context mesh
Preparing the list of ray directions and origins
``` python linenums="1"
# constructing the sun direction from the sun vectors in a numpy array
sun_dirs = -np.array(sun_vectors)
# exract the centroids of the envelope voxels
vox_cens = envelope_lattice_one.centroids
# next step we need to shoot in all of the sun directions from all of the voxels, todo so, we need repeat the sun direction for the number of voxels to construct the ray_dir (which is the list of all ray directions). We need to repeat the voxels for the 
ray_dir = []
ray_src = []
for v_cen in vox_cens:
    for s_dir in sun_dirs:
        ray_dir.append(s_dir)
        ray_src.append(v_cen)
# converting the list of directions and sources to numpy array
ray_dir = np.array(ray_dir)
ray_src = np.array(ray_src)

"""
# Further info: this is the vectorised version of nested for loops
ray_dir = np.tile(sun_dirs, [len(vox_cens),1])
ray_src = np.tile(vox_cens, [1, len(sun_dirs)]).reshape(-1, 3)
"""

print("number of voxels to shoot rays from :",vox_cens.shape)
print("number of rays per each voxel :",sun_dirs.shape)
print("number of rays to be shooted :",ray_src.shape)
```

Computing the intersection
``` python linenums="1"
tri_id, ray_id = context_mesh.ray.intersects_id(ray_origins=ray_src, ray_directions= ray_dir, multiple_hits=False)

# computing the intersections of rays with the context mesh
tri_invert_id, ray_invert_id = context_mesh.ray.intersects_id(ray_origins=ray_src, ray_directions= -ray_dir, multiple_hits=False)
```

### Aggregate simulation result in the sun access lattice
Computing the percentage of time that each voxel sees the sun
``` python linenums="1"
# initializing the hits list full of zeros
hits = [0]*len(ray_dir)
invert_hits = [0]*len(ray_dir)

# setting the rays that had an intersection to 1
for id in ray_id:
    hits[id] = 1

for id in ray_invert_id:
    invert_hits[id] = 1

sun_count = len(sun_dirs)
vox_count = len(vox_cens)
# initiating the list of ratio
vox_shadow = []
# iterate over the voxels
for v_id in range(vox_count):
    # counter for the intersection
    int_count = 0
    invert_int_count = 0
    # iterate over the sun rays
    for s_id in range(sun_count):
        # computing the ray id from voxel id and sun id
        r_id = v_id * sun_count + s_id

        # summing the intersections
        int_count += hits[r_id]

        if hits[r_id]==0:
         invert_int_count += invert_hits[r_id]
    
    # computing the percentage of the rays that DID NOT have 
    # an intersection (aka could see the sun)
    sun_access = int_count/sun_count
    shadowing =  invert_int_count/sun_count

    # add the ratio to list
  
    vox_shadow.append(shadowing)


hits = np.array(hits)
invert_hits = np.array(invert_hits)
vox_shadow = np.array(vox_shadow)
```

Store sun access information in a lattice
``` python linenums="1"
# getting the condition of all voxels: are they inside the envelop or not
env_all_vox = envelope_lattice_one.flatten()

# all voxels sun access
all_vox_shadow = []

# v_id: voxel id in the list of only interior voxels
v_id = 0

# for all the voxels, place the interiority condition of each voxel in "vox_in"
for vox_in in env_all_vox:
    # if the voxel was outside...
    if vox_in == True:
        # read its value of sun access and append it to the list of all voxel sun access
        all_vox_shadow.append(vox_shadow[v_id])
        # add one to the voxel id so the next time we read the next voxel
        v_id += 1
    # if the voxel was not inside... 
    else:
        # add 0.0 for its sun access
           all_vox_shadow.append(0.0)

# convert to array
shadow_array = np.array(all_vox_shadow)

# reshape to lattice shape
shadow_array = shadow_array.reshape(envelope_lattice_one.shape)

# convert to lattice
shadow_lattice = tg.to_lattice(shadow_array, envelope_lattice_one)
```

Visualize the sun access lattice
``` python linenums="1"
f.visualize(shadow_lattice, "Shadowing", "../data/shadowing_lowres.png")
```

### Interpolate lowres lattice to highres
Import highres lattice
``` python linenums="1"
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
highres_shadowing = interpolate(shadow_lattice)
highres_shadowing_lattice = highres_shadowing * highres_lattice
```

Save interpolated field to csv
``` python linenums="1"
# save the interpolated distance field to csv
csv_path = os.path.relpath("../data/shadowing.csv")
highres_shadowing_lattice.to_csv(csv_path)
```

Visualize highres field
``` python linenums="1"
f.visualize(highres_shadowing, "Shadowing", "../data/shadowing_highres.png")
```

### Generate configuration
Set conditions for the voxels
``` python linenums="1"
# the voxels with a higher shadow factor than 0.2 will be excluded
highres_shadow_minmax = (highres_shadowing_lattice <= 0.2) * (highres_shadowing_lattice != 0)
highres_shadow_minmax.sum()
```

Visualize the solar envelope
``` python linenums="1"
base_lattice = highres_shadow_minmax * highres_shadowing_lattice
f.visualize(base_lattice, "Shadowing", "../data/shadow_conf.png")
```

Save solar envelope into a csv
``` python linenums="1"
# save the solar envelope to csv
csv_path = os.path.relpath('../data/solar_envelope.csv')
base_lattice.to_csv(csv_path)
```

### Credits
``` python linenums="1"
__author__ = "Shervin Azadi and Pirouz Nourian"
__license__ = "MIT"
__version__ = "1.0"
__url__ = "https://github.com/shervinazadi/spatial_computing_workshops"
__summary__ = "Spatial Computing Design Studio Workshop on Solar Envelope"
```