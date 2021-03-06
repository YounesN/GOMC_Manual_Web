Input File Formats
==================
In order to run simulation in GOMC, the following files need to be provided:

- GOMC executable
- PDB file(s)
- PSF file(s)
- Parameter file
- Input file "NAME.conf" (proprietary control file)

PDB File
--------
GOMC requires only one PDB file for NVT and NPT ensembles. However, GOMC requires two PDB files for GEMC and GCMC ensembles.

What is PDB file
^^^^^^^^^^^^^^^^
The term PDB can refer to the Protein Data Bank (http://www.rcsb.org/pdb/), to a data file provided there, or to any file following the PDB format. Files in the PDB include various information such as the name of the compound, the ATOM and HETATM records containing the coordinates of the molecules, and etc. PDB widely used by NAMD, GROMACS, CHARMM, ACEMD, and Amber. GOMC ignore everything in a PDB file except for the REMARK, CRYST1, ATOM, and END records. An overview of the PDB standard can be found here:

http://www.wwpdb.org/documentation/file-format-content/format33/sect2.html#HEADER 
http://www.wwpdb.org/documentation/file-format-content/format33/sect8.html#CRYST1 
http://www.wwpdb.org/documentation/file-format-content/format33/sect9.html#ATOM

PDB contains foure major parts; ``REMARK``, ``CRYST1``, ``ATOM``, and ``END``. Here is the definition of each field and how GOMC is using them to get the information it requires.

- ``REMARK``:
  This header records present experimental  details, annotations, comments, and information not included in other records (http://www.wwpdb.org/documentation/file-format-content/format33/ sect2.html#HEADER). However, GOMC uses this header to print simulation informations.

  - **Max Displacement** (Å)
  - **Max Rotation** (Degree)
  - **Max volume exchange** (:math:`Å^3`)
  - **Monte Carlo Steps** (MC)

- ``CRYST1``:
  This header records the unit cell dimension parameters (http://www.wwpdb.org/documentation/file-format-content/format33/sect8.html#CRYST1).

  - **Lattice constant**: a,b,c (Å)
  - **Lattice angles**: :math:`\alpha, \beta, \gamma` (Degree)

- ``ATOM``:
  The ATOM records present the atomic coordinates for standard amino acids and nucleotides. They also present the occupancy and temperature factor for each atom (http://www.wwpdb.org/documentation/file-format-content/format33/sect9.html#ATOM).

  - **ATOM**: Record name
  - **serial**: Atom serial number.
  - **name**: Atom name.
  - **resName**: Residue name.
  - **chainID**: Chain identifier.
  - **resSeq**: Residue sequence number.
  - **x**: Coordinates for X (Å).
  - **y**: Coordinates for Y (Å).
  - **z**: Coordinates for Z (Å).
  - **occupancy**: GOMC uses to define which atoms belong to which box.
  - **beta**: Beta or Temperature factor. GOMC uses this value to define the mobility of the atoms. element: Element symbol.

- ``END``:
  A frame in the PDB file is terminated with the keyword.

Here are the PDB output of GOMC for the first molecule of isobutane:

.. code-block:: text

  REMARK       GOMC    122.790    3.14159    3439.817     1000000
  CRYST1     35.245     35.245     35.245       90.00       90.00     90.00
  ATOM            1         C1        ISB           1       0.911    -0.313    0.000     0.00     0.00        C
  ATOM            2         C1        ISB           1       1.424    -1.765    0.000     0.00     0.00        C
  ATOM            3         C1        ISB           1      -0.629    -0.313    0.000     0.00     0.00        C
  ATOM            4         C1        ISB           1       1.424     0.413   -1.257     0.00     0.00        C
  END

The fields seen here in order from left to right are the record type, atom ID, atom name, residue name, residue ID, x, y, and z coordinates, occupancy, temperature factor (called beta), and segment name.

The atom name is "C1" and residue name is "ISB". The PSF file (next section) contains a lookup table of atoms. These contain the atom name from the PDB and the name of the atom kind in the parameter file it corresponds to. As multiple different atom names will all correspond to the same parameter, these can be viewed "atom aliases" of sorts. The chain letter (in this case 'A') is sometimes used when packing a number of PDBs into a single PDB file.

.. Important::

  - VMD requires a constant number of ATOMs in a multi-frame PDB (multiple records terminated by "END" in a single file). To compensate for this, all atoms from all boxes in the system are written to the output PDBs of this code.
  - For atoms not currently in a box, the coordinates are set to ``< 0.00, 0.00, 0.00 >``. The occupancy is commonly just set to "1.00" and is left unused by many codes. We recycle this legacy parameter by using it to denote, in our output PDBs, the box a molecule is in (box 0 occupancy=0.00 ; box 1 occupancy=1.00)
  - The beta value in GOMC code is used to define the mobility of the molecule.

    - Beta = 0.00: molecule can move and transfer within and between boxes.
    - Beta = 1.00: molecule is fixed in its position.
    - Beta = 2.00: molecule can move within the box but cannot be transferred between boxes.

Generating PDB file
^^^^^^^^^^^^^^^^^^^

With that overview of the format in mind, the following steps describe how a PDB file is typically built.

1. A single molecule PDB is obtained. In this example, the QM software package Gaussian was used to draw the molecule, which was then edited by hand to adhere to the PDB spec properly. The end result is a PDB for a single molecule:

  .. code-block:: text

    REMARK   1 File created by GaussView 5.0.8
    ATOM     1  C1   ISB  1   0.911  -0.313    0.000  C
    ATOM     2  C1   ISB  1   1.424  -1.765    0.000  C
    ATOM     3  C1   ISB  1  -0.629  -0.313    0.000  C
    ATOM     4  C1   ISB  1   1.424   0.413   -1.257  C
    END

2. Next, packings are calculated to place the simulation in a region of vapor-liquid coexistence. There are a couple of ways to do this in Gibbs ensemble:

  - Pack both boxes to a single middle density, which is an average of the liquid and vapor densities.
  - Same as previous method, but add a modest amount to axis of one box (e.g. 10-30 A). This technique can be handy in the constant pressure Gibbs ensemble.
  - Pack one box to the predicted liquid density and the other to the vapor density.

  A good reference for getting the information needed to estimate packing is the NIST Web Book database of pure compounds:
  
  http://webbook.nist.gov/chemistry/

3. After packing is determined, a basic pack can be performed with a Packmol script. Here is one example:

  .. code-block:: text

    tolerance 3.0
    filetype pdb
    output STEP2 ISB packed BOX 0.pdb
    structure isobutane.pdb
    number 1000
    inside cube 0.1 0.1 0.1 70.20
    end structure

  Copy the above text into "pack_isobutane.inp" file, save it and run the script by typing the following line into the terminal:

  .. code-block:: bash

    $ ./packmol < pack_isobutane.inp

PSF File
--------

GOMC requires only one PSF file for NVT and NPT ensembles. However, GOMC requires two PSF files for GEMC and GCMC ensembles.

What is PSF file
^^^^^^^^^^^^^^^^

Protein structure file (PSF), contains all of the molecule-specific information needed to apply a particular force field to a molecular system. The CHARMM force field is divided into a topology file, which is needed to generate the PSF file, and a parameter file, which supplies specific numerical values for the generic CHARMM potential function. The topology file defines the atom types used in the force field; the atom names, types, bonds, and partial charges of each residue type; and any patches necessary to link or otherwise mutate these basic residues. The parameter file provides a mapping between bonded and nonbonded interactions involving the various combinations of atom types found in the topology file and specific spring constants and similar parameters for all of the bond, angle, dihedral, improper, and van der Waals terms in the CHARMM potential function. PSF file widely used by by NAMD, CHARMM, and X-PLOR.

The PSF file contains six main sections: ``remarks``, ``atoms``, ``bonds``, ``angles``, ``dihedrals``, and ``impropers`` (dihedral force terms used to maintain planarity). Each section starts with a specific header described bellow:

- ``NTITLE``: remarks on the file.
  The following is taken from a PSF file for isobutane:

  .. code-block:: text

    PSF
          3  !NTITLE
    REMARKS  original generated structure x-plor psf file
    REMARKS  topology ./Top Branched Alkanes.inp
    REMARKS  segment ISB { first NONE; last NONE; auto angles dihedrals }

- ``NATOM``: Defines the atom names, types, and partial charges of each residue type.

  .. code-block:: text

    atom ID
    segment name
    residue ID
    residue name
    atom name
    atom type
    atom charge
    atom mass

  The following is taken from a PSF file for isobutane:

  .. code-block:: text

    4000 !NATOM
    1    ISB  1  ISB    C1    CH1    0.000000   13.0190  0
    2    ISB  1  ISB    C2    CH3    0.000000   15.0350  0
    3    ISB  1  ISB    C3    CH3    0.000000   15.0350  0
    4    ISB  1  ISB    C4    CH3    0.000000   15.0350  0
    5    ISB  2  ISB    C1    CH1    0.000000   13.0190  0
    6    ISB  2  ISB    C2    CH3    0.000000   15.0350  0
    7    ISB  2  ISB    C3    CH3    0.000000   15.0350  0
    8    ISB  2  ISB    C4    CH3    0.000000   15.0350  0

  The fields in the atom section, from left to right are atom ID, segment name, residue ID, residue name, atom name, atom type, charge, mass, and an unused 0.

- ``NBOND``: The covalent bond section lists four pairs of atoms per line. The following is taken from a PSF file for isobutane:

  .. code-block:: text

    3000   !BOND:     bonds
       1   2          1  3  1  4  5  6
       5   7          5  8

- ``NTHETA``: The angle section lists three triples of atoms per line. The following is taken from a PSF file for isobutane:

  .. code-block:: text

    3000   !NTHETA:   angles
       2   1          4  2  1  3  3  1  4
       6   5          8  6  5  7  7  5  8

- ``NPHI``: The dihedral sections list two quadruples of atoms per line.

- ``NIMPHI``: The improper sections list two quadruples of atoms per line. GOMC currently does not support improper. For the molecules without dihedral or improper, PDF file look like the following:

  .. code-block:: text

    0   !NPHI: dihedrals
    0   !NIMPHI: impropers

- (other sections such as cross terms)

.. Important::

  - The PSF file format is a highly redundant file format. It repeats identical topology of thousands of molecules of a common kind in some cases. GOMC follows the same approach as NAMD, allowing this excess information externally and compiling it in the code.
  - Other sections (e.g. cross terms) contain unsupported or legacy parameters and are ignored.
  - Following the restrictions of VMD, the order of the PSF atoms must match the order in the.
  - Improper entries are read and stored, but are not currently used. Support will eventually be added for this.

Generating PSF file
^^^^^^^^^^^^^^^^^^^

The PSF file is typically generated using PSFGen. It is convenient to make a script, such as the example below, to do this:

.. code-block::text

  psfgen << ENDMOL
  topology ./Top branched Alaknes.inp segment ISB{
    pdb ./STEP2 ISB packed BOX 0.pdb
    first none
    last none
  }

  coordpdb ./STEP2 ISB packed BOX 0.pdb ISB

  writepsf ./STEP3 START ISB sys BOX 0.psf
  writepdb ./STEP3 START ISB sys BOX 0.pdb

Typically, one script is run per box to generate a finalized PDB/PSF for that box. The script requires one additional file, the NAMD-style topology file. While GOMC does not directly read or interact with this file, it's typically used to generate the PSF and, hence, is considered one of the integral file types. It will be briefly discussed in the following section.

Topology File
-------------
A CHARMM forcefield topology file contains all of the information needed to convert a list of residue names into a complete PSF structure file. The topology is a whitespace separated file format, which contains a list of atoms and their corresponding masses, and a list of residue information (charges, composition, and topology). Essentially, it is a non-redundant lookup table equivalent to the PSF file.

This is followed by a series of residues, which tell PSFGen what atoms are bonded to a given atom. Each residue is comprised of four key elements:

- A header beginning with the keyword RESI with the residue name and net charge
- A body with multiple ATOM entries (not to be confused with the PDB-style entries of the same name), which list the partial charge on the particle and what kind of atom each named atom in a specific molecule/residue is.
- A section of lines starting with the word BOND contains pairs of bonded atoms (typically 3 per line)
- A closing section with instructions for PSFGen.

Here's an example of topology file for isobutane:

.. code-block:: text

  * Custom top file -- branched alkanes *
  11
  !
  MASS 1 CH3 15.035 C !
  MASS 2 CH1 13.019 C !

  AUTOGENERATE ANGLES DIHEDRALS

  RESI ISB    0.00 !  isobutane - TraPPE
  GROUP
  ATOM C1 CH1 0.00 !  C3
  ATOM C2 CH3 0.00 !  C2-C1
  ATOM C3 CH3 0.00 !  C4
  ATOM C4 CH3 0.00 !
  BOND C1 C2 C1 C3 C1 C4
  PATCHING FIRS NONE LAST NONE

  END

.. Note:: The keyword END must be used to terminate this file and keywords related to the auto-generation process must be placed near the top of the file, after the MASS definitions.

.. Tip::

  More in-depth information can be found in the following links:

  - `Topology Tutorial`_

  .. _Topology Tutorial: http://www.ks.uiuc.edu/Training/Tutorials/science/topology/topology-tutorial.pdf

  - `NAMD Tutorial: Examining the Topology File`_

  .. _`NAMD Tutorial: Examining the Topology File`: http://www.ks.uiuc.edu/Training/Tutorials/science/topology/topology-html/node4.html

  - `Developing Topology and Parameter Files`_

  .. _Developing Topology and Parameter Files: http://www.ks.uiuc.edu/Training/Tutorials/science/forcefield-tutorial/forcefield-html/node6.html

  - `NAMD Tutorial: Topology Files`_

  .. _`NAMD Tutorial: Topology Files`: http://www.ks.uiuc.edu/Training/Tutorials/namd/namd-tutorial-win-html/node25.html

Parameter File(s)
-----------------

Currently, GOMC uses a single parameter file and the user has the two kinds of parameter file choices:

- ``CHARMM`` (Chemistry at Harvard Molecular Mechanics) compatible parameter file
- ``EXOTIC`` parameter file

If the parameter file type is not specified or if the chosen file is missing, an error will result.

Both force field file options are whitespace separated files with sections preceded by a tag. When a known tag (representing a molecular interaction in the model) is encountered, reading of that section of the force field begins. Comments (anything after a ``*`` or ``!``) and whitespace are ignored. Reading concludes when the end of the file is reached or another section tag is encountered.

CHARMM format parameter file
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
CHARMM contains a widely used model for describing energies in Monte Carlo and molecular dynamics simulations. It is intended to be compatible with other codes that use such a format, such as NAMD. See `here`_ for a general overview of the CHARMM force field.

.. _here: http://www.charmmtutorial.org/index.php/The_Energy_Function

Here's the basic CHARMM contributions that are supported in GOMC:

.. math::

  U_{\texttt{bond}}&=\sum_{\texttt{bonds}} K_b(b-b_0)^2\\
  U_{\texttt{dihedral}}&=\sum_{\texttt{dihedrals}} K_{\phi} [1+\cos(n\phi - \delta)]\\
  U_{\texttt{angle}}&=\sum_{\texttt{angles}} K_{\theta}(\theta-\theta_0)^2\\
  U_{\texttt{LJ}}&=\sum_{\texttt{nonbonded}} \epsilon_{ij}\left[\left(\frac{R_{min_{ij}}}{r_{ij}}\right)^{12}-2\left(\frac{R_{min_{ij}}}{r_{ij}}\right)^6\right]+ \frac{q_i q_j}{\epsilon r_{ij}} \\

As seen above, the following are recognized, read and used:

- ``BONDS``
  - Quadratic expression describing bond stretching based on bond length (b) in Angstrom
  – Typically, it is ignored as bonds are rigid for Monte Carlo simulations. To specify that it is to be ignored, put a very large value i.e. "999999999999" for :math:`K_b`.

  .. Note:: GOMC does not sample bond stretch.

  .. figure:: _static/bonds.png

    Oscillations about the equilibrium bond length

- ``ANGLES``
  - Describe the conformational lbehavior of an angle (:math:`\delta`) between three atoms, one of which is shared branch point to the other two. To fix any angle and ignore the related angle energy, put a very large value i.e. "999999999999" for :math:`K_\delta`.

  .. figure:: _static/angle.png

    Oscillations of 3 atoms about an equilibrium bond angle

- ``DIHEDRALS``
  - Describes crankshaft-like rotation behavior about a central bond in a series of three consecutive bonds (rotation is given as :math:`\phi`).

  .. figure:: _static/dihedrals.png

    Torsional rotation of 4 atoms about a central bond

- ``NONBONDED``
  - This tag name only should be used if CHARMM force files are being used. This section describes 12-6 (Lennard-Jones) non-bonded interactions. Non-bonded parameters are assigned by specifying atom type name followed by polarizabilities (which will be ignored), minimum energy, and (minimum radius)/2. In order to modify 1-4 interaction, a second polarizability (again, will be ignored), minimum energy, and (minimum radius)/2 need to be defined; otherwise, the same parameter will be considered for 1-4 interaction.

  .. figure:: _static/nonbonded.png

    Non-bonded energy terms (electrostatics and Lennard-Jones)

- ``NBFIX``
  - This tag name only should be used if CHARMM force field is being used. This section allows in- teraction between two pairs of atoms to be modified, done by specifying two atom type names followed by minimum energy and minimum radius. In order to modify 1-4 interaction, a second minimum energy and minimum radius need to be defined; otherwise, the same parameter will be considered for 1-4 interaction.

  .. Note:: Please pay attention that in this section we define minimum radius, not (minimum ra- dius)/2 as it is defined in the NONBONDED section.

  Currently, supported sections of the ``CHARMM`` compliant file include ``BONDS``, ``ANGLES``, ``DIHEDRALS``, ``NONBONDED``, ``NBFIX``. Other sections such as ``CMAP`` are not currently read or supported.

BONDS
^^^^^

("bond stretching") is one key section of the CHARMM-compliant file. Units for the :math:`K_b` variable in this section are in kcal/mol; the :math:`b_0` section (which represents the equilibrium bond length for that kind of pair) is measured in Angstroms.

.. code-block:: text

  BONDS
  !V(bond) = Kb(b - b0)**2
  !
  !Kb:  kcal/mole/A**2
  !b0:  A
  !
  !  Kb (kcal/mol) = Kb (K) * Boltz.  const.;
  !
  !atom type Kb b0 description
  CH3 CH1 9999999999 1.540 !  TraPPE 2 

.. note:: The :math:`K_b` value may appear odd, but this is because a larger value corresponds to a more rigid bond. As Monte Carlo force fields (e.g. TraPPE) typically treat molecules as rigid constructs, :math:`K_b` is set to a large value - 9999999999. Sampling bond stretch is not supported in GOMC.

ANGLES
^^^^^^

("bond bending"), where :math:`\theta` and :math:`\theta_0` are commonly measured in degrees and :math:`K_\theta` is measured in kcal/mol/K. These values, in literature, are often expressed in Kelvin (K). To convert Kelvin to kcal/mol/K, multiply by the Boltzmann constant – :math:`K_\theta`, 0.0019872041 kcal/mol. In order to fix the angle, it requires to set a large value for :math:`K_\theta`. By assigning a large value like 9999999999, specified angle will be fixed and energy of that angle will considered to be zero.

Here is an example of what is necessary for isobutane:

.. code-block:: text

  ANGLES
  !
  !V(angle) = Ktheta(Theta - Theta0)**2
  !
  !V(Urey-Bradley) = Kub(S - S0)**2
  !
  !Ktheta:  kcal/mole/rad**2
  !Theta0:  degrees
  !S0:  A
  !
  !  Ktheta (kcal/mol) = Ktheta (K) * Boltz.  const.
  !
  !atom types Ktheta Theta0 Kub(?)  S0(?)
  CH3 CH1 CH3 62.100125 112.00 !  TraPPE 2

Some CHARMM ANGLES section entries include ``Urey-Bradley`` potentials (:math:`K_{ub}`, :math:`b_{ub}`), in addition to the standard quadratic angle potential. The constants related to this potential function are currently read, but the logic has not been added to calculate this potential function. Support for this potential function will be added in later versions of the code.

DIHEDRALS
^^^^^^^^^

The final major bonded interactions section of the CHARMM compliant parameter file are the DIHEDRALS. Each dihedral is composed of a dihedral series of 1 or more terms. Often, there are 4 to 6 terms in a dihedral. Angles for the dihedrals' deltas are given in degrees.

Since isobutane has no dihedral, here are the parameters pertaining to 2,3-dimethylbutane:

.. code-block:: text

  DIHEDRALS
  !
  !V(dihedral) = Kchi(1 + cos(n(chi) - delta))
  !
  !Kchi:  kcal/mole
  !n:  multiplicity
  !delta:  degrees
  !
  !  Kchi (kcal/mol) = Kchi (K) * Boltz.  const.
  !
  !atom types Kchi n delta description
  X CH1 CH1 X -0.498907 0 0.0   !  TraPPE 2
  X CH1 CH1 X  0.851974 1 0.0   !  TraPPE 2
  X CH1 CH1 X -0.222269 2 180.0 !  TraPPE 2
  X CH1 CH1 X  0.876894 3 0.0   !  TraPPE 2

.. note:: The code allows the use of 'X' to indicate ambiguous positions on the ends. This is useful because this kind is often determined solely by the two middle atoms in the middle of the dihedral, according to literature.

IMPROPERS
^^^^^^^^^

Energy parameters used to describe out-of-plane rocking are currently read, but unused. The section is often blank. If it becomes necessary, algorithms to calculate the improper energy will need to be added.

NONBONDED
^^^^^^^^^

The next section of the CHARMM style parameter file is the NONBONDED. In order to use TraPPE this section of the CHARMM compliant file is critical. Here's an example with our isobutane potential model:

.. code-block:: text

  NONBONDED
  !
  !V(Lennard-Jones) = Eps,i,j[(Rmin,i,j/ri,j)**12 - 2(Rmin,i,j/ri,j)**6]
  !
  !atom ignored epsilon Rmin/2 ignored eps,1-4 Rmin/2,1-4
  !
  CH3 0.0 -0.194745992 2.10461634058 0.0 0.0 0.0 !  TraPPE 1
  CH1 0.0 -0.019872040 2.62656119304 0.0 0.0 0.0 !  TraPPE 2
  End

.. note:: The :math:`R_{min}` is different from :math:`\sigma`. :math:`\sigma` is the distance to the x-intercept (where interaction energy goes from being repulsive to positive). :math:`R_{min}` is the potential well-depth, where the attraction is maximum. To convert :math:`\sigma` to :math:`R_{min}`, simply multiply :math:`\sigma` by 0.56123102415, and flag it with a negative sign.

NBFIX
^^^^^

The last section of the CHARMM style parameter file is the NBFIX. In this section, individual pair interaction will be modified. First, pseudo non-bonded parameters have to be defined in NONBONDED and modified in NBFIX. Here?s an example if it is required to modify interaction between CH3 and CH1 atoms:

.. code-block:: text

  NBFIX
  !V(Lennard-Jones) = Eps,i,j[(Rmin,i,j/ri,j)**12 - 2(Rmin,i,j/ri,j)**6]
  !
  !atom atom epsilon Rmin eps,1-4 Rmin,1-4
  CH3 CH1 -0.294745992 1.10461634058 !
  End

Exotic Parameter File
---------------------

The exotic file is intended for use with nonstandard/specialty models of molecular interaction, which are not included in CHARMM standard. Currently, two custom interaction are included:

- ``NONBODED_MIE`` This section describes n-6 (Lennard-Jones) non-bonded interactions. The Lennard- Jones potential (12-6) is a subset of this potential. Non-bonded parameters are assigned by specifying atom type name followed by minimum energy, atom diameter, and repulsion exponent. In order to modify 1-4 interaction, a second minimum energy, atom diameter, and repulsion exponent need to be defined; otherwise, the same parameters would be considered for 1-4 interaction.
- ``NBFIX_MIE`` This section allows n-6 (Lennard-Jones) interaction between two pairs of atoms to be mod- ified. This is done by specifying two atoms type names followed by minimum energy, atom diameter, and repulsion exponent. In order to modify 1-4 interaction, a second minimum energy, atom diameter, and repulsion exponent need to be defined; otherwise, the same parameter will be considered for 1-4 interaction.

.. note:: In ``EXOTIC`` force field, the definition of atom diameter(:math:`\sigma`) is same for both NONBONDED_MIE and NBFIX_MIE.

Otherwise, the exotic file reuses the same geometry section headings - BONDS / ANGLES / DIHEDRALS / etc. The only difference in these sections versus in the CHARMM format force field file is that the energies are in Kelvin ('K'), the unit most commonly found for parameters in Monte Carlo chemical simulation literature. This precludes the need to convert to kcal/mol, the energy unit used in CHARMM.
The most frequently used section of the exotic files in the Mie potential section is NONBONDED_MIE. Here are the parameters that are used to simulate alkanes:

.. code-block:: text

  NONBONDED_MIE
  !
  !V(mie) = 4*eps*((sig ij/r ij)^n-(sig ij/r ij)^6)
  !
  !atom eps sig n eps,1-4 sig,1-4 n,1-4
  CH4 161.00 3.740 14 0.0 0.0 0.0 ! Potoff, et al. '09
  CH3 121.25 3.783 16 0.0 0.0 0.0 ! Potoff, et al. '09
  CH2  61.00 3.990 16 0.0 0.0 0.0 ! Potoff, et al. '09

.. note:: Although the units (Angstroms) are the same, the exotic file uses :math:`\sigma`, not the :math:`R_{min}` used by CHARMM. The energy in the exotic file are expressed in Kelvin (K), as this is the standard convention in the literature.

Control File (\*.conf)
----------------------
The control file is GOMC's proprietary input file. It contains key settings. The settings generally fall under three categories:

- Input/Simulation Setup
- System Settings for During Run
- Output Settings

.. note:: The control file is designed to recognize logic values, such as "yes/true/on" or "no/false/off".

Input/Simulation Setup
^^^^^^^^^^^^^^^^^^^^^^

In this section, input file names are listed. In addition, if you want to restart your simulation or use integer seed for running your simulation, you need to modify this section according to your purpose.

``Restart``
  Determines whether to restart the simulation from previous simulation or not.

  - Value 1: Boolean - True if restart, false otherwise.

``PRNG``
  Dictates how to start the pseudo-random number generator (PRNG)

  - Value 1: String

    - RANDOM: Randomizes Mersenne Twister PRNG with random bits based on the system time.

    .. code-block:: text

       #################################
       # kind {RANDOM, INTSEED}
       #################################
       PRNG RANDOM

    - INTSEED: This option "seeds" the Mersenne Twister PRNG with a standard integer. When the same integer is used, the generated PRNG stream should be the same every time, which is helpful in tracking down bugs.

``Random_Seed``
  Defines the seed number. If "INTSEED" is chosen, seed number needs to be specified; otherwise, the program will terminate.

  - Value 1: ULONG - If "INTSEED" option is selected for PRNG (See above example)

  .. code-block:: text

    #################################
    # kind {RANDOM, INTSEED}
    #################################
    PRNG INTSEED
    Random Seed 50

``ParaTypeCHARMM``
  Sets force field type to CHARMM style.

  - Value 1: Boolean - True if it is CHARMM forcefield, false otherwise.

  .. code-block:: text

    #################################
    # FORCE FIELD TYPE
    #################################
    ParaTypeCHARMM true

``ParaTypeEXOTIC``
  Sets force field type to EXOTIC style.

  - Value 1: Boolean - True if it is EXOTIC forcefield, false otherwise.

  .. code-block:: text

    #################################
    # FORCE FIELD TYPE
    #################################
    ParaTypeEXOTIC true

``ParaTypeMARTINI``
  Sets force field type to MARTINI style.

  - Value 1: Boolean - True if it is MARTINI forcefield, false otherwise.

  .. code-block:: text

    #################################
    # FORCE FIELD TYPE
    #################################
    ParaTypeMARTINI true

``Parameters``
  Provides the name and location of the parameter file to use for the simulation.

  - Value 1: String - Sets the name of the parameter file.

  .. code-block:: text

    #################################
    # FORCE FIELD TYPE
    #################################
    ParaTypeCHARMM yes
    Parameters ../../common/Par_TraPPE_Alkanes.inp

``Coordinates``
  Defines the PDB file names (coordinates) and location for each box in the system.

  - Value 1: Integer - Sets box number (starts from '0').

  - Value 2: String - Sets the name of PDB file.

  .. note:: NVT and NPT ensembles requires only one PDB file and GEMC/GCMC requires two PDB files. If the number of PDB files is not compatible with the simulation type, the program will terminate.

  Example of NVT or NPT ensemble:

  .. code-block:: text

    #################################
    # INPUT PDB FILES - NVT or NPT ensemble
    #################################
    Coordinates 0 STEP3_START_ISB_sys.pdb

  Example of Gibbs or GC ensemble:

  .. code-block:: text

    #################################
    # INPUT PDB FILES - Gibbs or GC ensemble
    #################################
    Coordinates 0 STEP3_START_ISB_sys_BOX_0.pdb
    Coordinates 1 STEP3_START_ISB_sys_BOX_1.pdb

  .. note:: In case of Restart true, the restart PDB output file from GOMC (OutputName_BOX_0_restart.pdb) can be used for each box.

  Example of Gibbs ensemble when Restart mode is active:

  .. code-block:: text

    #################################
    # INPUT PDB FILES
    #################################
    Coordinates 0 ISB_T_270_k_BOX_0_restart.pdb
    Coordinates 1 ISB_T_270_k_BOX_1_restart.pdb

``Structures``
  Defines the PSF filenames (structures) for each box in the system.

  - Value 1: Integer - Sets box number (start from '0')

  - Value 2: String - Sets the name of PSF file.

  .. note:: NVT and NPT ensembles requires only one PSF file and GEMC/GCMC requires two PSF files. If the number of PSF files is not compatible with the simulation type, the program will terminate.

  Example of NVT or NPT ensemble: 

  .. code-block:: text

    #################################
    # INPUT PSF FILES
    #################################
    Structure 0 STEP3_START_ISB_sys.psf

  Example of Gibbs or GC ensemble:

  .. code-block:: text

    #################################
    # INPUT PSF FILES
    #################################
    Structure 0 STEP3_START_ISB_sys_BOX_0.psf
    Structure 1 STEP3_START_ISB_sys_BOX_1.psf

  .. note:: In case of Restart true, the PSF output file from GOMC (OutputName merged.psf) can be used for both boxes.

  Example of Gibbs ensemble when Restart mode is active:

  .. code-block:: text

    #################################
    # INPUT PSF FILES
    #################################
    Structure 0 ISB_T_270_k_merged.psf
    Structure 1 ISB_T_270_k_merged.psf

System Settings for During Run Setup
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This section contains all the variables not involved in the output of data during the simulation, or in the reading of input files at the start of the simulation. In other words, it contains settings related to the moves, the thermodynamic constants (based on choice of ensemble), and the length of the simulation.
Note that some tags, or entries for tags, are only used in certain ensembles (e.g. Gibbs ensemble). These cases are denoted with colored text.

``GEMC``
  *(For Gibbs Ensemble runs only)* Defines the type of Gibbs Ensemble simulation you want to run. If neglected in Gibbs Ensemble, it simply defaults to const volume (NVT) Gibbs Ensemble.

  - Value 1: String - Allows you to pick between isovolumetric ("NVT") and isobaric ("NPT") Gibbs ensemble simulations.

  .. code-block:: text

    #################################
    # GEMC TYPE (DEFAULT IS NVT GEMC) 
    #################################
    GEMC NVT

``Pressure``
  If "NPT" simulation is chosen, imposed pressure (in bar) needs to be specified; otherwise, the program will terminate.
  
  - Value 1: Double - Constant pressure in bars.

  .. code-block:: text

    #################################
    # GEMC TYPE (DEFAULT IS NVT GEMC) 
    #################################
    GEMC NPT
    Pressure 5.76

``Temperature``
  Sets the temperature at which the system will run.

  - Value 1: Double - Constant temperature of simulation in degrees Kelvin.

``Rcut``
  Sets a specific radius that non-bonded interaction energy and force will be considered and calculated using defined potential function.

  - Value 1: Double - The distance to truncate the Lennard-Jones potential at.

``RcutLow``
  Sets a specific minimum possible in angstrom that reject any move that place any atom closer than specified distance.

  - Value 1: Double - The minimum possible distance between any atoms.

``LRC``
  Defines whether or not long range corrections are used.
  
  - Value 1: Boolean - True to consider long range correction. In case of using "SHIFT" or "SWITCH" potential functions, LRC will be ignored.

``Exclude``
  Defines which pairs of bonded atoms should be excluded from non-bonded interactions.

  - Value 1: String - Allows you to choose between "1-2", "1-3", and "1-4".
  
    1-2
      All interactions pairs of bonded atoms, except the ones that separated with one bond, will be considered and modified using 1-4 parameters defined in parameter file.
    1-3
      All interaction pairs of bonded atoms, except the ones that separated with one or two bonds, will be considered and modified using 1-4 parameters defined in parameter file.
    1-4
      All interaction pairs of bonded atoms, except the ones that separated with one, two or three bonds, will be considered using non-bonded parameters defined in parameter file.
      
    .. note:: The default value is "1-4".

    .. note:: In CHARMM force field, the 1-4 interaction needs to be considered. Choosing "Exclude 1-3" will modify 1-4 interaction based on 1-4 parameter in parameter file. If a kind force field is used, where 1-4 interaction needs to be ignored, such as TraPPE, either "exclude 1-4" needs to be chosen or 1-4 parameter needs to be assigned a value of zero in the parameter file.

``Potential``
  Defines the potential function type to calculate non-bonded interaction energy and force between atoms.

  - Value 1: String - Allows you to pick between "VDW", "SHIFT" and "SWITCH".
    
    VDW
      Nonbonded interaction energy and force calculated based on n-6 (Lennard-Johns) equation. This function will be discussed further in the Intermolecular energy and Virial calculation section.

      .. code-block:: text

        #################################
        # SIMULATION CONDITION
        #################################
        Temperature 270.00
        Potential VDW
        LRC true
        Rcut 10
        Exclude 1-4

    SHIFT
      This option forces the potential energy to be zero at Rcut distance. This function will be discussed further in the Intermolecular energy and Virial calculation section.

      .. code-block:: text

        #################################
        # SIMULATION CONDITION
        #################################
        Temperature 270.00
        Potential SHIFT
        LRC false
        Rcut 10
        Exclude 1-4

    SWITCH
      This option smoothly forces the potential energy to be zero at Rcut distance and starts modifying the potential at Rswitch distance. Depending on force field type, specific potential function will be applied. These functions will be discussed further in the Intermolecular energy and Virial calculation section.

``Rswitch``
  In the case of choosing "SWITCH" as potential function, a distance is set in which non-bonded interaction energy is truncated smoothly from to cutoff distance.

  - Value 1: Double - Define switch distance in angstrom. If the "SWITCH" function is chosen, Rswitch needs to be defined; otherwise, the program will be terminated.

``ElectroStatic``
  Considers coulomb interaction or not. This function will be discussed further in the Inter- molecular energy and Virial calculation section.

  - Value 1: Boolean - True if coulomb interaction needs to be considered and false if not.

    .. note:: If MARTINI force field was used and charged molecule was used in simulation, ElectroStatic needs to be turn on. MARTINI force field uses short range coulomb interaction with constant dielectric 15.0.

``Ewald``
  Considers standard Ewald summation method for electrostatic calculation. This function will be discussed further in the Intermolecular energy and Virial calculation section.

  - Value 1: Double - True if Ewald summation calculation needs to be considered and false if not.

    .. note:: By default, ``ElectroStatic`` will be set to true if Ewald summation method was used to calculate coulomb interaction.

``CachedFourier``
  Considers storing the reciprocal terms for Ewald summation calculation in order to improve the code performance. This option would increase the code performance with the cost of memory usage.

  - Value 1: Boolean - True to store reciprocal terms of Ewald summation calculation and false if not.

    .. note:: By default, CachedFourier will be set to true if not value was set.

``Tolerance``
  Specifies the accuracy of the Ewald summation calculation. Ewald separation parameter and number of reciprocal vectors for the Ewald summation are determined based on the accuracy parameter.

  - Value 1: Double - Sets the accuracy in Ewald summation calculation. A reasonable value for te accuracy is 0.00001.

    .. note:: If "Ewald" was chosen and no value was set for Tolerance, the program will be terminated.
    
``Dielectric``
  Defines dielectric constant for coulomb interaction in MARTINI force field.

  - Value 1: Double - Sets dielectric value used in coulomb interaction.

    .. note:: In MARTINI force field, Dielectric needs to be set to 15.0. If MARTINI force field was chosen and if Dielectric was not specified, a default value of 15.0 will be assigned.

``PressureCalc``
  Considers to calculate the pressure or not. If it is set to true, the frequency of pressure calculation need to be set.

  - Value 1: Boolean - True enabling pressure calculation during the simulation, false disabling pressure calculation.

  - Value 2: Ulong - The frequency of calculating the pressure.
  
``1-4scaling``
  Defines constant factor to modify intra-molecule coulomb interaction.

  - Value 1: Double - A fraction number between 0.0 and 1.0.
  
    .. note:: CHARMM force field uses a value between 0.0 and 1.0. In MARTINI force field, it needs to be set to 1.0 because 1-4 interaction will not be modified in this force field.

    .. code-block:: text

      #################################
      # SIMULATION CONDITION
      #################################
      ElectroStatic true
      Ewald true
      Tolerance 0.00001
      1-4scaling 0.0

``RunSteps``
  Sets the total number of steps to run (one move is performed for each step) (cycles = this value / number of molecules in the system)
  
  - Value 1: Ulong - Total run steps

``EqSteps``
  Sets the number of steps necessary to equilibrate the system; averaging will begin at this step.

  - Value 1: Ulong - Equilibration steps

``AdjSteps``
  Sets the number of steps per adjustment to the maximum constants associated with each move (e.g. maximum distance in xyz to displace, the maximum volume in :math:`Å^3` to swap, etc.)
  
  - Value 1: Ulong - Number of steps per move adjustment

    .. code-block:: text

      #################################
      # STEPS
      #################################
      RunSteps 25000000
      EqSteps 5000000
      AdjSteps 1000

``ChemPot``
  For Grand Canonical (GC) ensemble runs only: Chemical potential at which simulation is run.

  - Value 1: String - The resname to apply this chemical potential.
  - Value 2: Double - The chemical potential value in degrees Kelvin (should be negative).

  .. note:: For binary systems, include multiple copies of the tag (one per residue kind).

  .. note:: If there is a molecule kind that cannot be transfer between boxes (in PDB file the beta value is set to 1.00 or 2.00), an arbitrary value (e.g. 0.00) can be assigned to the resname.

  .. code-block:: text

    #################################
    # Mol.  Name Chem.  Pot.  (K)
    #################################
    ChemPot AR -968

``Fugacity``
  For Grand Canonical (GC) ensemble runs only: Fugacity at which simulation is run.
  
  - Value 1: String - The resname to apply this fugacity.
  - Value 2: Double - The fugacity value in bar.

  .. note:: For binary systems, include multiple copies of the tag (one per residue kind).
  
  .. note:: If there is a molecule kind that cannot be transfer between boxes (in PDB file the beta value is set to 1.00 or 2.00) an arbitrary value e.g. 0.00 can be assigned to the resname.

  .. code-block:: text

    #################################
    # Mol.  Name Fugacity (bar)
    #################################
    Fugacity AR 0.1
    Fugacity Si 0.0
    Fugacity O 0.0

``DisFreq``
  Fractional percentage at which displacement move will occur.
  
  - Value 1: Double - % Displacement

``RotFreq``
  Fractional percentage at which rigid rotation move will occur.
  
  - Value 1: Double - % Rotatation

``IntraSwapFreq``
  Fractional percentage at which molecule will be removed from a box and inserted into the same box using configurational bias algorithm.

  - Value 1: Double - % Intra molecule swap

``RegrowthFreq``
  Fractional percentage at which part of the molecule will be deleted and then regrown using configurational bias algorithm.

  - Value 1: Double - % Molecular growth

``VolFreq``
  For isobaric-isothermal ensemble and Gibbs ensemble runs only: Fractional percentage at which molecule will be removed from one box and inserted into the other box using configurational bias algorithm.

  - Value 1: Double - % Volume swaps

``SwapFreq``
  For Gibbs and Grand Canonical (GC) ensemble runs only: Fractional percentage at which molecule swap move will occur.

  - Value 1: Double - % Molecule swaps

  .. code-block:: text

    #################################
    # MOVE FREQEUNCY
    #################################
    DisFreq 0.49
    RotFreq 0.10
    VolFreq 0.01
    SwapFreq 0.20
    IntraSwapFreq 0.10
    RegrowthFreq 0.10

  .. note:: All move percentages should add up to 1.0; otherwise, the program will terminate.

``useConstantArea``
  For Isobaric-Isothermal ensemble and Gibbs ensemble runs only: Considers to change the volume of the simulation box by fixing the cross-sectional area (x-y plane).

  - Value 1: Boolean - If true volume will change only in z axis, If false volume will change with constant axis ratio.

  .. note:: By default, useConstantArea will be set to false if no value was set. It means, the volume of the box will change in a way to maintain the constant axis ratio.

``FixVolBox0``
  For adsorption simulation in NPT Gibbs ensemble runs only: Changing the volume of fluid phase (Box 1) to maintain the constant imposed pressure and temperature, while keeping the volume of adsorbed phase (Box 0) fix.

  - Value 1: Boolean - If true volume of adsorbed phase will remain constant, If false volume of adsorbed phase will change.

``CellBasisVector``
  Defines the shape and size of the simulation periodic cell. ``CellBasisVector1``, ``CellBasisVector2``, ``CellBasisVector3`` represent the cell basis vector :math:`a,b,c`, respectively. This tag may occur multiple times. It occurs once for NVT and NPT, but twice for Gibbs ensemble or GC ensemble.

  - Value 1: Integer - Sets box number (first box is box '0'). 
  - Value 2: Double - x value of cell basis vector in Angstroms.
  - Value 3: Double - y value of cell basis vector in Angstroms.
  - Value 4: Double - z value of cell basis vector in Angstroms.

  .. note:: If the number of defined boxes were not compatible to simulation type, the program will be terminated.

  Example for NVT and NPT ensemble. In this example, each vector is perpendicular to the other two (:math:`\alpha = 90, \beta = 90, \gamma = 90`), as indicated by a single x, y, or z value being specified by each and making a rectangular 3-D box:

  .. code-block:: text

    #################################
    # BOX DIMENSION #, X, Y, Z
    #################################
    CellBasisVector1 0 40.00 00.00 00.00
    CellBasisVector2 0 00.00 40.00 00.00
    CellBasisVector3 0 00.00 00.00 80.00

  Example for Gibbs ensemble and GC ensemble ensemble. In this example, In the first box, only vector a and c are perpendicular to each other (:math:`\alpha = 90, \beta = 90, \gamma = 120`), and making a non-orthogonal simulation cell with the cell length :math:`a, b, c` of 36.91, 39.91, and 76.98 Angstroms, respectively. In the second box, each vector is perpendicular to the other two (:math:`\alpha = 90, \beta = 90, \gamma = 90`), as indicated by a single x, y, or z value being specified by each and making a cubic box:

  .. code-block:: text
  
    #################################
    # BOX DIMENSION #, X, Y, Z
    #################################
    CellBasisVector1 0 36.91 00.00 00.00
    CellBasisVector2 0 -18.45 31.96 00.00
    CellBasisVector3 0 00.00 00.00 76.98
    
    CellBasisVector1 1 60.00 00.00 00.00
    CellBasisVector2 1 00.00 60.00 00.00
    CellBasisVector3 1 00.00 00.00 60.00

  .. warning:: In case of ``Restart true``, box dimension does not need to be specified. If it is specified, program will read it but it will be ignored and replaced by the printed cell dimensions and angles in the restart PDB output file from GOMC (``OutputName_BOX_0_restart.pdb`` and ``Output_Name_BOX_1_restart.pdb``).

``CBMC_First``
  Number of CBMC trials to choose the first atom position (Lennard-Jones trials for first seed growth).

  - Value 1: Integer - Number of initial insertion sites to try.

``CBMC_Nth``
  Number of CBMC trials to choose the later atom positions (Lennard-Jones trials for first seed growth).

  - Value 1: Integer - Number of LJ trials for growing later atom positions.

``CBMC_Ang``
  Number of CBMC bending angle trials to perform for geometry (per the coupled-decoupled CBMC scheme).

  - Value 1: Integer - Number of trials per angle.

``CBMC_Dih``
  Number of CBMC dihedral angle trials to perform for geometry (per the coupled-decoupled CBMC scheme).

  - Value 1: Integer - Number of trials per dihedral.

  .. code-block:: text

    #################################
    # CBMC TRIALS
    #################################
    CBMC_First 10
    CBMC_Nth 4
    CBMC_Ang 100
    CBMC_Dih 30

Output Controls
^^^^^^^^^^^^^^^

This section contains all the values that control output in the control file. For example, certain variables control the naming of files dumped of the block-averaged thermodynamic variables of interest, the PDB files, etc.

``OutputName``
  Unique name for simulation used to name the block average, PDB, and PSF output files.
  
  - Value 1: String - Unique phhrase to identify this system.

  .. code-block:: text

    #################################
    # OUTPUT FILE NAME
    #################################
    OutputName ISB T 270 K

``CoordinatesFreq``
  Controls output of PDB file (coordinates). If PDB dumping was enabled, one file for NVT or NPT and two files for Gibbs ensemble or GC ensemble will be dumped into ``OutputName_BOX_n.pdb``, where n defines the box number.

  - Value 1: Boolean - "true" enables dumping these files; "false" disables dumping.

  - Value 2: Ulong - Steps per dump PDB frame. It should be less than or equal to RunSteps. If this keyword could not be found in configuration file, its value will be assigned a default value to dump 10 frames.

  .. note:: The PDB file contains an entry for every ATOM, in all boxes read. This allows VMD (which requires a constant number of atoms) to properly parse frames, with a bit of help. Atoms that are not currently in a specific box are given the coordinate (0.00, 0.00, 0.00). The occupancy value corresponds to the box a molecule is currently in (e.g. 0.00 for box 0; 1.00 for box 1).

  .. note:: At the beginning of simulation, a merged PSF file will be dumped into OutputName merged.pdb, in which all boxes will be dumped. It also contains the topology for every molecule in both boxes, corresponding to the merged PDB format. Loading PDB files into merged PSF file in VMD allows the user to visualize and analyze the results. In addition, this file can be used to load into GOMC once restart simulation was active.

``RestartFreq``
  Controls the output of the last state of simulation at a specified step in PDB files (coordinates) OutputName BOX n restart.pdb, where n defines the box number. Header part of this file contains important information and will be needed to restart the simulation:

  - Simulation cell dimensions and angles.
  - Maximum amount of displacement (Å), rotation (:math:`\delta`), and volume (:math:`Å^3`) that used in Displacement, Rotation, and Volume move.
  
  If PDB dumping was enabled, one file for NVT or NPT and two files for Gibbs ensemble or GC ensemble will be dumped.

  - Value 1: Boolean - "true" enables dumping these files; "false" disables dumping.
  - Value 2: Ulong - Steps per dump last state of simulation to PDB files. It should be less than or equal to RunSteps. If this keyword could not be found in the configuration file, RestartFreq value will be assigned by default.

  .. note:: The restart PDB file contains only ATOM that exist in each boxes at specified steps. This allows the user to load this file into GOMC once restart simulation was active.
  .. note:: CoordinatesFreq must be a common multiple of RestartFreq or vice versa.

``ConsoleFreq``
  Controls the output to STDIO ("the console") of messages such as acceptance statistics, and run timing info. In addition, instantaneously-selected thermodynamic properties will be output to this file.

  - Value 1: Boolean - "true" enables message printing; "false" disables dumping.
  - Value 2: Ulong - Number of steps per print. If this keyword could not be found in the configuration file, the value will be assigned by default to dump 1000 output for RunSteps greater than 1000 steps and 100 output for RunSteps less than 1000 steps.

``BlockAverageFreq``
  Controls the block averages output of selected thermodynamic properties. Block averages are averages of thermodynamic values of interest for chunks of the simulation (for post-processing of averages or std. dev. in those values).

  - Value 1: Boolean - "true" enables printing block average; "false" disables it.
  - Value 2: Ulong - Number of steps per block-average output file. If this keyword cannot be found in the configuration file, its value will be assigned a default to dump 100 output.

``HistogramFreq``
  Controls the histograms. Histograms are a binned listing of observation frequency for a specific thermodynamic variable. In this code, they also control the output of a file containing energy/molecule samples; it only will be used in GC ensemble simulations for histogram reweighting purposes.

  - Value 1: Boolean - "true" enables printing histogram; "false" disables it.
  - Value 2: Ulong - Number of steps per histogram output file. If this keyword cannot be found in the configuration file, a value will be assigned by default to dump 1000 output for RunSteps greater than 1000 steps and 100 output for RunSteps less than 1000 steps.

  .. code-block:: text

    #################################
    # STATISTICS Enable, Freq.
    #################################
    CoordinatesFreq   true 10000000
    RestartFreq       true 1000000
    ConsoleFreq       true 100000
    BlockAverageFreq  true 100000
    HistogramFreq     true 10000

The next section controls the output of the energy/molecule sample file and the distribution file for molecule counts, commonly referred to as the "histogram" output. This section is only required if Grand Canonical ensemble simulation was used.

``DistName``
  Sets short phrase to naming molecule distribution file.

  - Value 1: String - Short phrase which will be combined with *RunNumber* and *RunLetter* to use in the name of the binned histogram for molecule distribution.
  
``HistName``
  Sets short phrase to naming energy sample file.

  - Value 1: String - Short phrase, which will be combined with *RunNumber* and *RunLetter*, to use in the name of the energy/molecule count sample file.

``RunNumber``
  Sets a number, which is a part of *DistName* and *HistName* file name.
  
  - Value 1: Uint – Run number to be used in the above file names.

``RunLetter``
  Sets a letter, which is a part of *DistName* and *HistName* file name.
  
  - Value 1: Character – Run letter to be used in above file names.

``SampleFreq``
  Controls histogram sampling frequency.

  - Value 1: Uint – the number of steps per histogram sample.

  .. code-block:: text

    #################################
    # OutHistSettings
    #################################
    DistName   dis
    HistName   his
    RunNumber  5
    RunLetter  a
    SampleFreq 200
    
``OutEnergy, OutPressure, OutMolNumber, OutDensity, OutVolume, OutSurfaceTension``
  Enables/Disables for specific kinds of file output for tracked thermodynamic quantities

  - Value 1: Boolean – "true" enables message output of block averages via this tracked parameter (and in some cases such as entry, components); "false" disables it.
  - Value 2: Boolean – "true" enables message output of a fluctuation into the console file via this tracked parameter (and in some cases, such as entry, components); "false" disables it.

  The keywords are available for the following ensembles

  ====================  ====================  ====================  ====================
  Keyword               NVT                   NPT & Gibbs           GC
  ====================  ====================  ====================  ====================
  OutEnergy             :math:`\checkmark`    :math:`\checkmark`    :math:`\checkmark`
  OutPressure           :math:`\checkmark`    :math:`\checkmark`    :math:`\checkmark`
  OutMolNumber                                :math:`\checkmark`    :math:`\checkmark`
  OutDensity                                  :math:`\checkmark`    :math:`\checkmark`
  OutVolume                                   :math:`\checkmark`    :math:`\checkmark`
  OutSurfaceTension     :math:`\checkmark`                                            
  ====================  ====================  ====================  ====================

  Here is an example:
  
  .. code-block:: text

    #################################
    # ENABLE: BLK AVE., FLUC.
    #################################
    OutEnergy         true true
    OutPressure       true true
    OutMolNum         true true
    OutDensity        true true
    OutVolume         true true
    OutSurfaceTention false false