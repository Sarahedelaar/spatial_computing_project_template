# Functions

# Kopjes toevoegen

Import
``` python linenums="1"
import os
import topogenesis as tg
import pyvista as pv
import trimesh as tm
import numpy as np
import networkx as nx
from scipy.interpolate import RegularGridInterpolator
import matplotlib.pyplot as plt

# convert mesh to pv_mesh
def tri_to_pv(tri_mesh):
    faces = np.pad(tri_mesh.faces, ((0, 0),(1,0)), 'constant', constant_values=3)
    pv_mesh = pv.PolyData(tri_mesh.vertices, faces)
    return pv_mesh

def visualize(object_to_visualize, title, name_save):
    # load the context mesh
    context_path = os.path.relpath('../data/immediate_context.obj')
    context_mesh = tm.load(context_path)

    pv.set_plot_theme("document")

    # initiating the plotter
    p = pv.Plotter(notebook=True)

    # Create the spatial reference
    grid = pv.UniformGrid()

    # Set the grid dimensions: shape because we want to inject our values
    grid.dimensions = object_to_visualize.shape
    # The bottom left corner of the data set
    grid.origin = object_to_visualize.minbound
    # These are the cell sizes along each axis
    grid.spacing = object_to_visualize.unit

    # Add the data values to the cell data
    grid.point_arrays[title] = object_to_visualize.flatten(order="F")  # Flatten the Lattice

    # adding the meshes
    p.add_mesh(tri_to_pv(context_mesh), opacity=0.1, style='wireframe')

    # adding the volume
    opacity = np.array([0,0.6,0.6,0.6,0.6,0.6,0.6])
    p.add_volume(grid, cmap="coolwarm",opacity=opacity, shade=False)

    p.show(screenshot=name_save)

def save_image(object_to_visualize, title, name_save):
    # load the context mesh
    context_path = os.path.relpath('../data/immediate_context.obj')
    context_mesh = tm.load(context_path)

    pv.set_plot_theme("document")

    # initiating the plotter
    p = pv.Plotter(notebook=True)

    # Create the spatial reference
    grid = pv.UniformGrid()

    # Set the grid dimensions: shape because we want to inject our values
    grid.dimensions = object_to_visualize.shape
    # The bottom left corner of the data set
    grid.origin = object_to_visualize.minbound
    # These are the cell sizes along each axis
    grid.spacing = object_to_visualize.unit

    # Add the data values to the cell data
    grid.point_arrays[title] = object_to_visualize.flatten(order="F")  # Flatten the Lattice

    # adding the meshes
    p.add_mesh(tri_to_pv(context_mesh), opacity=0.1, style='wireframe')

    # adding the volume
    opacity = np.array([0,0.6,0.6,0.6,0.6,0.6,0.6])
    p.add_volume(grid, cmap="coolwarm",opacity=opacity, shade=False)

    p.screenshot(name_save)

def save_image_lattice(object_to_visualize, title, name_save, program):
    pv.set_plot_theme("document")
    p = pv.Plotter(notebook=True)

    # Set the grid dimensions: shape + 1 because we want to inject our values on the CELL data
    grid = pv.UniformGrid()
    grid.dimensions = np.array(object_to_visualize.shape) + 1
    # The bottom left corner of the data set
    grid.origin = object_to_visualize.minbound - object_to_visualize.unit * 0.5
    # These are the cell sizes along each axis
    grid.spacing = object_to_visualize.unit 

    # adding the boundingbox wireframe
    p.add_mesh(grid.outline(), color="grey", label="Domain")

    # adding axes
    p.add_axes()
    p.show_bounds(grid="back", location="back", color="#777777")

    # Add the data values to the cell data
    grid.cell_arrays[title] = object_to_visualize.flatten(order="F").astype(int)  # Flatten the array!
    # filtering the voxels
    agn_num = len(program)
    threshed = grid.threshold([-0.1, agn_num - 0.9])
    # adding the voxels
    p.add_mesh(threshed, show_edges=True, opacity=1.0, show_scalar_bar=False)

    p.screenshot(name_save)
    ```