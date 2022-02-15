# Distance entrance
Import packages
``` python linenums="1"
import os
import topogenesis as tg
import pyvista as pv
import trimesh as tm
import numpy as np
import networkx as nx
from scipy.interpolate import RegularGridInterpolator
import resources.functions as f
```

Import meshes
``` python linenums="1"
envelope_path = os.path.relpath('../data/new_envelope.obj')
context_path = os.path.relpath('../data/immediate_context.obj')

# load the mesh from file
envelope_mesh = tm.load(envelope_path)
context_mesh = tm.load(context_path)
```

Import envelope lattice
``` python linenums="1"
# loading the lowres lattice from csv
field_path = os.path.relpath("../data/envelope_lowres.csv")
envelope_lattice = tg.lattice_from_csv(field_path)

init_avail_lattice = tg.to_lattice(np.copy(envelope_lattice), envelope_lattice)
# create a full lattice full with ones
full_lattice = envelope_lattice * 0 + 1

# loading the highres lattice from csv
field_path = os.path.relpath("../data/envelope_highres.csv")
highres_lattice = tg.lattice_from_csv(field_path)
```

Import street points + public transport points
``` python linenums="1"
street_transport_path = os.path.relpath('../data/mainstreet_publictransport.xyz.txt')
street_transport = np.genfromtxt(street_transport_path, delimiter=',')
```

### Euclidean distance lattice
Distance matrix
``` python linenums="1"
# extracting the centroid of all voxels
env_cens = envelope_lattice.centroids_threshold(-1)

# initializing the distance matrix
dist_m = []
# for each voxel ...
for voxel_cen in env_cens:
    # initializing the distance vector (per each voxel)
    dist_v = []
    # for each street point ...
    for street_transport_point in street_transport:
        # find the difference vector
        diff = voxel_cen - street_transport_point
        # raise the components to the power of two
        diff_p2 = diff**2
        # sum the components
        diff_p2s = diff_p2.sum()
        # compute the square root 
        dist = diff_p2s**0.5
        # add the distance to the distance vector
        dist_v.append(dist)
    # add the distance vector to the distance matrix
    dist_m.append(dist_v)
# change the distance matrix type, from list to array
dist_m = np.array(dist_m)
```

### Selecting entrance voxel
Finding min distance to street and public
``` python linenums="1"
# average of the distances to each point per voxel
av_dist = []
for str_trans_points in dist_m:
    # calculate av_dist by taking the sum of the distances and divide by the amount of street points
    av_dist_vox = str_trans_points.sum()/len(street_transport)
    av_dist.append(av_dist_vox)
    
# Find the voxel with the minimum average distance 
min_dist = np.min(av_dist)

# Find the index of this voxel
min_index = np.argmin(av_dist)
# get the 3d location
min_index_3d = np.unravel_index(min_index, envelope_lattice.shape)
```

### Import the stencils
Stencil horizontal
``` python linenums="1"
# creating neighborhood definition 1 (horizontal)
stencil_1 = tg.create_stencil("von_neumann", 0, 1)
# setting the indices
stencil_1.set_index([0,0,0], 0)
stencil_1.set_index([0,-1,0], 1)
stencil_1.set_index([0,1,0], 1)
stencil_1.set_index([-1,0,0], 1)
stencil_1.set_index([1,0,0], 1)
```

Stencil vertical
``` python linenums="1"
# creating neighborhood definition 2 (vertical)
stencil_2 = tg.create_stencil("von_neumann", 0, 1)
# setting the indices
stencil_2.set_index([0,0,0], 0)
stencil_2.set_index([0,0,-1], 1)
stencil_2.set_index([0,0,1], 1)
```

### Make envelope padded
``` python linenums="1"
# make envelope padded to prevent seeing one side as a neighbour of the other side
env_padded_array = np.pad(envelope_lattice, 1)
padded_minbound = envelope_lattice.minbound - envelope_lattice.unit
env_padded_lattice = tg.to_lattice(env_padded_array, minbound=padded_minbound, unit=envelope_lattice.unit)

# convert the location of the main entrance to the right envelope
str_con_lattice = env_padded_lattice * False
ent_sel_3d_ind_padded = tuple(np.array(np.unravel_index(min_index, envelope_lattice.shape)) + 1)
str_con_lattice[ent_sel_3d_ind_padded] = True
```

### Distance lattice
Retrieve neighbours
``` python linenums="1"
# retrieve the neighbour list of each cell
neighs_1 = str_con_lattice.find_neighbours(stencil_1)
neighs_2 = str_con_lattice.find_neighbours(stencil_2)

# set the maximum distance to sum of the size of the lattice in all dimensions.
max_dist = np.sum(str_con_lattice.shape)

# initialize the street network distance lattice with all the street cells as 0, and all other cells as maximum distance possible
mn_dist_lattice = 1 - str_con_lattice
mn_dist_lattice[mn_dist_lattice==1] = max_dist

# flatten the distance lattice for easy access
mn_dist_lattice_flat = mn_dist_lattice.flatten()

# flatten the envelope lattice
env_pad_lat_flat = env_padded_lattice.flatten()
```

Giving every voxel values
``` python linenums="1"
# main loop for breath-first traversal
for i in range(1, max_dist):
    # find the neighbours of the previous step
    next_step_1 = neighs_1[mn_dist_lattice_flat == i - 1]
    # find the unique neighbours
    next_unq_step_1 = np.unique(next_step_1.flatten())
    # check if the neighbours of the next step are inside the envelope
    validity_condition_1 = env_pad_lat_flat[next_unq_step_1]
    # select the valid neighbours
    next_valid_step_1 = next_unq_step_1[validity_condition_1]

    # find the neighbours of the previous step
    next_step_2 = neighs_2[mn_dist_lattice_flat == i - 1]
    # find the unique neighbours
    next_unq_step_2 = np.unique(next_step_2.flatten())
    # check if the neighbours of the next step are inside the envelope
    validity_condition_2 = env_pad_lat_flat[next_unq_step_2]
    # select the valid neighbours
    next_valid_step_2 = next_unq_step_2[validity_condition_2]

    # make a copy of the lattice to prevent overwriting in the memory
    mn_nex_dist_lattice_flat = np.copy(mn_dist_lattice_flat)

    # set the next step cells to the current distance
    mn_nex_dist_lattice_flat[next_valid_step_2] = i + 1
    # set the next step cells to the current distance
    mn_nex_dist_lattice_flat[next_valid_step_1] = i

    # find the minimum of the current distance and previous distances to avoid overwriting previous steps
    mn_dist_lattice_flat = np.minimum(mn_dist_lattice_flat, mn_nex_dist_lattice_flat)
    
    # check how many of the cells have not been traversed yet
    filled_check = mn_dist_lattice_flat * env_pad_lat_flat == max_dist
    # if all the cells have been traversed, break the loop
    if filled_check.sum() == 0:
        print(i)
        break

# reshape and construct a lattice from the street network distance list
mn_dist_lattice = mn_dist_lattice_flat.reshape(mn_dist_lattice.shape)

mn_dist_lattice = mn_dist_lattice.astype(float)
mn_dist_lattice *= env_padded_lattice 

# reverse the lattice values (1 = close, 0 = far)
mn_dist_lattice = env_padded_lattice - (mn_dist_lattice - mn_dist_lattice.min()) / mn_dist_lattice.max()
```

### Visualize distance lattice
``` python linenums="1"
f.visualize(mn_dist_lattice, "Entrance distance", "../data/distance_field.png")
```

Write distance lattice to csv
``` python linenums="1"
# write distance lattice to csv
csv_path = os.path.relpath("../data/field_distance_entrance.csv")
mn_dist_lattice.to_csv(csv_path)
```

### Interpolate lowres lattice to highres
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
# Interpolate from lowres to highres
highres_entrance = interpolate(mn_dist_lattice)
highres_entrance_lattice = highres_entrance * highres_lattice
```

Save interpolated field to csv
``` python linenums="1"
# save the interpolated distance field to csv
csv_path = os.path.relpath("../data/ent_acc.csv")
highres_entrance_lattice.to_csv(csv_path)
```

Visualize highres field
``` python linenums="1"
f.visualize(highres_entrance, "Entrance distance", "../data/ent_acc_highres.png")
```

# Generate configuration
Set conditions for the voxels
``` python linenums="1"
# this is an example configuration of looking only at the distance to the entrance
highres_entrance_minmax = (highres_entrance_lattice > 0.5) * (highres_entrance_lattice != 0)
highres_entrance_minmax.sum()
```
Visualize the conditioned voxels
``` python linenums="1"
base_lattice = highres_entrance_minmax * highres_entrance_lattice

f.visualize(base_lattice, "Entrance distance", "distance_field_conf.png")
```
Save configuration to csv
``` python linenums="1"
# save the configuration to csv
csv_path = os.path.relpath("../data/configuration_distance_entrance.csv")
highres_entrance_minmax.to_csv(csv_path)
```
# Credits
``` python linenums="1"
__author__ = "Shervin Azadi"
__license__ = "MIT"
__version__ = "1.0"
__url__ = "https://github.com/shervinazadi/earthy_workshops"
__summary__ = "Earthy Design Studio"
```