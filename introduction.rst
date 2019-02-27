Introduction
============

GPU Optimized Monte Carlo (GOMC) is open-source software for simulating many-body molecular systems using the Metropolis Monte Carlo algorithm. GOMC is written in object oriented C++, which was chosen since it offers a good balance between code development time, interoperability with existing software elements, and code performance. The software may be compiled as a single-threaded application, a multi-threaded application using OpenMP, or to use many-core heterogeneous CPU-GPU architectures using OpenMP and CUDA. GOMC officially supports Windows 7 or newer and most modern distribution of GNU/Linux. This software has the ability to compile on recent versions of macOS; however, such a platform is not officially supported.

GOMC employs widely-used simulation file types (PDB, PSF, CHARMM-style parameter file) and supports polar and non-polar linear and branched molecules. GOMC can be used to study vapor-liquid and liquid-liquid equilibria, adsorption in porous materials, surfactant self-assembly, and condensed phase structure for complex molecules.

To cite GOMC software, please refer to `GOMC paper <https://www.sciencedirect.com/science/article/pii/S2352711018301171>`_.

GOMC supported ensembles:
-------------------------

- Canonical (NVT)
- Isobaric-isothermal (NPT)
- Grand canonical (μVT)
- Constant volume Gibbs (NVT-Gibbs) 
- Constant pressure Gibbs (NPT-Gibbs)

GOMC supported Monte Carlo moves:
---------------------------------
- Rigid-body displacement
- Rigid-body rotation
- Regrowth using coupled-decoupled configurational-bias
- Crankshaft using coupled-decoupled configurational-bias
- Intra-box swap using coupled-decoupled configurational-bias
- Intra-box `molecular exchange Monte Carlo <https://aip.scitation.org/doi/abs/10.1063/1.5025184>`_
- Inter-box swap using coupled-decoupled configurational-bias
- Inter-box `molecular exchange monter carlo <https://www.sciencedirect.com/science/article/pii/S0378381218305351>`_ 
- Volume exchange (both isotropic and anisotropic)

GOMC supported force fields:
----------------------------
- OPLS
- CHARMM 
- TraPPE
- Mie
- Martini
