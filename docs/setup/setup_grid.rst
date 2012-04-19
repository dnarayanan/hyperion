.. _grid:

Coordinate grids and physical quantities
========================================

In general, coordinate grids and density grids are set using methods of the
form::

    m = Model()
    m.set_<grid_type>_grid(...)
    m.add_density_grid(density, dust)

where ``<grid_type>`` is the grid type being used, and ``dust`` is a dust file
in HDF5 format specified either by filename, or as a dust object. See
:ref:`dustfile` for more details about creating and using dust files. For
example, if you are using a dust file named ``kmh.hdf5``, you can specify this
with::

    m.add_density_grid(density, 'kmh.hdf5')

The ``add_density_grid`` method can be called multiple times if multiple
density arrays are needed (for example if different dust sizes have different
spatial distributions).

Optionally, a specific energy distribution can also be specified in
``add_density_grid`` using the ``specific_energy=`` argument::

    m.add_density_grid(density, dust, specific_energy=specific_energy)

where ``specific_energy`` is given in the same format as ``density`` (see
sections below).

.. note:: Specifying a specific energy distribution is only useful if the
          number of initial iterations for the RT code is set to zero
          (see :doc:`setup_conf`), otherwise the input specific energy
          will be overwritten with the self-consistently computed one.

Hyperion currently supports five types of 3-d grids:

* Cartesian grids
* Spherical polar grids
* Cylindrical polar grids
* AMR grids
* Octree grids

The following sections show how the different kinds of grids should be set up.

Regular 3-d grids
----------------

Geometry
^^^^^^^^

In the case of the cartesian and polar grids, the user should define the wall
position in each of the three directions, using cgs units for the spatial
coordinates, and radians for the angular coordinates. These wall positions
should be stored in one 1-d NumPy array for each dimension, with one element
more than the number of cells defined. The walls can then be used to create a
coordinate grid using methods of the form ``set_x_grid(walls_1, walls_2,
walls_3)``. The following examples demonstrate how to do this for the various
grid types

* A 10x10x10 cartesian grid from -1 pc to +1 pc in each direction::

    x = np.linspace(-pc, pc, 11)
    y = np.linspace(-pc, pc, 11)
    z = np.linspace(-pc, pc, 11)
    m.set_cartesian_grid(x, y, z)

* A 2-d 399x199 spherical polar grid::

    r = np.logspace(np.log10(rsun), np.log10(100*au), 400)
    theta = np.linspace(0., pi., 199)
    phi = np.array([0., 2*pi])
    m.set_spherical_polar_grid(r, theta, phi)

* A 3-d 100x100x10 cylindrical polar grid::

    w = np.logspace(np.log10(rsun), np.log10(100*au), 101)
    z = np.linspace(-10*au, 10*au, 101)
    phi = np.linspace(0, 2*pi, 11)
    m.set_cylindrical_polar_grid(w, z, phi)

Density and Specific Energy
^^^^^^^^^^^^^^^^^^^^^^^^^^^

For regular cartesian and polar grids, a 3-d NumPy array containing
the density array is required, for example::

    m.add_density_grid(np.ones(100,100,100), 'kmh.hdf5')

for a 100x100x100 grid. Due to Numpy array conventions, the dimensions should
be specified in reverse order, i.e. ``(n_z, n_y, n_x)`` for a cartesian grid,
``(n_phi, n_theta, n_r)`` for a spherical polar grid, or ``(n_phi, n_z, n_r)``
for a cylindrical polar grid.

Note that once you have set the grid geometry on a model, you can access
variables that make it easy (if you wish) to set up densities from analytical
equations:

* ``m.grid.gx``, ``m.grid.gy``, and ``m.grid.gz`` for cartesian grids
* ``m.grid.gr``, ``m.grid.gt``, and ``m.grid.gp`` for spherical polar grids
* ``m.grid.gw``, ``m.grid.gz``, and ``m.grid.gp`` for cylindrical polar grids

These variables are the coordinates of the center of the cells, and each of
these variables is a full 3-d array. For example, ``m.grid.gx`` is the x
position of the center of *all* the cells, and has the same shape as the
density array needs to have. In addition, the ``m.grid.shape`` variable
contains the shape of the grid. This makes it easy to use analytical
prescriptions for the density. For example, to set up a sphere of dust with
radius R in a cartesian grid, you could do::

    density = np.zeros(m.grid.shape)
    density[(m.grid.gx ** 2 + m.grid.gy ** 2 + m.grid.gz ** 2) < R ** 2] = 1.

This grid would have a density of 0 outside R, and 1 inside R. Note that of
course you should probably be using a spherical polar grid if you want to set
up a sphere of dust, but the above example can be applied to more complicated
analytical dust structures.

AMR grids
---------

Geometry
^^^^^^^^

AMR grids have to be constructed using the ``AMRGrid`` class::

    from hyperion.grid import AMRGrid
    amr = AMRGrid()

Levels can be added with::

    level = amr.add_level()

And grids can be added to a level with::

    grid = level.add_grid()

Grid objects have the following attributes which should be set:

* ``xmin`` - lower x position of the grid
* ``xmax`` - upper x position of the grid
* ``ymin`` - lower y position of the grid
* ``ymax`` - upper y position of the grid
* ``zmin`` - lower z position of the grid
* ``zmax`` - upper z position of the grid
* ``nx`` - number of cells in x direction
* ``ny`` - number of cells in y direction
* ``nz`` - number of cells in z direction
* ``quantities`` - a dictionary containing physical quantities (see below)

Once we have an AMR grid object, which we call ``amr`` here, the geometry can
be set using::

    m.set_amr_grid(amr)

The ``quantities`` attribute is unimportant for this step, as long as the
geometry is correct.

For more details on how to create or read in an AMR object, see
:ref:`amr_indepth`.

Density and Specific Energy
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Since AMR grids have a more complex structure than regular 3-d arrays, the
density should be added using an ``AMRGrid`` object. In this case, the
``quantity`` attribute should be set for each grid object. For each physical
quantity in the AMR grid, the dictionary should have an entry of the form::

    grid.quantities[<quantity>] = quantity_array

where ``<quantity>`` is a string containing the name of the quantity (e.g.
``density``) and ``quantity_array`` should be a Numpy array with dimensions
``(grid.nz, grid.ny, grid.nx)`` (see :ref:`amr_indepth` for more details).

When calling ``add_density_grid``, the density should be specified as an item
of the AMRGrid object::

    m.add_density_grid(amr_object['density'], dust_file)

for example::

    m.add_density_grid(amr['density'], 'kmh.hdf5')

Specific energies can be specified using the same kinds of objects and using
the ``specific_energy`` argument::

    m.add_density_grid(amr['density], dust_file,
                       specific_energy=amr['specific_energy'])

Note that in this example, the ``amr`` object contains the geometry, the
density, and the specific energy (i.e. it is not necessary to create a
separate AMRGrid object for each one).

Octree grids
------------

Geometry
^^^^^^^^

An `Octree <http://en.wikipedia.org/wiki/Octree>`_ is a hierarchical grid
format where each cell can be divided into eight children cells. At the top
level is a single cell that covers the whole spatial domain being considered.
To set up an Octree, the following information is needed:

* ``x``, ``y``, ``z`` - the coordinates of the center of the parent cell
* ``dx``, ``dy``, ``dz`` - the size of the parent cell
* ``refined`` a 1-d sequence of booleans giving the structure of the grid.

The ``refined`` sequence contains all the information regarding the hierarchy
of the grid, and is described in :ref:`indepth_oct`. Once this sequence is
set, the geometry can be set with::

    m.set_octree_grid(x, y, z, dx, dy, dz, refined)

Density and Specific Energy
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Densities (and optionally specific energies) should be specified in the same
manner as the regular grids, but should be specified as a 1-d Numpy array with
the same length as the ``refined`` list, where each density value corresponds
to the equivalent cell in the refined list.