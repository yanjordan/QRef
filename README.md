# QRef
`QRef` is a plugin for the crystallographic software suite `Phenix` enabling what in the literature commonly is referred to as "quantum refinement" (QR) in both real and reciprocal space, utilising the (for academic users) free software Orca as the quantum chemistry engine. This first version of `QRef` using `Phenix` was implemented under the paradigm "correctness and completeness first, performance and adherence to coding standards later".

## Theory
Refinement of macromolecules in both real and and reciprocal space relies on previous knowledge (i.e. a Bayesian prior) for the structure. This is usually encoded as a (pseudo-energy) penalty term, $E_{restraints}(\mathbf{R})$, giving rise to a target function for the refinement with the general appearance
```math
E_{total}\left(\mathbf{R}\right) = E_{exp}\left(\mathbf{R}\right) + wE_{restraints}\left(\mathbf{R}\right),
```
where $\mathbf{R}$ is the coordinate set for the current model and $w$ is a weight factor.
$E_{restraints}(\mathbf{R})$ can in turn be broken down into its components:
```math
E_{restraints}(\mathbf{R}) = E_{chem}(\mathbf{R}) + E_{SS}(\mathbf{R}) + E_{NCS}(\mathbf{R}) + \dots
```
In this QR implementation $E_{chem}(\mathbf{R})$ (which traditionally is a molecular mechanics force field) is replaced using a subtractive QM/MM scheme (using hydrogen link-atoms) according to
```math
E_{chem}(\mathbf{R}) = \sum_{i} \left(w_{QM}E_{QM1, i}(\mathbf{R_{syst1, i}}) - E_{MM1, i}(\mathbf{R})\right) + E_{MM}(\mathbf{R}),
```
where index 1 in turn indicates small, but interesting, parts of the structure. Additionally another scaling factor, $w_{QM}$, is needed due to the fact that crystallographic MM force fields are of a statistical nature, whereas $E_{QM1, i}$ represents physical energies. $\mathbf{R_{syst1, i}}$ in turn is the coordinate set for the i:th region of QM atoms. Placing the hydrogen link-atoms at
```math
\overline{r_{H_L}} = \overline{r_X} + g_{bond}\left(\overline{r_{C_L}} - \overline{r_X}\right)
```
implies that $\mathbf{R_{syst1, i}} = \mathbf{R_{syst1, i}}(\mathbf{R})$, thus the gradient for the chemical restraints is then obtained as
```math
\nabla E_{restraints}(\mathbf{R}) = \sum_{i} \left( w_{QM} \nabla E_{QM1, i}(\mathbf{R_{syst1, i}(R)}) \cdot J(\mathbf{R_{syst1,i}}; \mathbf{R}) - \nabla E_{MM1, i}(\mathbf{R}) \right) + \nabla E_{MM}(\mathbf{R})
```
where $J(\mathbf{R_{syst1,i}}; \mathbf{R})$ is the Jacobian between $\mathbf{R_{syst1,i}}$ and $\mathbf{R}$. While this obviosuly is a matrix of size $3N_{syst1, i} \times 3N$, for a single junction in one QM system this becomes a 6x6 matrix with the general shape
```math
\begin{pmatrix}
1 & 0 & 0 & 0 & 0 & 0 \\
0 & 1 & 0 & 0 & 0 & 0 \\
0 & 0 & 1 & 0 & 0 & 0 \\
(1 - g_{bond}) & 0 & 0 & g_{bond} & 0 & 0 \\
0 & (1 - g_{bond}) & 0 & 0 & g_{bond} & 0 \\
0 & 0 & (1 - g_{bond}) & 0 & 0 & g_{bond}
\end{pmatrix}.
```

## Installation
### Modules
The directory `modules` should be placed under `$PHENIX`; `qref` will thus be a new directory under `modules`, whereas the user should manually overwrite `energies.py` in `modules/cctbx_project/cctbx/geometry_restraints` and `model.py` in `modules/cctbx_project/mmtbx/model`, respectively, with the version of the file corresponding to their installation of `Phenix`.

There is a commented out guard clause in `energies.py`:<br>

    # if not os.path.exists('qm.lock') and (os.path.exists('xyz_reciprocal.lock') or os.path.exists('xyz.lock')):

This is the recommended way to use the quantum restraints, as they are not always needed. In order to make this work one has to edit the file `$PHENIX/modules/phenix/phenix/refinement/xyz_reciprocal_space.py` and `import os` as well as surround the call to `mmtbx.refinement.minimization.lbfgs(...)` in the method `run_lbfgs` in the class `run_all` with<br>

    with open('xyz_reciprocal.lock', 'w'):
        pass

and

    os.remove('xyz_reciprocal.lock')

Likewise the file `$PHENIX/modules/phenix/phenix/refinement/macro_cycle_real_space.py` should be edited in a similar manner, i.e. with an added `import os` as well as surrounding the calls to `self.minimization_no_ncs()` and `self.minimization_ncs()` in the method `refine_xyz` in the class `run` with<br>

    with open('xyz.lock', 'w'):
      pass

and

    os.remove('xyz.lock')

This implementation of `QRef` has been verified to work with `Phenix` versions 1.20.1-4487, 1.21-5207, 1.21.1-5286 as well as 1.21.2-5419.

### Scripts
The scripts in the folder `scripts` should be placed somewhere accessible by `$PATH`. The shebang in the scripts might need to be updated to point towards wherever `cctbx.python` is located.

### Orca
`Orca` can be found at [orcaforum.kofo.mpg.de](https://orcaforum.kofo.mpg.de) - follow their guide for installation. QRef has been verified to work with `Orca` versions 5.0.4 and 6.0.0.

## Usage
The general procedure to set up a quantum refinement job consists of

1. Select QM region(s). Best practises for selecting proper QM region(s) can be found at for example:
    - [doi.org/10.1002/anie.200802019](https://doi.org/10.1002/anie.200802019)
    - [doi.org/10.1016/bs.mie.2016.05.014](https://doi.org/10.1016/bs.mie.2016.05.014)
    - [doi.org/10.1021/cr5004419](https://doi.org/10.1021/cr5004419)

2. Build a model.
    - The model in the QM regions needs to make chemical sense. This for example means that the QM regions should be protonated as well as being complete.
    - The model outside of the QM region (as well as the protonation of the carbon link atom) can be incomplete.
    - `phenix.ready_set add_h_to_water=True` can be useful for this purpose.

3. Prepare restraint files for unknown residues and ligands. The script `qref_prep.py` will tell you if there are any missing restraint files.
    - This can be achieved using `phenix.ready_set` and `phenix.elbow`.

4. Prepare `syst1` files; these files define the QM regions.
    - If there is only one QM region the default is to look for a file named `syst1` by the software. For multiple QM regions the recommended, and default, naming scheme is `syst11`, `syst12`, etc.
    - Which atoms to include in the QM regions is defined using the serial number from the PDB file describing the entire model.
        - While setting `sort_atoms = False` in the input to `Phenix` should ensure that the ordering in the input model is preserved, we have encountered instances where this is not adhered to. Thus it is recommended to use `iotbx.pdb.sort_atoms` (supplied with `Phenix`) which will give you a new PDB file with the suffix `_sorted.pdb` where the atoms are sorted in the same order as that which `Phenix` uses internally. It is recommended to use the `_sorted.pdb` file as the input model for refinement, as well as the reference when defining the `syst1` files.
    - The `syst1` files allows for multiple atoms or intervals of atoms to be specified on a single line, where `,` or `blank` works as delimiters; `-` is used to indicate an interval.
    - `#` and `!` can be used to include comments in the `syst1` files.
    - The second occurence of an atom in the `syst1` files will indicate that this is a link atom, i.e. it will be replaced by a hydrogen at the appropriate position in the QM calculation.
    - Examples are included in the `examples` folder.

5. Run `qref_prep.py <model>_sorted.pdb` to generate `qref.dat` (a file containing settings for the QR interface), as well as PDB files describing the QM regions.
    - The `junctfactor` file needs to be present in the same directory as where `qref_prep.py` is run. The `junctfactor` file contain ideal QM distances for the $C_L - H_L$ bonds for the link-atoms. If another `junctfactor` file is to be used this can be specified with the `-j` or `--junctfactor` option.
    - The theory used for the ideal $C_L - H_L$ QM distance can be changed with the `-l` or `--ltype` option. Default is type 12. The options are:
        - 6: B3LYP/6-31G*
        - 7: BP(RI)/6-31G*
        - 8: BP(RI)/SVP
        - 9: BP(RI)/def2-SV(P)
        - 10: PBE(RI)/def2-SVP
        - 11: B3LYP(RI)/def2-SV(P)
        - 12: TPSS(RI)/def2-SV(P)
        - 13: B97-D(RI)/def2-SV(P)

        There needs to be a parametrisation in the `junctfactor` file for the bond one intends to cleave; it is recommended that the user inspects the `junctfactor` file to verify that there is support to cleave the intended bond type. In the case parametrisation is lacking another selection for the QM system (and in particular where the link between QM and MM occurs) needs to be made or appropriate parametrisation added to the `junctfactor` file.
    - Ideally only the input model is needed as an argument for `qref_prep.py`. If there was a need to prepare restraint files for novel residues or ligands in point 3 above, `qref_prep.py` needs to be made aware of these. This can be achieved with the `-c` or `--cif` option.
    - The output from `qref_prep.py` should be $\left(1+2 n_{syst1}\right)$ files as well as recommended selection strings for both real and reciprocal space. Additionally, a warning is given if the `syst1` file covers more than one conformation. In the case that all atoms in the `syst1` definition belong to the same conformation, this will be indicated by an `altloc` specifier.
        - `qref.dat`, which contains the settings for the QR interface. This file can be changed manually and it is a good idea to inspect that the value for `orca_binary` is the correct path for the actual Orca binary file (`qref_prep.py` tries to locate this file automatically but may sometimes fail).
        - `mm_i_c.pdb`, which is the model used to calculate $E_{MM1, i}$.
        - `qm_i_h.pdb`, which is the model used to calculate $E_{QM1, i}$.

        The output PDB files can, and probably should, be used to inspect that the QM selection is proper.
        - Two selection strings are printed on the screen, one for reciprocal space and one for real space. They are intended to be used in regards to which selection of the model to refine when crafting the input to either `phenix.refine` or `phenix.real_space_refine`, see point 7 below.

    - Harmonic (bond) distance restraints can be added through the `-rd` or `--restraint_distance` option, using the syntax `i atom1_serial atom2_serial desired_distance_in_Å force_constant`. Experience has shown that the force constant needs to be $\geq$ 2500 to achieve adherence to the restraint.

    - Harmonic (bond) angle restraints can be added through the `-ra` or `--restraint_angle` option, using the syntax `i atom1_serial atom2_serial atom3_serial desired_angle_in_degrees force_constant`, where `atom2_serial` defines the angle tip. Experience has shown that the force constant needs to be $\geq$ 10 (?) to achieve adherence to the restraint.

    - Symmetry interactions are handled through the `-t` or `--transform` option, using the syntax `i "<atoms>" R11 R12 R13 R21 R22 R23 R31 R32 R33 t1 t2 t3` for each of the desired transforms, where `<atoms>` follow the same syntax as for `syst1` files (do note the usage of quotation marks, i.e. `<atoms>` should be given as a string). `R` is the rotation matrix (in Cartesian coordinates) in row-wise order and `t` is the translation vector (in Cartesian coordinates). To obtain $R$ and $t$, find for example the fractional rotation matrix $R_{frac}$ and the fractional translation vector $t_{frac}$ for the symmetry operator of interest (which can be found through using for example Coot), together with the desired fractional unit cell translation vector, $t_{u}$. $R$ is then calculated as $R = S^{-1}R_{frac}S$ and $t = S^{-1}(t_{frac} + t_{u})$, where $S$ is the Cartesian to fractional conversion matrix for the space group the crystal belongs to (which can be found in the `SCALEn` records in a PDB file).

    - All available options for `qref_prep.py` can be seen through `-h` or `--help`.

6. The next step is to prepare the input files for Orca. Examples can be found in the `examples` folder.
    - The input files should be named `qm_i.inp`, where `i` refers to the i:th QM region; counting starts at 1.
    - The level of theory should match the junction type; using the default (type 12) the corresponding input to `Orca` then becomes `! TPSS D4 DEF2-SV(P)`.
    - Energy and gradient needs to be written to disk. This is achieved through the keyword `! ENGRAD`.
    - To read coordinates from a PDB file (`qm_i_h.pdb`) use `*pdbfile <charge> <multiplicity> qm_i_h.pdb`, where again `i` refers to the i:th QM region.
    - Custom settings can also be supplied (see the `examples` folder).

7. At this point it is possible to start a quantum refinement. It is however recommended to first create an empty file named `qm.lock` (this disables QRef - additionally, if `qref.dat` is not present in the folder QRef will not run, i.e. this step can be run before starting to set up the QR job) through for example the `touch` command, then start a refinement so that an `.eff` file is obtained (this file contains all the `Phenix` refinement settings for the current experimental data and model). A good idea is to rename this file to `phenix.params` or similar, then edit this file and make sure the following options are set:
    - For reciprocal space refinement (`phenix.refine`):
        - `refinement.pdb_interpretation.restraints_library.cdl = False`
        - `refinement.pdb_interpretation.restraints_library.mcl = False`
        - `refinement.pdb_interpretation.restraints_library.cis_pro_eh99 = False`
        - `refinement.pdb_interpretation.secondary_structure.enabled = False`
        - `refinement.pdb_interpretation.sort_atoms = False`
        - `refinement.pdb_interpretation.flip_symmetric_amino_acids = False`
        - `refinement.refine.strategy = *individual_sites individual_sites_real_space rigid_body *individual_adp group_adp tls occupancies group_anomalous`
        - `refinement.refine.sites.individual = <reciprocal selection string>`
        - `refinement.hydrogens.refine = *individual riding Auto`
        - `refinement.hydrogens.real_space_optimize_x_h_orientation = False`
        - `refinement.main.nqh_flips = False`
    - For real space refinement (`phenix.real_space_refine`):
        - `refinement.run = *minimization_global rigid_body local_grid_search morphing simulated_annealing adp occupancy nqh_flips`
        - `pdb_interpretation.restraints_library.cdl = False`
        - `pdb_interpretation.restraints_library.mcl = False`
        - `pdb_interpretation.restraints_library.cis_pro_eh99 = False`
        - `pdb_interpretation.flip_symmetric_amino_acids = False`
        - `pdb_interpretation.sort_atoms = False`
        - `pdb_interpretation.secondary_structure = False`
        - `pdb_interpretation.reference_coordinate_restraints.enabled = True`
        - `pdb_interpretation.reference_coordinate_restraints.selection = <real space selection string>`
        - `pdb_interpretation.reference_coordinate_restraints.sigma = 0.01`
        - `pdb_interpretation.ramachandran_plot_restraints.enabled = False`
    - Other options can be set as needed.

8. To run the quantum refinement job, make sure that the `qm.lock` file has been deleted, then execute either `phenix.refine phenix.params` or `phenix.real_space_refine phenix.params`. If there is a need to restart the job with different settings for `Phenix`, make sure to delete the file `settings.pickle`.

## Notes
### For COSMOS@LUNARC users
When using ORCA/6.0.0 either manually run (before you submit your job) or add to the beginning of your submit script the exports below:<br>

    export OMPI_MCA_btl='^uct,ofi'
    export OMPI_MCA_pml='ucx'
    export OMPI_MCA_mtl='^ofi'

## Todo
- ~~Add symmetry support for the QM calculations.~~ Done.
- ~~Add support for distance restraints.~~ Done.
- ~~Add support for angle restraints.~~ Done.
- Refactor code to be OOP.
- Turn QRef into a proper restraint_manager class.

## Citation
Lundgren, K. J. M., Caldararu, O., Oksanen, E., & Ryde, U. (2024). "Quantum refinement in real and reciprocal space using the *Phenix* and *ORCA* software", IUCrJ, **11**(6), 921-937.  
[doi.org/10.1107/S2052252524008406](https://doi.org/10.1107/S2052252524008406)