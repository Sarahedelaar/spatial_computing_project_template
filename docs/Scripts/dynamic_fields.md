# Dynamic Fields

# Kopjes toevoegen

Import
``` python title="Dynamic_fields.ipynb" linenums="1"
import os
import topogenesis as tg
import pyvista as pv
import trimesh as tm
import pandas as pd
import numpy as np
import copy
```

``` python linenums="1"
def distance_field(occ_lattice, env_lattice, a_id):
    # creating neighborhood definition 1
    stencil = tg.create_stencil("von_neumann", 1, 1)
    # setting the indices
    stencil.set_index([0,0,0], 0)

    occ_array_padded = np.pad(occ_lattice, 1, mode="constant", constant_values=-1)
    env_array_padded = np.pad(env_lattice, 1, mode="constant", constant_values=False)

    padded_minbound = env_lattice.minbound - env_lattice.unit
    env_lattice_padded = tg.to_lattice(env_array_padded, minbound=padded_minbound, unit=env_lattice.unit)
    occ_lattice_padded = tg.to_lattice(occ_array_padded, minbound=padded_minbound, unit=env_lattice.unit)

    distance_lattice = env_lattice_padded * False
    a_voexls_3d_ind_padded = tuple(np.argwhere(occ_lattice_padded == a_id).T + 1)
    # print(a_voexls_3d_ind_padded)
    distance_lattice[a_voexls_3d_ind_padded] = True  

    # retrieve the neighbour list of each cell
    neighs = distance_lattice.find_neighbours(stencil)

    # set the maximum distance to sum of the size of the lattice in all dimensions.
    max_dist = np.sum(distance_lattice.shape)

    # initialize the street network distance lattice with all the street cells as 0, and all other cells as maximum distance possible
    mn_dist_lattice = 1 - distance_lattice
    mn_dist_lattice[mn_dist_lattice==1] = max_dist

    # flatten the distance lattice for easy access
    mn_dist_lattice_flat = mn_dist_lattice.flatten()

    # flatten the envelope lattice
    env_pad_lat_flat = env_lattice_padded.flatten()

    # main loop for breath-first traversal
    for i in range(1, max_dist):
        # find the neighbours of the previous step
        next_step = neighs[mn_dist_lattice_flat == i - 1]
        # find the unique neighbours
        next_unq_step = np.unique(next_step.flatten())
        # check if the neighbours of the next step are inside the envelope
        validity_condition = env_pad_lat_flat[next_unq_step]
        # select the valid neighbours
        next_valid_step = next_unq_step[validity_condition]

        # make a copy of the lattice to prevent overwriting in the memory
        mn_nex_dist_lattice_flat = np.copy(mn_dist_lattice_flat)

        # set the next step cells to the current distance
        mn_nex_dist_lattice_flat[next_valid_step] = i

        # find the minimum of the current distance and previous distances to avoid overwriting previous steps
        mn_dist_lattice_flat = np.minimum(mn_dist_lattice_flat, mn_nex_dist_lattice_flat)
        
        # check how many of the cells have not been traversed yet
        filled_check = mn_dist_lattice_flat * env_pad_lat_flat == max_dist
        # if all the cells have been traversed, break the loop
        if filled_check.sum() == 0:
            # print(i)
            break

    # reshape and construct a lattice from the street network distance list
    mn_dist_lattice = mn_dist_lattice_flat.reshape(mn_dist_lattice.shape)

    mn_dist_lattice = mn_dist_lattice.astype(float)
    mn_dist_lattice *= env_lattice_padded 

    mn_dist_lattice = env_lattice_padded - (mn_dist_lattice - mn_dist_lattice.min()) / mn_dist_lattice.max()
    return mn_dist_lattice
```
