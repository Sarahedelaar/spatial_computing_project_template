# Generative relations: corridor generation
### Initialization
Load required libraries
``` python linenums="1"
# !pip install scikit-learn

import os
import topogenesis as tg
import pyvista as pv
import numpy as np
import networkx as nx
import pandas as pd
import trimesh as tm
from sklearn.cluster import KMeans
np.random.seed(0)
```

Define the neighbourhood (stencil)
``` python linenums="1"
# creating neighbourhood definition
stencil = tg.create_stencil("von_neumann", 1, 1)
# setting the center to zero
stencil.set_index([0,0,0], 0)
stencil.set_index([0,0,1], 0)
stencil.set_index([0,0,-1], 0)
print(stencil)

# creating neighbourhood definition
v_stencil = tg.create_stencil("von_neumann", 1, 1)
# setting the center to zero
v_stencil.set_index([0,0,0], 0)
v_stencil.set_index([0,-1,0], 0)
v_stencil.set_index([0,1,0], 0)
v_stencil.set_index([-1,0,0], 0)
v_stencil.set_index([1,0,0], 0)
print(v_stencil)
```

Load the envelope lattice as the availability lattice
``` python linenums="1"
# loading the lattice from csv
lattice_path = os.path.relpath('../data/envelope_highres.csv')
avail_lattice = tg.lattice_from_csv(lattice_path)
init_avail_lattice = tg.to_lattice(np.copy(avail_lattice), avail_lattice)
```

Load agents information
``` python linenums="1"
# loading program (agents information) from CSV
program_complete = pd.read_csv("../data/program_req.csv")
# program_complete
```

### Creation of vertical shaft
Agent initialization
``` python linenums="1"
# Finding the index of the available voxels in avail_lattice
avail_flat = avail_lattice.flatten()
avail_index = np.array(np.where(avail_lattice == 1)).T

field_path = os.path.relpath("../data/cluster_center_lattice.csv")
occ_lattice = tg.lattice_from_csv(field_path)

agn_num = len(program_complete)

select_id = np.where(occ_lattice.flatten() != -1)
program_complete[["OX", "OY", "OZ"]] = avail_index[select_id]

# adding the origins to the agents locations
agn_locs = []
# for each agent origin ... 
for a_id, a_info in program_complete.iterrows():

    a_origin = a_info[["OX", "OY", "OZ"]].to_list()

    # add the origin to the list of agent locations
    agn_locs.append(a_origin)

    # set the origin in availability lattice as 0 (UNavailable)
    avail_lattice[tuple(a_origin)] = 0
```

Visualizing the agent seeds
``` python linenums="1"
p = pv.Plotter(notebook=True)

base_lattice = occ_lattice

# Set the grid dimensions: shape + 1 because we want to inject our values on the CELL data
grid = pv.UniformGrid()
grid.dimensions = np.array(base_lattice.shape) + 1
# The bottom left corner of the data set
grid.origin = base_lattice.minbound - base_lattice.unit * 0.5
# These are the cell sizes along each axis
grid.spacing = base_lattice.unit 

# adding the boundingbox wireframe
p.add_mesh(grid.outline(), color="grey", label="Domain")

# adding axes
p.add_axes()
p.show_bounds(grid="back", location="back", color="#aaaaaa")

# Add the data values to the cell data
grid.cell_arrays["Agents"] = base_lattice.flatten(order="F").astype(int)  # Flatten the array!
# filtering the voxels
threshed = grid.threshold([-0.1, agn_num - 0.9])
# adding the voxels
p.add_mesh(threshed, name='sphere', show_edges=True, opacity=1.0, show_scalar_bar=False)

#p.show(use_ipyvtk=True)
```

Cluster the existing voxels and set the vertical column of cluster centers as vertical shafts
``` python linenums="1"
def grow_shafts(occ_lattice, avail_lattice):

    # extract the address of all occupied voxels
    occ_ind = np.array(np.where(occ_lattice > -1)).T

    # construct kmeans model and fit it to find the clustering
    kmeans_model = KMeans(n_clusters=4, random_state=0).fit(occ_ind)

    # extract cluster centers
    cluster_centers = np.round(kmeans_model.cluster_centers_).astype(np.int8)
    print(cluster_centers)
    # init shaft lattice
    shft_lattice = occ_lattice * 0
    # set the shafts
    for cl_cen in cluster_centers:
        shft_lattice[cl_cen[0],cl_cen[1], :] = 1
    
    return shft_lattice

    shft_lattice = grow_shafts(occ_lattice, avail_lattice)
```

Visualize vertical shafts
``` python linenums="1"
pv.set_plot_theme("document")
p = pv.Plotter(notebook=True)

base_lattice = shft_lattice

# Set the grid dimensions: shape + 1 because we want to inject our values on the CELL data
grid = pv.UniformGrid()
grid.dimensions = np.array(base_lattice.shape) + 1
# The bottom left corner of the data set
grid.origin = base_lattice.minbound - base_lattice.unit * 0.5
# These are the cell sizes along each axis
grid.spacing = base_lattice.unit 

# adding the boundingbox wireframe
p.add_mesh(grid.outline(), color="grey", label="Domain")

# adding axes
p.add_axes()
p.show_bounds(grid="back", location="back", color="#aaaaaa")

# Add the data values to the cell data
grid.cell_arrays["Agents"] = base_lattice.flatten(order="F").astype(int)  # Flatten the array!
# filtering the voxels
threshed = grid.threshold([0.9, 1.1])
# adding the voxels
p.add_mesh(threshed, name='sphere', show_edges=True, opacity=1.0, show_scalar_bar=False)

#p.show(use_ipyvtk=True)
p.show(screenshot="../data/shafts.png")
```

Save shafts to csv
``` python linenums="1"
# save the distance latice to csv
csv_path = os.path.relpath("../data/shafts.csv")
shft_lattice.to_csv(csv_path)
```

### Select the entrance voxel (highres)
Make envelope padded
``` python linenums="1"
env_padded_array = np.pad(avail_lattice, 1)
padded_minbound = avail_lattice.minbound - avail_lattice.unit
env_padded_lattice = tg.to_lattice(env_padded_array, minbound=padded_minbound, unit=avail_lattice.unit)
```

Importing the street points and public transport points
``` python linenums="1"
street_transport_path = os.path.relpath('../data/mainstreet_publictransport.xyz.txt')
street_transport = np.genfromtxt(street_transport_path, delimiter=',')
```

Distance matrix
``` python linenums="1"
# extracting the centroid of all voxels
env_cens = env_padded_lattice.centroids_threshold(-1)

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

Finding min distance to street and public transport
``` python linenums="1"
# amount of points in the list (street points and public transport points)
print(len(street_transport))

# average of the distances to each point per voxel
av_dist = []
for str_trans_points in dist_m:
    av_dist_vox = str_trans_points.sum()/len(street_transport)
    av_dist.append(av_dist_vox)
#print(av_dist)
    
# Find the voxel with the minimum average distance 
min_dist = np.min(av_dist)
print(min_dist)

# Find the index of this voxel
min_index_ent = np.argmin(av_dist)
ent_index_3d = np.unravel_index(min_index_ent, env_padded_lattice.shape)
ent_ar_3d = np.array(ent_index_3d)
print(ent_ar_3d)
```

### Creation of horizontal corridors
Find shortest paths
``` python linenums="1"
# find the number of all voxels
vox_count = avail_lattice.size 

# initialize the adjacency matrix
adj_mtrx = np.zeros((vox_count,vox_count))

# Finding the index of the available voxels in avail_lattice
avail_index = np.array(np.where(avail_lattice == 1)).T

# fill the adjacency matrix using the list of all neighbours
for vox_loc in avail_index:
    # find the 1D id
    vox_id = np.ravel_multi_index(vox_loc, avail_lattice.shape)
    # retrieve the list of neighbours of the voxel based on the stencil
    vox_neighs = avail_lattice.find_neighbours_masked(stencil, loc = vox_loc)
    # iterating over the neighbours
    for neigh in vox_neighs:
        # setting the entry to one
        adj_mtrx[vox_id, neigh] = 1.0

# construct the graph 
g = nx.from_numpy_array(adj_mtrx)

# initialize corridor lattice
cor_lattice = shft_lattice * 0
cor_flat = cor_lattice.flatten()

# for each voxel that needs to have access to shafts
occ_ind = np.array(np.where(occ_lattice > -1)).T
for a_vox in occ_ind:
    # slice the corridor lattice horizontally
    cor_floor = shft_lattice[:,:,a_vox[2]]

    # find the vertical shaft voxel indices
    shaft_vox_inds = np.array(np.where(cor_floor > 0)).T
    paths = []
    path_lens = []
    paths_entrance = []

    for shft_ind in shaft_vox_inds:
        dst_vox_ent = np.array([shft_ind[0],shft_ind[1],ent_index_3d[2]])
        # construct 1-dimensional indices
        src_ind_ent = np.ravel_multi_index(ent_ar_3d, shft_lattice.shape)
        dst_ind_ent = np.ravel_multi_index(dst_vox_ent, shft_lattice.shape)
        path_ent = nx.algorithms.shortest_paths.astar.astar_path(g, src_ind_ent, dst_ind_ent)
        paths_entrance.append(path_ent)
        for p in range(len(paths_entrance)):
            cor_flat[paths_entrance[p]] = 1 
    for shft_ind in shaft_vox_inds:
        # construct the destination address
        dst_vox = np.array([shft_ind[0],shft_ind[1],a_vox[2]])
        # construct 1-dimensional indices
        src_ind = np.ravel_multi_index(a_vox, shft_lattice.shape)
        dst_ind = np.ravel_multi_index(dst_vox, shft_lattice.shape)
        # find the shortest path
        path = nx.algorithms.shortest_paths.astar.astar_path(g, src_ind, dst_ind)

        paths.append(path)
        path_lens.append(len(path))
    # find the shortest path
    shortest_path = paths[np.array(path_lens).argmin()]
    print(shortest_path)

    # set the shortest path occupied in the
    cor_flat[shortest_path] = 1

# reshape the flat lattice
cor_lattice = cor_flat.reshape(cor_lattice.shape)
```

Visualize the corridors
``` python linenums="1"
p = pv.Plotter(notebook=True)

base_lattice = cor_lattice

# Set the grid dimensions: shape + 1 because we want to inject our values on the CELL data
grid = pv.UniformGrid()
grid.dimensions = np.array(base_lattice.shape) + 1
# The bottom left corner of the data set
grid.origin = base_lattice.minbound - base_lattice.unit * 0.5
# These are the cell sizes along each axis
grid.spacing = base_lattice.unit

# adding the boundingbox wireframe
p.add_mesh(grid.outline(), color="grey", label="Domain")

# adding the availability lattice
#init_avail_lattice.fast_vis(p)

# adding axes
p.add_axes()
p.show_bounds(grid="back", location="back", color="#aaaaaa")


# Add the data values to the cell data
grid.cell_arrays["Agents"] = base_lattice.flatten(order="F").astype(int)  # Flatten the array!
# filtering the voxels
threshed = grid.threshold([0.9, 2.1])
# adding the voxels
p.add_mesh(threshed, name='sphere', show_edges=True, opacity=1.0, show_scalar_bar=False)

#p.show(use_ipyvtk=True)
#p.show(screenshot="../data/corridors.png")
```

Save corridors to csv
``` python linenums="1"
#save the distance latice to csv
csv_path = os.path.relpath("../data/corridors.csv")
cor_lattice.to_csv(csv_path)
```

Combine shafts and corridors
``` python linenums="1"
path_lattice = shft_lattice + cor_lattice

# save the distance latice to csv
csv_path = os.path.relpath("../data/shafts_corridors.csv")
path_lattice.to_csv(csv_path)
```

### Credits
``` python linenums="1"
__author__ = "Shervin Azadi and Pirouz Nourian"
__license__ = "MIT"
__version__ = "1.0"
__url__ = "https://github.com/shervinazadi/spatial_computing_workshops"
__summary__ = "Spatial Computing Design Studio Workshop on Path Finding and Corridorfor Generative Spatial Relations"
```