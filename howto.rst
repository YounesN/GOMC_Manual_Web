How to?
=======

In this section, we are providing a summary of what actions or modification need to be done in order to answer your simulaiton problem.


Build a molecule and topology file
----------------------------------

There are many open-source software that can build a molecule for you, such as `Avagadro <https://avogadro.cc/docs/getting-started/drawing-molecules/>`__ ,
`molefacture <http://www.ks.uiuc.edu/Research/vmd/plugins/molefacture/>`__ in VMD and more. Here we use molefacture features to not only build a molecule,
but also creating the topology file.

First, make sure that VMD is installed on your computer. Then, to learn how to build a single PDB file and topology file for united atom butane molecule, 
please refer to this `document <https://github.com/GOMC-WSU/Workshop/blob/master/NVT/butane/build/Molefacture.pdf>`__ .

We encourage to try to go through our workshop materials:

-   To try two days workshop, execute the following command in your terminal to clone the workshop:

    .. code-block:: bash

         $ git  clone    https://github.com/GOMC-WSU/Workshop.git --branch master --single-branch
         $ cd   Workshop

    or simply download it from `here <https://github.com/GOMC-WSU/Workshop/tree/master>`__ .

-   To try two hours workshop, execute the following command in your terminal to clone the workshop:

    .. code-block:: bash

         $ git  clone    https://github.com/GOMC-WSU/Workshop.git --branch AIChE --single-branch
         $ cd   Workshop

    or simply download it from `here <https://github.com/GOMC-WSU/Workshop/tree/AIChE>`__ .



Restart the simulation
----------------------

Make sure that in the previous simulation config file, the flag ``RestartFreq`` was activated and the restart PDB file/files (``OutputName``\_BOX_0_restart.pdb) 
and merged PSF file (``OutputName``\_merged.psf) were printed. 

In order to restart the simulation from previous simulation we need to perform the following steps to modify the config file:

1.  Set the ``Restart`` to True.

2.  Use the dumped restart PDB file to set the ``Coordinates`` for each box.

3.  Use the dumped merged PSF file to set the ``Structure`` for both boxes.

4.  It is a good practice to comment out the ``CellBasisVector`` by adding '#' at the beginning of each cell basis vector. However, GOMC will override 
    the cell basis information with the cell basis data from restart PDB file/files.

5.  Use the different ``OutputName`` to avoid overwriting the output files.


Here is the example of starting the NPT simulation of dimethyl ether, from equilibrated NVT simulation:

.. code-block:: text

    ########################################################
    # Parameters need to be modified
    ########################################################
    Restart         true

    Coordinates     0   dimethylether_NVT_BOX_0_restart.pdb

    Structure       0   dimethylether_NVT_merged.psf

    #CellBasisVector1   0	45.00	0.00	0.00
    #CellBasisVector2   0	0.00	55.00	0.00
    #CellBasisVector3   0	0.00	0.00	45.00

    OutputName          dimethylether_NPT


Here is the example of starting the NPT-GEMC simulation of dimethyl ether, from equilibrated NVT simulation:

.. code-block:: text

    ########################################################
    # Parameters need to be modified
    ########################################################
    Restart         true

    Coordinates     0   dimethylether_NVT_BOX_0_restart.pdb
    Coordinates     1   dimethylether_NVT_BOX_1_restart.pdb

    Structure       0   dimethylether_NVT_merged.psf
    Structure       1   dimethylether_NVT_merged.psf

    #CellBasisVector1   0	45.00	0.00	0.00
    #CellBasisVector2   0	0.00	55.00	0.00
    #CellBasisVector3   0	0.00	0.00	45.00

    #CellBasisVector1   1	45.00	0.00	0.00
    #CellBasisVector2   1	0.00	55.00	0.00
    #CellBasisVector3   1	0.00	0.00	45.00

    OutputName          dimethylether_NPT_GEMC


Recalculate the energy of simulation snapshot
---------------------------------------------

GOMC is capable of recalculate the energy of previous simulation snapshot, with same or different force field. Simulation snapshot is the printed molecule's 
coordinates at specific steps, which controls by ``CoordinatesFreq``. First, we need to make sure that in the previous simulation config file, the flag ``CoordinatesFreq`` 
was activated and the coordinates PDB file/files (``OutputName``\_BOX_0.pdb) and merged PSF file (``OutputName``\_merged.psf) were printed. 

In order to recalculate the energy from previous simulation we need to perform the following steps to modify the config file:

1.  Set the ``Restart`` to True.

2.  Use the dumped coordinates PDB file to set the ``Coordinates`` for each box.

3.  Use the dumped merged PSF file to set the ``Structure`` for both boxes.

4. Set the ``RunSteps`` to zero to activare the energy recalculation.

5.  Use the different ``OutputName`` to avoid overwriting the merged PSF files.

.. note::   GOMC only recalculated the energy terms and does not recalulate the thermodynamic properties. Hence, no output file, except merged PSF file, will be 
            generated.

Here is the example of recalculating energy from previous NVT simulation snapshot:

.. code-block:: text

    ########################################################
    # Parameters need to be modified
    ########################################################
    Restart         true

    Coordinates     0   dimethylether_NVT_BOX_0.pdb

    Structure       0   dimethylether_NVT_merged.psf

    RunSteps        0

    OutputName          Recalculate


Recalculate the energy of simulation snapshot
---------------------------------------------