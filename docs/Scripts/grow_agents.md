# Initialise and ABM
Load required libraries
``` python linenums="1"
# !pip install pyvista==0.28.1 ipyvtklink

import os
import topogenesis as tg
import pyvista as pv
import trimesh as tm
import pandas as pd
import numpy as np
import copy
import random
from sklearn.cluster import KMeans
np.random.seed(0)

import resources.dynamic_fields as df
import resources.functions as func

# convert mesh to pv_mesh
def tri_to_pv(tri_mesh):
    faces = np.pad(tri_mesh.faces, ((0, 0),(1,0)), 'constant', constant_values=3)
    pv_mesh = pv.PolyData(tri_mesh.vertices, faces)
    return pv_mesh
```

### Define the neighborhood (stencil)
Stencil: horizontal
``` python linenums="1"
# creating neighborhood definition 1
stencil = tg.create_stencil("von_neumann", 1, 1)
# setting the indices
stencil.set_index([0,0,0], 0)

# creating neighborhood definition 1
stencil_hor = tg.create_stencil("von_neumann", 1, 1)
# setting the indices
stencil_hor.set_index([0,0,0], 0)
stencil_hor.set_index([0,0,1], 0)
stencil_hor.set_index([0,0,-1], 0)
```

### Setup the environment
Load the envelope lattice as the availability lattice
``` python linenums="1"
# loading the lattice from csv
lattice_path = os.path.relpath('../data/envelope_highres.csv')
avail_lattice = tg.lattice_from_csv(lattice_path)
init_avail_lattice = tg.to_lattice(np.copy(avail_lattice), avail_lattice)

# loading the lattice from csv
field_path = os.path.relpath("../data/envelope_lowres.csv")
envelope_lattice = tg.lattice_from_csv(field_path)

#init_avail_lattice = tg.to_lattice(np.copy(envelope_lattice), envelope_lattice)
full_lattice = envelope_lattice * 0 + 1
```

Load program
``` python linenums="1"
program_full = pd.read_csv("../data/program_req.csv")
#program_full

program_complete = program_full.drop(["vox_amount"], 1)
#program_complete

program_prefs = program_complete.drop(["space_name","space_id"], 1)
#program_prefs
```

Load value fields
``` python linenums="1"
# loading the lattice from csv, when it is there
fields = {}
for f in program_full.columns:
    lattice_path = os.path.relpath('../data/' + f + '.csv')
    try:
        fields[f] = tg.lattice_from_csv(lattice_path)
    except:
        fields[f] = copy.deepcopy(avail_lattice * 0 + 1)
```

Initialize agents
``` python linenums="1"
def initialize_agents(avail_lattice, program_prefs, fields):   
    # initialize the occupation lattice
    occ_lattice = avail_lattice * 0 - 1

    # create a list for agents locations
    agn_locs = []
    # for each agent origin ... 
    for a_id, a_prefs in program_prefs.iterrows():
        # create a preference lattice
        pref_lattice = (avail_lattice * 0.0 + 1.0) * avail_lattice
        # choosing three available voxels
        for f, w in a_prefs.iteritems():
            pref_lattice *= fields[f] ** w

        select_id = np.argmax(pref_lattice)
        a_origin_1 = np.unravel_index(select_id, avail_lattice.shape)
        # set the initialised agent to ground level, than later there will be no floating parts
        a_origin_2 = (a_origin_1[0], a_origin_1[1], 1)
        # to prevent voxels from occupying the same place
        for n in range(5):
            if avail_lattice[a_origin_2] == 0:
                a_origin_2 = (a_origin_1[0], a_origin_1[1], 1+n)
        
        # add the origin to the list of agent locations
        agn_locs.append([a_origin_2])

        # set the origin in availablity lattice as 0 (UNavailable)
        avail_lattice[a_origin_2] = 0

        # set the origin in occupation lattice as the agent id (a_id)
        occ_lattice[a_origin_2] = a_id
        
    return occ_lattice, agn_locs
```

Save agents to csv
``` python linenums="1"
csv_path = os.path.relpath("../data/initialized_agents.csv")
occ_lattice.to_csv(csv_path)
```

Visualize initialized agents
``` python linenums="1"
pv.set_plot_theme("document")
p = pv.Plotter(notebook=True)

# Set the grid dimensions: shape + 1 because we want to inject our values on the CELL data
grid = pv.UniformGrid()
grid.dimensions = np.array(occ_lattice.shape) + 1
# The bottom left corner of the data set
grid.origin = occ_lattice.minbound - occ_lattice.unit * 0.5
# These are the cell sizes along each axis
grid.spacing = occ_lattice.unit 

# adding the boundingbox wireframe
p.add_mesh(grid.outline(), color="grey", label="Domain")

# adding axes
p.add_axes()
p.show_bounds(grid="back", location="back", color="#777777")

# Add the data values to the cell data
grid.cell_arrays["Agents"] = occ_lattice.flatten(order="F").astype(int)  # Flatten the array!
# filtering the voxels
agn_num = len(program_complete)
threshed = grid.threshold([-0.1, agn_num - 0.9])
# adding the voxels
p.add_mesh(threshed, show_edges=True, opacity=1.0, show_scalar_bar=False)

#p.show(use_ipyvtk=True)
#p.show(screenshot="../data/init_agents.png")
```

### ABM simulation
``` python linenums="1"
# make a dictionary from the max amount of voxels per space
program_dict = program_full.to_dict()
vox_am_program = program_dict["vox_amount"]
```

Functions for in the ABM simulation
``` python linenums="1"
def evaluate_voxels(a_prefs, fns):
    # find the value of neighbours
    # init the agent value array
    a_eval = np.ones(len(fns))
    # for each field...
    for f in a_prefs.keys():
        # find the raw value of free neighbours...
        vals = fields[f][fns[:,0], fns[:,1], fns[:,2]]
        # raise the the raw value to the power of preference weight of the agent
        a_weighted_vals = vals ** a_prefs[f]
        # multiply them to the previous weighted values
        a_eval *= a_weighted_vals
    return a_eval
```

Function change value of a voxel
``` python linenums="1"
# A defition to change the value of voxel in the avail_lattice and occ_lattice to occupied or not occupied (True = occupie, False = remove)
def change_value(occ_lattice, avail_lattice, id_3d, new_id, old_id=0, new_index_1d=0):
    # set the newly selected neighbour as UNavailable (0) or available (1) in the availability lattice
    if new_id > -1:
        # find the location of the newly selected neighbour
        selected_neigh_loc = np.array(id_3d).flatten()
        agn_locs[new_id].append(selected_neigh_loc)
        avail_lattice[id_3d] = 0
    else:
        agn_locs[old_id].pop(new_index_1d)
        avail_lattice[id_3d] = 1
    # set the newly selected neighbour as OCCUPIED by current agent 
    # (-1 means not-occupied so a_id)
    occ_lattice[tuple(id_3d)] = new_id
    return occ_lattice
```

Function update availability lattice
``` python linenums="1"
def update_avail_lat(occ_lattice):
    occ_all = occ_lattice > -1
    rolled_al = np.roll(occ_all, 1, axis=2)
    dif = rolled_al.astype(int) - occ_all.astype(int)
    dif[:, :, 1] = (occ_lattice[:,:, 1] == -1) * (avail_lattice[:, :, 1]) 
    new_avail_lattice = dif ==1
    return new_avail_lattice
```

Function limit buliding depth
``` python linenums="1"
def depth(occ_lattice, avail_lattice, max_depth):

    # Constructing the stencils to check in different directions

    # filled array for x direction of the stencils
    sx_ar = np.ones((max_depth + 1, 1, 1), dtype=np.int8)

    # filled array for y direction of the stencils
    sy_ar = np.ones((1, max_depth + 1, 1), dtype=np.int8)

    # east and west stencils
    se = tg.stencil(sx_ar, origin=[max_depth,0,0], function=tg.sfunc.sum, dtype=np.int8)
    sw = tg.stencil(sx_ar, origin=[0,0,0], function=tg.sfunc.sum, dtype=np.int8)
    # north and south stencils
    sn = tg.stencil(sy_ar, origin=[0,max_depth,0], function=tg.sfunc.sum, dtype=np.int8)
    ss = tg.stencil(sy_ar, origin=[0,0,0], function=tg.sfunc.sum, dtype=np.int8)

    # the depth condition executed on the all occupations
    occ_all = occ_lattice > -1
    de_con = occ_all.apply_stencil(se) < max_depth
    dw_con = occ_all.apply_stencil(sw) < max_depth
    dn_con = occ_all.apply_stencil(sn) < max_depth
    ds_con = occ_all.apply_stencil(ss) < max_depth

    # apply the conditions to the availability lattice
    return avail_lattice * de_con * dw_con * dn_con * ds_con
```

Run the simulation
``` python linenums="1"
# make a deep copy of occupation lattice
cur_occ_lattice = tg.to_lattice(np.copy(occ_lattice), occ_lattice)
lat_flat = avail_lattice.flatten()
# initialzing the list of frames
frames = [cur_occ_lattice]

squareness_weight = 1.05
# setting the time variable to 0
t = 0
n_frames = 1000

# main feedback loop of the simulation (for each time step ...)
while t<n_frames:
    print(t)
    for a_id, a_prefs in program_full.iterrows():
        # update the availability lattice to prevent the voxels to grow in the air
        #update_avail_lattice = update_avail_lat(occ_lattice)
        new_avail_lattice = depth(occ_lattice, avail_lattice, 3)
        #new_avail_lattice = avail_lattice
        # retrieve the list of the locations of the current agent
        a_locs = agn_locs[a_id]
        # initialize the list of free neighbours
        free_neighs = []
        free_neighs_1d = []
        
        # for each location of the agent
        for loc in a_locs:
            # retrieve the list of neighbours of the agent based on the stencil
            neighs = avail_lattice.find_neighbours_masked(stencil, loc = loc)
            
            # for each neighbour ... 
            for n in neighs:
                # compute 3D index of neighbour
                neigh_3d_id = np.unravel_index(n, avail_lattice.shape)
                # if the neighbour is available... (also not taken by a corridor)
                if new_avail_lattice[neigh_3d_id]:
                    # add the neighbour to the list of free neighbours
                    free_neighs.append(neigh_3d_id)
                    free_neighs_1d.append(n)                    
            fns = np.array(free_neighs)

        # check if found any free neighbour
        if len(free_neighs)>0:
            # retrieve a list of 1d locations of the neighbours + how often do these neighbours occur
            fns_1d, fn_ind, fn_count = np.unique(np.array(free_neighs_1d),return_index=True, return_counts=True)
            # find the value of each of these free neighbours
            a_eval = evaluate_voxels(program_prefs.iloc[a_id], fns[fn_ind])
            # rais the value with a squareness factor according to how often the neighbour occurs       
            new_a_eval = a_eval * (squareness_weight ** (fn_count - 1))

            # check if the the vox_amount of the agent is satisfied
            if len(a_locs) >= vox_am_program[a_id]:
                # evaluate the internal voxels with the right a_id
                internal_eval = evaluate_voxels(program_prefs.iloc[a_id], np.array(a_locs))

                # find the value of the min internal in the correct a_id
                min_internal = min(internal_eval)
                # find the maximum value of the neighbours
                max_neigh = new_a_eval.max()
                
                # if the maximum neigbhour has a 10% higher value than the min internal voxel, it will be replaced
                if max_neigh * 1.1 >= min_internal:
                    # find index of max neighbour
                    max_neigh_in = np.argmax(new_a_eval)
                    # find the 1d index of max neighbour
                    selected_neigh_1d_id = fns_1d[max_neigh_in]
                    # make it 3d location
                    selected_neigh_3d_id = np.unravel_index(selected_neigh_1d_id, avail_lattice.shape)

                    # find index of min internal
                    index_min_int = np.argmin(internal_eval)
                    # make it 3d location
                    min_int_3d_id = tuple(a_locs[index_min_int])

                    # occupy max neighbour
                    occ_lattice = change_value(occ_lattice, avail_lattice, selected_neigh_3d_id, a_id)
                    # drop the min internal
                    occ_lattice = change_value(occ_lattice, avail_lattice, min_int_3d_id, -1, a_id, index_min_int)
        
            else:
                # select the neighbour with highest evaluation
                selected_int = np.argmax(new_a_eval)
                # find the 1d index of max neighbour
                selected_neigh_1d_id = fns_1d[selected_int]
                # make it 3d location
                selected_neigh_3d_id = np.unravel_index(selected_neigh_1d_id, avail_lattice.shape)
                # change the value of the voxel
                occ_lattice = change_value(occ_lattice, avail_lattice, selected_neigh_3d_id, a_id)

            # run the update of the distance field of this agent
            fields[str(a_id)] = df.distance_field(occ_lattice, avail_lattice, a_id)
            # save the evaluation field of agent 0
            # if a_id == 0:
            #     frns = np.argwhere(init_avail_lattice > -1)
            #     vox_vals = evaluate_voxels(program_prefs.iloc[0], frns)
            #     val_lattice = tg.to_lattice(vox_vals.reshape(avail_lattice.shape), avail_lattice)
            #     func.save_image(val_lattice, "a_id = 0", "../data/evaluation/0_"+str(np.sum(occ_lattice == 0))+"_field.png")
    
    # constructing the new lattice
    new_occ_lattice = tg.to_lattice(np.copy(occ_lattice), occ_lattice)
    # adding the new lattice to the list of frames
    frames.append(new_occ_lattice)
    # adding one to the time counter
    t += 1
```

Visualizing the simulation
``` python linenums="1"
pv.set_plot_theme("document")
p = pv.Plotter(notebook=True)
base_lattice = frames[0]

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

def create_mesh(value):
    f = int(value)
    lattice = frames[f]

    # Add the data values to the cell data
    grid.cell_arrays["Agents"] = lattice.flatten(order="F").astype(int)  # Flatten the array!
    # filtering the voxels
    threshed = grid.threshold([-0.1, agn_num - 0.9])
    # adding the voxels
    p.add_mesh(threshed, name='sphere', show_edges=True, opacity=1.0, show_scalar_bar=False)
    return

p.add_slider_widget(create_mesh, [0, n_frames], title='Time', value=0, event_type="always", style="classic")
p.show(use_ipyvtk=True)
```

Saving lattice frames in csv
``` python linenums="1"
for i, lattice in enumerate(frames):
    csv_path = os.path.relpath('../data/abm_animation/abm_f_'+ f'{i:03}' + '.csv')
    lattice.to_csv(csv_path)

    csv_path = os.path.relpath('../data/new_avail_lat.csv')
avail_lattice.to_csv(csv_path)
```

### Find the cluster centers
``` python linenums="1"
cluster_centers = []
for func_id in range(0, int(np.max(frames[-1])) + 1):
    function_locs = np.array(np.where(frames[-1] == func_id)).T

    # for every multiple of 100 voxels a cluster center is created and at least 1 per agent
    nclusters = int(len(function_locs) / 100 + 1)
    kmeans_model = KMeans(n_clusters= nclusters, random_state=0).fit(function_locs)
    cluster_centers.append(np.round(kmeans_model.cluster_centers_).astype(np.int8))

# making shure it's no longer a nested cluster center list
cluster_center_list = []
for i in range(len(cluster_centers)):
    cluster_center_list.append(cluster_centers[i][0])

#creating a lattice that shows all cluster centers
cluster_center_lattice = avail_lattice * 0 - 1
for cluster_center in cluster_center_list:
    cluster_center_lattice[cluster_center[0], cluster_center[1], cluster_center[2]] = 1
    print(cluster_center)
```

Visualize voxel centers
``` python linenums="1"
pv.set_plot_theme("document")
p = pv.Plotter(notebook=True)

base_lattice = cluster_center_lattice

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
#p.show(screenshot="../data/cluster_centers.png")
```

Save voxel centers to csv
``` python linenums="1"
# save cluster centers to csv
# csv_path = os.path.relpath("../data/cluster_center_lattice.csv")
# cluster_center_lattice.to_csv(csv_path)
```

### Create floor plans
Slice the massing lattice
``` python linenums="1"
# loading the lattice from csv
lattice_path = os.path.relpath('../data/abm_animation/abm_f_1000.csv')
massing_lattice = tg.lattice_from_csv(lattice_path)

# slice the massing lattice to create floorplans
floor_lattice = tg.to_lattice(np.copy(massing_lattice), massing_lattice)
# slice to get a floor plan, the rest will be -1 (empty)
floor_lattice[:,:,:11] = -1
floor_lattice[:,:,12:] = -1
```

Visualize floorplans
``` python linenums="1"
def tri_to_pv(tri_mesh):
    faces = np.pad(tri_mesh.faces, ((0, 0),(1,0)), 'constant', constant_values=3)
    pv_mesh = pv.PolyData(tri_mesh.vertices, faces)
    return pv_mesh

# load the context mesh
context_path = os.path.relpath('../data/immediate_context.obj')
context_mesh = tm.load(context_path)

# make a dictionary from the space names connected to the id
program_dict = program_complete.to_dict()
space_list = program_dict["space_name"]

pv.set_plot_theme("document")
p = pv.Plotter(notebook=False)

base_lattice = massing_lattice

# Set the grid dimensions: shape + 1 because we want to inject our values on the CELL data
grid = pv.UniformGrid()
grid.dimensions = np.array(base_lattice.shape) + 1
# The bottom left corner of the data set
grid.origin = base_lattice.minbound - base_lattice.unit * 0.5
# These are the cell sizes along each axis
grid.spacing = base_lattice.unit 

# adding axes
p.add_axes()

# adding the meshes
p.add_mesh(tri_to_pv(context_mesh), opacity=0.1, style='wireframe')

# Add the data values to the cell data
grid.cell_arrays["Agents"] = base_lattice.flatten(order="F").astype(int)  # Flatten the array!
# filtering the voxels
threshed = grid.threshold([0, 21])

# add a legend
sargs = dict(interactive=True, label_font_size=6, shadow= True)
p.add_mesh(threshed, name='sphere', show_edges=True, opacity=1.0, show_scalar_bar=True, annotations = space_list, scalar_bar_args=sargs, cmap="tab20b")

p.camera_position = 'xy'
p.camera.zoom(3.0)
p.show(use_ipyvtk=False)
#p.show(screenshot="../data/floor_plan_legend.png")
```

### Credits
``` python linenums="1"
__author__ = "Shervin Azadi "
__license__ = "MIT"
__version__ = "1.0"
__url__ = "https://github.com/shervinazadi/spatial_computing_workshops"
__summary__ = "Spatial Computing Design Studio Workshop on Agent Based Models for Generative Spaces"
```