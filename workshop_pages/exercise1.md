# Exercise 1 - Simulating a Protein-Ligand System

## Contents

1. Overview
2. Preparation
3. System Setup
4. Analysis
5. Clean Up

## Overview

We will be using the ligand binding domain of the ionotropic glutamate receptor GluR2 bound to the agonist AMPA. As it would take about 6 hours to execute the simulation of this protein-ligand system, we are going to setup the calculations and then analyse the output a of job that has been previously executed. You can compare with the results from your own runs after the workshop.

## Preparation

Please start a session on the workshop server [here](https://workshop.biosimspace.org/hub/tmplogin). You will be presented with a Jupyter session containing a file browser. Make sure to keep this browser tab open as we'll be needing it throughout. For now, press the button marked "New" near the top right and select Terminal from the drop-down menu. A new tab will open containing a terminal emulator, we'll use this for the majority of the workshop. Use:

> cd gcmc\_protoms\_workshop

to change into the master directory for this workshop.

## System Setup

The master folder for this workshop contains input files for the system.

The files necessary to set up this kind of simulation have been provided - a pdb file containing the protein structure, here protein.pdb, and a pdb file containing the ligand, here amq.pdb. We can generate all the necessary inputs for the ProtoMS simulation code through use of the python setup tools. A convenient interface is provided by the master script protoms.py. To setup a simulation of our protein-ligand system simply type:

> python2.7 $PROTOMSHOME/protoms.py -s sampling -p protein.pdb -l amq.pdb --charge -1 -r 3

Hopefully, it should be straightforward to understand the interface. The -s flag is used to request the type of simulation, in this case, simply some vanilla MC sampling, -p gives the input protein structure to use and -l provides the ligand to include in the system. Additional small molecules can be included by providing them as additional arguments to -l. The --charge flag refers to the formal charge of the ligand and is required for setup. The last argument -r 3 requests input files to perform three independent simulations. This is a common approach to lower the statistical uncertainty of the computed quantities. Typically more independent simulations are executed until the uncertainty, estimated as the standard error, falls below an acceptable limit, e.g. 0.5 kcal.mol-1.

You'll notice that we provided very little information indeed to set up a complete simulation. There is of course a great deal going on under the hood and, using the above command, ProtoMS is making a lot of decisions on your behalf.  Fortunately, the majority of simulation set up options can be controlled from the command line. For a description of commonly used options (such as the -s and -l flags above) and their arguments you can type:

> python2.7 $PROTOMSHOME/protoms.py -h

Alternatively, for a full description of all available options you can type:

> python2.7 $PROTOMSHOME/protoms.py --fullhelp

Executing protoms.py as above has performed essentially three actions:

* Prepared the protein structure
* Prepared the ligand structure
* Generated a .cmd file to execute the desired simulation in ProtoMS 

Lets look at the actions in detail and the files that were produced.

### Protein Preparation

First, the protein is truncated such that any residue further than 20 A from any ligand atom is removed. We call this a protein scoop, and it is used to significantly reduce the computational resources required to run a simulation. By default, residues between 16 and 20 A from the ligand will have their backbone fixed during the Monte Carlo simulation, whilst those less than 16 A are fully flexible. The scoop that is produced can be found in the file protein_scoop.pdb, produced by protoms.py. Inside the file, the line:

> REMARK chunk fixbackbone 1 1-2, 14-17, 22, 24-29, 41-45, 55-57, 59, 91-96, 98-99, 101-104, 121-123, 125-126, 129, 133-135, 151-154, 171-174, 192-198, 200-203, 205-206, 209-210, 214-218

will be read automatically by ProtoMS when executing and will fix the backbone of the residues indicated in this line as described above.
Second, the protein has been solvated in a droplet of water molecules with a radius of 30 A centred on the ligand(s). Hence, we will not apply periodic boundary conditions to the protein simulations. The droplet is written to a separate file, water.pdb and the first line of this PDB file (below) indicates the centre and radius of the droplet. The last number 1.5 indicates that a semi-harmonic potential with a force constant equal to 1.5 kcal.mol-1.A-1 is applied to water molecules when they move more than 30 A away from the centre of the droplet to prevent them from evaporating.

> HEADER cap 76.3885 28.0335 41.1472 30.0000 1.5

### Ligand Preparation

Performing a molecular mechanics simulation with small molecules requires the assignment of suitable parameters. Fortunately, for ProtoMS this process is automatically handled and out sourced to the very capable tools of the Amber simulation package. The antechamber program is used to assign am1-bcc partial charges to ligands whilst bonded and Lennard-Jones parameters are taken from the General Amber Force Field (GAFF). The file amq.prepi was created by antechamber and, combined with amq.frcmod (produced by the ancillary Amber utility parmchk), contains the necessary parameters for simulation. These are used together, along with some additional information, to produce the actual file used by ProtoMS to fully describe AMPA - amq.tem. This is an example of a ProtoMS template file and we will consider it's contents in more detail as some aspects may need to be fine tuned to improve simulation efficiency.

The contents of amq.tem is separated into blocks headed by lines beginning with 'mode'. Some of the key sections are highlighted and discussed below.

**mode bond/angle/dihedral**: Contain force field parameters that were not present in the default GAFF force field and were generated by the utility parmchk. These parameters are loaded by ProtoMS as a part of template file.

**mode clj**: lists atom types, charges and Lennard-Jones parameters for each atom in the solute

**mode template**: The atom section lists for each atom in the solute, its name, residue name, forcefield parameters, connectivity. The bond section lists the chemical bonds in the solute.

The connectivity information is in essence an internal coordinates representation of a molecule of AMPA. This type of data structure is also called a z-matrix. For example the line:

> atom  CD2 AMQ 3002 3002   CG AMQ   CB AMQ  DM3 DUM

Tells us that atom CD2 is bonded to atom CG, has an angle with CB and a torsion with atom DM3. If you look at the picture below you will see that this is sufficient information to define a molecular topology.
Note that the bonds/angles/torsions in the atom lines of the par file do not have to follow chemical bonds. For instance the torsion between atom HE22 and HE23 doesn't and, in fact, is called an improper torsion. The chemical bonds of the molecule should, however, be defined with a set of bond lines.
Note that some atoms are bonded to DMX atoms (X=1,2,3). These are dummy atoms. Though they are not present in the molecule of AMPA and not seen in the pdb file, they are added by ProtoMS. The three dummy atoms DM1,DM2,DM3 are present by convention to allow, for instance, a z-matrix entry for atoms CB, CG and CD2.

The z-matrix representation of the ligand is used to allow for the efficient generation of proposal moves in the MC sampling scheme. The simplest scheme by which one may generate novel configurations for an MC simulation, would simply be to choose an atom at random and shift its position slightly. Because of the diverse chemical environments in which atoms may find themselves within a drug-like molecule however, attempting such Cartesian displacements of individual nuclei would be highly inefficient. Instead ProtoMS makes changes within the internal coordinate system provided by the z-matrix, and then maps these changes back to derive the positions of atoms in Cartesian space. Such a scheme allows move proposals that capture concerted motions of atoms e.g. movement of groups around rotable bonds. Towards the end of the template files you will see lines such as:

> dihedral   CA AMQ   CB AMQ   CG AMQ  CD2 AMQ flex 2.182

The final flex keyword indicates that this degree of freedom should be sampled and the following value denotes the range of attempted move sizes (Angstroms for bonds, degrees for angles and dihedrals) that are attempted. You'll notice that by default bonds have no flex term and hence bond lengths will not change for a simulation using this template. Some dihedrals from the z-matrix are not present in the list at the bottom of the system and these are likewise not sampled. These are omitted as they are dihedrals associated with rigid degrees of freedom that show little variability, e.g. ring dihedrals in aromatic residues, and hence may be safely fixed.

### Command File

ProtoMS is executed by calling the binary file protoms3 with a command file as input. You should inspect the command file run1_bnd.cmd, that is in the same directory, to see that there are 5 sections:

* Parameter files: Set the path to input files containing force field parameters. The value of $PROTOMSHOME is taken from the environment variable.
* PDB files: Contains the coordinates to start the simulations.
* Output files: Each streamXXX corresponds to a different style of output, warnings, verbose details etc...These can be "off" and "on" and will be be printed to the standard output or could point to a file, often named after the stream. Here we have turned on info, and accept. All of the files will be written to the folder indicated by outfolder.
* Simulation parameters: In this example we have for instance set the cutoff and the temperature
* Dumps: This specifies output at specific frequencies. In this example we will write results to the file results and snapshots to all.pdb every 100000th step.
* Chunks: This is the part of the command file that actually controls what ProtoMS will do. A complex simulation is usually broken down into different chunks (i.e. steps) during the simulation. Here ProtoMS is instructed to equilibrate for 50 million moves and then simulate for 40 million. The only differences between an equilibration and simulation chunk is that during equilibration, averages are not recorded and outputs from dumps are not written. The various terms following the number of moves e.g. solvent=688 instructs ProtoMS on the relative probabilities to attempt moves of different system components. More move types are available than are shown here and performing different types of moves can fundamentally alter the type of simulation being performed.

Visualise protein_scoop.pdb, water.pdb and amq.pdb together. This is the entirety of the system we are going to simulate. Then go to analysis, as we will not execute the simulations in this workshop.

### Execution

You will **NOT** execute these calculations if taking part in the workshop, as they take a long time!

To execute the simulation you must simply invoke the protoms3 executable followed by the name of the command file. For instance to execute the first of our repeats:

> $PROTOMSHOME/protoms3 run1_bnd.cmd

This simulation should take about 6 hours to complete

## Analysis

The sample output MC run can be found in the archive named out1\_bnd.tar.gz in the example outputs folder. You can extract the data from the archive with the command:

> tar xf out1_bnd.tar.gz

You will find the results from the MC simulations in the folder starting with out1\_bnd. Perhaps the most obvious place to start is to visualise the simulation. Download all.pdb via the Jupyterhub page to your local machine and view it in your favourite molecular viewer. As you watch the trajectory you'll notice that the protein backbone is fixed in place. This corresponds to the fixbackbone line in protein_scoop.pdb, discussed above. Locate the ligand in the binding site and any nearby solvent molecules. How many waters are present in the binding site? Are they present for the entire simulation?

Another straightforward way to examine the progress of the simulation is to consider the total energy of the system. To see this we can use the analysis tools that come with ProtoMS:

> python2.7 $PROTOMSHOME/tools/calc\_series.py -f out1\_bnd/results -s total -o total_bnd

The workshop environment does not support X window forwarding so this script will simply have saved its output as total\_bnd.png. Again, via the Jupyterhub page download total\_bnd.png to your local machine to view.

Examining the plot total_bnd.png we see that the total energy of system takes a long time level off. The vertical dashed line indicates the point at which equilibration is detected using an automated significance test procedure. This is only an approximate measure of when thermal equilibrium has been reached but can be illustrative to consider. To understand what is driving this behaviour we can consider the different energy components of the system in more detail:

> python2.7 $PROTOMSHOME/tools/calc\_series.py -f out1\_bnd/results -o results_bnd

Unlike before we have not specified the -s flag to indicate which series to plot. In this case, calc_series.py will list all available series that you can plot and analyse. What we are interested in here is the total energy and the various interaction energies, so when prompted type:

> total  
> inter/solvent-solvent/sum  
> inter/amq-solvent/sum  
> inter/protein1-amq1/sum  
> inter/protein1-solvent/sum

where amq is the residue name of the ligand. The program will now estimate the equilibration time for all of the series and plot them. You will be prompted for how you want to plot the multiple series, choose "single plot + subtract last snapshot" by simply typing:

> 5

This allows us to view the behaviour of series of different magnitudes on the same plot, but all energies are now given relative to the last snapshot of the series.  The resulting graph will be saved as results_bnd.png. Copy it to your local machine and take a look.

Equilibration points for the individual series are indicated by the vertical dashed lines. It is apparent from this plot that the rate-limiting factor in convergence of the total energy is interaction between protein and solvent, and to a lesser extent interactions between solvent molecules. Notice that the interaction energies involving the ligand equilibrate much faster than the total energy. This suggests that the interactions of the ligand are largely unaffected by the changing total energy and that the conformations of the ligand and its immediate environment produced in this simulation can be considered representative. To see the interaction energies with the ligand more clearly, we can analyse them separately:

> python2.7 $PROTOMSHOME/tools/calc\_series.py -f out1\_bnd/results -o amq\_bnd -s inter/amq-solvent/sum inter/protein1-amq1/sum -p single_last0

By now you should be able to find the file with the plot. Based on this analysis we can consider about half of the simulation length to provide well equilibrated structures of the protein-ligand complex. Whilst we've chosen the ligand interaction energies, it is important to consider the energy components for the aspect of the simulation you are interested to gauge equilibration.

Another factor that is important to consider for an MC simulation is the acceptance rate of attempted moves. In the output directory find the accept file. This contains entries for every residue in the system, recording the number of attempted and successful moves. There is an inherent tension in MC between attempted move size and the resulting acceptance rate for the simulation. If your move size is too large your acceptance rate will suffer, but if the move size is too small you will not sample efficiently and simulation convergence will be poorer. Attempted move sizes are directly related to the flexibility parameters provided in amq.tem and discussed above. Find the entry for the ligand AMQ. A good rule of thumb is that your acceptance rate should be in the range of around 20-50%. The acceptance rate for AMQ falls comfortably in this range so the default flexibilities in amq.tem from the setup are about right. This will not always be the case however so you should be prepared to modify the flexibility values as required.

## Clean Up

As the workshop server only has a limited amount of disk space please remove the decompressed workshop data from this exercise. This will help to ensure that performance of the server remains crisp for all users. Remove the folder out1_bnd with:

> rm -r example\_outputs/out1\_bnd

[Next exercise](exercise2.md)
