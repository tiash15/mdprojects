gmx pdb2gmx   -f foxs1_final.pdb   -o processed_wt.gro   -p topol.top   -i posre.itp   -ff charmm36-jul2022   -water tip3p   -ignh   -ter


# 1. Define box
gmx editconf \
  -f processed_wt.gro \
  -o boxed_wt.gro \
  -c -d 1.2 -bt dodecahedron



# 2. Solvate
gmx solvate \
  -cp boxed_wt.gro \
  -cs spc216.gro \
  -o solvated_wt.gro \
  -p topol.top


# create ioins.mdp
; ions.mdp - used for generating the tpr for genion
integrator              = steep
emtol                   = 1000.0
emstep                  = 0.01
nsteps                  = 50000
nstlist                 = 1
cutoff-scheme           = Verlet
ns_type                 = grid
pbc                     = xyz
coulombtype             = PME
rcoulomb                = 1.2

; CHARMM36m requires force-switching, not plain cutoff
vdwtype                 = cutoff
vdw-modifier            = force-switch
rvdw-switch             = 1.0
rvdw                    = 1.2

; CHARMM36m: no dispersion correction
DispCorr                = no


gmx grompp -f ions.mdp -c solvated_wt.gro -p topol.top -o ions.tpr -maxwarn 1

choose SOL when prompted

# create em.mdp

; Energy Minimization for CHARMM36m FOXS1-DNA
integrator              = steep
emtol                   = 1000.0
emstep                  = 0.01
nsteps                  = 50000

; Neighbor searching
cutoff-scheme           = Verlet
nstlist                 = 20
rlist                   = 1.2
pbc                     = xyz

; Electrostatics
coulombtype             = PME
rcoulomb                = 1.2
pme_order               = 4
fourierspacing          = 0.12

; VdW (Standard CHARMM Force-Switching)
vdwtype                 = cutoff
vdw-modifier            = force-switch
rvdw-switch             = 1.0
rvdw                    = 1.2
DispCorr                = no

; Constraints
constraints             = h-bonds



 gmx grompp -f em.mdp -c ions_wt.gro -p topol.top -o em.tpr -maxwarn 1


gmx mdrun -v -deffnm em


gmx make_ndx -f em.gro -o index.ndx

select protein | dna

gmx grompp -f nvt.mdp -c em.gro -r em.gro -p topol.top -n index.ndx -o nvt.tpr -maxwarn 1

gmx mdrun -v -deffnm nvt -nb gpu -ntmpi 1 -ntomp 8 -pme gpu

gmx energy -f nvt.edr -o temperature.xvg
check temp


# create npt.md

; GROMACS .mdp file - NVT/NPT Equilibration
; Updated for 200 ps duration and CHARMM36m compatibility

; 1. Run Control
integrator              = md        ; leap-frog integrator
nsteps                  = 100000    ; 2 * 100,000 = 200,000 fs = 200 ps
dt                      = 0.002     ; 2 fs

; 2. Output Control
nstxout                 = 500       ; save coordinates every 1.0 ps
nstvout                 = 500       ; save velocities every 1.0 ps
nstenergy               = 500       ; save energies every 1.0 ps
nstlog                  = 500       ; update log file every 1.0 ps

; 3. Neighbor Searching & Electrostatics
cutoff-scheme           = Verlet    ; Buffered neighbor searching
ns_type                 = grid      ; search neighboring grid cells
nstlist                 = 20        ; 40 fs (standard for GPU/Verlet)
rcoulomb                = 1.2       ; short-range electrostatic cutoff (nm)
rvdw                    = 1.2       ; short-range van der Waals cutoff (nm)
vdwtype                 = cutoff
vdw-modifier            = force-switch
rvdw-switch             = 1.0       ; CHARMM-specific VdW switching

; 4. Electrostatics (PME)
coulombtype             = PME       ; Particle Mesh Ewald
pme_order               = 4         ; cubic interpolation
fourierspacing          = 0.16      ; Standard for CHARMM36m (efficiency)

; 5. Temperature Coupling
tcoupl                  = V-rescale                     ; modified Berendsen thermostat
tc-grps                 = Protein_DNA Water_and_ions    ; Case-sensitive match for .ndx
tau_t                   = 0.1     0.1                   ; time constant (ps)
ref_t                   = 300     300                   ; reference temperature (K)

; Pressure coupling is ADDED
pcoupl                  = C-rescale             ; or Parrinello-Rahman
pcoupltype              = isotropic             ; applies pressure equally in X, Y, Z
tau_p                   = 2.0                   ; time constant (ps)
ref_p                   = 1.0                   ; reference pressure (bar)
compressibility         = 4.5e-5                ; isothermal compressibility of water
refcoord_scaling        = com
; 7. Periodic Boundary Conditions
pbc                     = xyz       ; 3D PBC

; 8. Dispersion Correction
DispCorr                = no        ; CHARMM uses force-switching, usually no DispCorr

; 9. Constraint Algorithms
continuation            = yes        ; not first dynamics run
constraint_algorithm    = lincs     ; holonomic constraints 
constraints             = h-bonds   ; bonds involving H are constrained
lincs_iter              = 1         ; accuracy of LINCS
lincs_order             = 4         ; also related to accuracy

; 10. Position Restraints
define                  = -DPOSRES  ; Placed at bottom per convention

gmx grompp -f npt.mdp -c nvt.gro -t nvt.cpt -r nvt.gro -p topol.top -n index.ndx -o npt.tpr


gmx mdrun -v -deffnm npt -nb gpu -ntmpi 1 -ntomp 8 -pme gpu


# create md.mdp

integrator              = md
dt                      = 0.002
nsteps                  = 150000000     ; 300 ns

nstxout-compressed      = 5000          ; save every 10 ps
nstxout                 = 0
nstvout                 = 0
nstenergy               = 5000
nstlog                  = 5000

continuation            = yes
constraint_algorithm    = lincs
constraints             = h-bonds
lincs_iter              = 1
lincs_order             = 4

cutoff-scheme           = Verlet
nstlist                 = 20
rlist                   = 1.2

vdwtype                 = cutoff
vdw-modifier            = force-switch
rvdw-switch             = 1.0
rvdw                    = 1.2
DispCorr                = no

coulombtype             = PME
rcoulomb                = 1.2
pme_order               = 4
fourierspacing          = 0.16

tcoupl                  = V-rescale
tc-grps                 = Protein_DNA  Water_and_ions
tau_t                   = 0.1          0.1
ref_t                   = 300          300

pcoupl                  = Parrinello-Rahman
pcoupltype              = isotropic
tau_p                   = 2.0
ref_p                   = 1.0
compressibility         = 4.5e-5

pbc                     = xyz
gen_vel                 = no

