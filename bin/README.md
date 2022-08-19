
## Usage example 1 - generating a single or a few input files with probe interaction count ranking 

If your structure is not appropriately pre-processed then go through steps 1-4 to protonate and clean up substrate molecules. This is the user's responsibility to verify, since RINRUS will generate models from provided structure and cannot yet run sanity checks

1. After getting a raw PDB file (`3bwm.pdb`), check for nonstandard fragments, clean up atoms or residues with alternate positions/conformations. Listed below are some possible tasks that need to be done before starting the workflow:
   a) if there are multiple conformations for a residue then keep only one conformation
   b) If starting from MD simulation, you may want to delete neutralizing counterions if they have migrated into the active site
   c) If crystallographic symmetry is embedded into the pdb, one needs to be very careful and manually "unfold" the multimers from the PDB website. This might be more common with older X-ray crystal structures. RINRUS will eventually be improved to recognize this and automate symmetry unpacking. 
  d) We have seen situations where AMBER truncates atom names of two-letter elements when writing to PDB format (turning FE into F for example). If the PDB file comes from source other than PDB repository or similar database, check all atom names, residue names, and residue ID #s.
  e) If coming from MD, check that metal coordination is correct

2. If the protein is not yet protonated, run `reduce` to generate a new protonated PDB file  (`3bwm_h.pdb`): (or protonate with other program of your choice). Skip this step if the pdb file is from MD simulation since it has been protonated already.
```bash
$HOME/git/RINRUS/bin/reduce -NOFLIP 3bwm.pdb > 3bwm_h.pdb 
``` 

3. copy the protonated PDB to a new filename (`3bwm_h.pdb > 3bwm_h_modify.pdb`)

4. Check the new PDB file. Probe doesn't recognize covalent bonds between fragments, which becomes a problem with metalloenzymes and metal-ligand coordination. Our workaround is to replace a metal coordination center with an atom that probe recognizes as a H-bond donor or acceptor to capture interactions with the coordinating ligand atoms. For example in COMT, we replace Mg with O, and save as a new PDB file.

5. Check all ligands, make sure H atoms were added correctly (may need to delete or add more H based on certain conditions). The user needs to protonate ligand and substrate properly since reduce does not recognize most substrates and may add H improperly. Check the ligand 2D drawing for the PDB entry on rcsb website for some guidance on substrate protonation if starting from scratch. Performing this step correctly is the user's responsibility! RINRUS will generate models from provided structure and does not yet have comprehensive sanity checks!
   
6. If there are any "CA" or "CB" atoms in substrate/ligands/noncanonical amino acids, replace them with "CA'" and "CB'", respectively. An example of this would be if the substrate is a polypeptide (like in Tyrosyl DNA-phosphodiesterase 1). You would not want to freeze substrate Carbon alphas/betas. Renaming CA' / CA' will cause RINRUS to ignore a peptide substrate atoms when it makes the list of frozen atoms. We plan to automate this process eventually. 

7. Use the new PDB file (`3bwm_h_modify.pdb`) to run `probe` to generate .probe file	
``` bash
$HOME/git/RINRUS/bin/probe -unformated -MC -self "all" 3bwm_h_modify.pdb > 3bwm_h_modify.probe 
```
###NOTE: When creating QM-cluster models, remember to replace metal atom in PDB if it was replaced with O in step 4

***Step 8 is one of the most important steps of the QM-cluster model building process. The user must now define the "seed". What is the seed? Typically, the seed will be the substrate(s) (or ligand in biochemical terms) participating in the chemical reaction. Any amino acid residues, co-factors, or fragments which participate in the active site catalytic breaking and forming of chemical bonds may also need to be included as part of the seed, but this will generate much larger models compare to only using the substrate***

##Using the example of PDB:3BWM, we will select the PDB residue ID# of three fragments: 300(metal Mg2+), 301 (SAM) and 302 (Catechol - the substrate) as the seed. 3BWM only has one chain, so we must specify chain A throughout. If the PDB does not have chain identifiers, you will need to specify ":XXX" where XXX is residue id number to use them in this step and beyond. If the protein is multimeric, use the chain of your choice for seed fragments. Note that some multimeric x-ray crystal structures may not necessarily have equivalent active sites!
##NOTE: If there is no Chain ID in the PDB file, our defaults are wonky and need to be improved. 
8. Run `probe2rins.py`. The seed is a comma-separated list of colon-separated pairs, the first part being the ID of the PDB subunit, the second part being the residue number in that subunit:(Chain:ResID)
``` bash
python3 $HOME/git/RINRUS/bin/probe2rins.py -f 3bwm_h_modify.probe -s A:300,A:301,A:302
```
```bash
  -f PROBEFILE  probe_file
  -s SEED       seed for select RIN, in the format of A:300,A:301,A:302
```
This produces `freq_per_res.dat`, `rin_list.dat`, `res_atoms.dat`, and `*.sif`.

9. With the res_atoms.dat file generated, use run rinrus_trim2_pdb to generate the trimmed PDB model:
```bash
python3 ~/git/RINRUS/bin/rinrus_trim2_pdb.py -s A:300,A:301,A:302 -pdb 3bwm_h_modify.pdb -c res_atom.dat
```

This generates automatically generate the entire "ladder" of possible models based on a ranking scheme which contain `res_NNN.pdb`, `res_NNN_froz_info.dat` and `res_NNN_atom_info.dat` for the all models, where `NNN` is the number of residues in that model.

#Note: if you want to generate one model based on a any ranking scheme, you will need to use the flag `-model NNN`:
```bash
python3 ~/git/RINRUS/bin/rinrus_trim2_pdb.py -s A:300,A:301,A:302 -pdb 3bwm_h_modify.pdb -c res_atom.dat -model NNN
```
```bash
  -pdb R_PDB     protonated pdbfile
  -s SEED        Chain:Resid,Chain:Resid
  -c R_ATOM      atom info for each residue (eg. res_atom.dat or res_atom-5.00.dat, etc.)
  -model METHOD  generate one or all trimmed models, if "7" is given, then
                 will generate the 7th model
```

10. The trimming procedure creates uncapped backbone pieces. Next use pymol_script.py to add capping hydrogens where bonds were broken when the model was trimmed. You must have a local copy of pymol installed! Run `pymol_scripts.py` to add hydrogens to one or more `res_NNN.pdb` files:

which
- generates a `log.pml` PyMOL input file containing commands that perform the hydrogen addition, and then
- runs PyMOL to perform the addition.
If `-resids` is specified, those residue IDs will not have hydrogens added. NOTE: This is an important part of the process and you will most likely want to put the seed residues in this list. If you don't, pymol might (probably will) reprotonate your noncanonical amino acids/substrate molecules and make very poor decisions. This step will also ignore re-protonating histidine side chain protons. Please check histidine protonation state of your pdb file before proceeding.
```bash
python3 $HOME/git/RINRUS/bin/pymol_scripts.py -resids "300,301,302" -pdbfilename res_.pdb
```
```bash
-resid "300,301,302" (if you do not want pymol to re-protonate fragments 300, 301, and 302)
-resid "300,301,302 and not name NE2" (to ignore fragments 300, 301, 302, and all NE2 atoms in the model. See [pymol boolean algebra](https://www.pymolwiki.org/index.php/Selection_Algebra) page for formatting details. 
```

Note: To properly loop pymol_script.py over all possible models, you will have to write a bash or python script to loop. An example of a bash loop is given here:
```bash
ls -lrt| grep -v slurm |awk '{print $9}'|grep -E _atom_info |cut -c 5-6 |cut -d_ -f1>list; mkdir pdbs; for i in `cat list`; do mkdir model-${i}-01; cd model-${i}-01; mv ../res_${i}.pdb .;mv ../res_${i}_atom_info.dat  .;mv ../res_${i}_froz_info.dat .; python3 ~/git/RINRUS/bin/pymol_scripts.py -resids "300,301,302" -pdbfilename *.pdb; cp *_h.pdb model-${i}_h.pdb; cp model-${i}_h.pdb ../pdbs/ ; cd ..; done
```

11. Run `write_input.py` for a single model to generate an input file for quantum chemistry packages. You will have to loop this with python or a shell script to iterate over all possible models. Simple DFT/xTB templates are included:
```bash
python3 ~/git/RINRUS/bin/write_input.py -intmp ~/git/RINRUS/bin/gaussian_input_template.txt -format gaussian -c -2 -noh res_NN.pdb -adh res_NN_h.pdb 
```
```bash
-noh NO_H_PDB     trimmed_pdb_file
-adh H_ADD_PDB    hadded_pdb_file
-intmp INPUT_TMP  template_for_write_input
-m MULTIPLICITY   multiplicity
-c LIGAND_CHARGE  charge_of_ligand
-format FMAT      input_file_format eg.'gaussian','qchem','gau-xtb'
"run python3 ~/git/RINRUS/bin/write_input.py --help" to see more information about flags
```

## Usage example 2 - generating a single or a few input files with distance-based ranking: 

1. Follow step 1-5 from example 1 to get protonated pdb. In this case we will work with `3bwm.pdb` and after step 2 of example 1 it will generate `3bwm_h.pdb`. If you have modified anything from `3bwm_h.pdb` then use that for following process.

2. To create a ranking list of residues/fragments within 5 Angstroms from either the center of mass of the seed OR the average of Cartesian coordinates of the seed fragments (in this example, A:300,A:301,A:302), run pdb_dist_rank.py. This script can be run with or without hydrogens included in the calculation of distances by adding the "-nohydro" flag to neglect hydrogen.
```bash
python3 ~/git/RINRUS/bin/pdb_dist_rank.py -pdb 3bwm_h.pdb -s A:300,A:301,A:302 -cut 5 -type avg -nohydro
python3 ~/git/RINRUS/bin/pdb_dist_rank.py -pdb 3bwm_h.pdb -s A:300,A:301,A:302 -cut 5 -type avg
python3 ~/git/RINRUS/bin/pdb_dist_rank.py -pdb 3bwm_h.pdb -s A:300,A:301,A:302 -cut 5 -type mass -nohydro
python3 ~/git/RINRUS/bin/pdb_dist_rank.py -pdb 3bwm_h.pdb -s A:300,A:301,A:302 -cut 5 -type mass
```
```bash
  -type       "mass" or "avg"
  -nohydro     If flag present, ignore hydrogen atoms from distance calculations               
  -cut COFF    cut_off_dist, default = 3 Å
  -s SEED      center_residues, examples: A:300,A:301,A:302
```
This script will generate a file named `dist_per_res-5.00.dat`, which has same format as `freq_per_res.dat`, contains information about all residue atoms within 5 Angstroms in increasing order of distance from the seed, and `res_atom-5.00.dat`, which has the same format as `res_atoms.dat`, contains information about important atoms to be included in the identified residues. 

3. Once the `res_atom-5.00.dat` file is generated, generate the trimmed PDB model using RINRUS by following steps 9 and 11 of example 1 to protonate the pdbs and generate input files.

## Usage example 3 - generating a single or a few input files with arpeggio ranking by either interaction counts or number of interaction types (will be available in public release shortly!)

## Usage example 4 - generating a single or a few input files with manual ranking (from SAPT, ML, or from some scheme that doesn't yet interface with RINRUS automatically)

Here we'll discuss the formatting of freq_per_res.dat and res_atoms.dat and how to manually construct these from an arbitrary ranking scheme or a scheme not tabulated internally with RINRUS. Will be available shortly!

## Usage example 5 - GENERATE ALL THE THINGS!!! Combinatorial model building from probe and arpeggio (will be available in public release shortly!)
## Usage example 5a - Combinatorial model building from arpeggio 
## Usage example 5b - Combinatorial model building from probe
