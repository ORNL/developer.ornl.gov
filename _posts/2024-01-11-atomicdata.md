---
author: David M. Rogers
title: Collecting Atomic Datasets
tags: ["chemistry", "curation"]
---

There are many, many datasets of atomic structures out there.
However, most of them define and use their own storage format.
In this post, I'll go over how to import completed
VASP calculations to
[ASE.db format](https://wiki.fysik.dtu.dk/ase/ase/db/db.html).
VASP is a very popular code for electronic structure calculations, and ASE
is a natural choice among many possible alternative formats
because it offers a builtin web interface, is linked to
analysis routines, and already has [many databases](https://cmr.fysik.dtu.dk/)
published in this format.

![database viewer](/images/ase_db.png)

There are, however, several notable python packages intended
to wrap VASP calculations for various purposes,

* [JARVIS](https://pages.nist.gov/jarvis/#intro): A Toolkit for atomistic data-driven materials design.

* [AMSET: ab initio scattering and transport](https://hackingmaterials.lbl.gov/amset/)

* [AiiDA](https://aiida-vasp.readthedocs.io/en/latest/concepts/immigrations.html)

* [Ferroelectric Search Database](https://github.com/blondegeek/ferroelectric_search_site) - state of the art, but slightly custom plotly and d3js visualization of json-formatted calculation output data.
  Described in Tess E. Smidt, Stephanie A. Mack, Sebastian E. Reyes-Lillo, Anubhav Jain & Jeffrey B. Neaton, An automatically curated first-principles database of ferroelectrics [Scientific Data 7: 72 (2020)](https://www.nature.com/articles/s41597-020-0407-9).

* [pymatgen+atomate](https://workshop.materialsproject.org/lessons/05_automated_dft/Lesson/#lesson-2-parsing-output-files)

* [WFL toolkit](https://github.com/libAtoms/workflow)
  - [wfl Python toolkit for creating machine learning interatomic potentials and related atomistic simulation workflows](https://doi.org/10.1063/5.0156845)

* [Materials Design Ontology](https://github.com/LiUSemWeb/Materials-Design-Ontology) - a well-designed, parsed datastructure for storing calculation inputs and outputs with extremely flexible search capabilities.

These all accomplish the data parsing/loading step, but
many rely on ASE tools anyway, and there is additional
development effort required to support each data format.

There's also a worthwhile
[write-up on the basic process used since 2014](https://kitchingroup.cheme.cmu.edu/blog/2014/03/26/writing-VASP-calculations-to-ase-db-formats/).


## Environment Setup

Working through [ASE's tutorial on the subject](https://wiki.fysik.dtu.dk/ase/tutorials/tut06_database/database.html),
we find that the package requires both ase and flask
(for serving the database to as a webpage).

```
python3 -m venv $HOME/venvs/asedata
. $HOME/venvs/asedata
pip install -U pip
pip install ase
pip install flask
```

This sets up a python virtual environment with ase
ready to load.

ASE has the ability to write VASP input files
an run calculations if you also set up
a few
[environment variables](https://wiki.fysik.dtu.dk/ase/ase/calculators/vasp.html#environment-variables).

    export ASE_VASP_COMMAND='mpirun vasp_std'
    export VASP_PP_PATH=$HOME/vasp/mypps # directory containing pseudopotential directories: potpaw (LDA XC) potpaw_GGA (PW91 XC) and potpaw_PBE (PBE XC)
    export ASE_VASP_VDW=$HOME/<path-to-vdw_kernel.bindat-folder>

The last is needed if VASP is run using `luse_vdw=True`.  See [VASP vdW wiki](https://www.vasp.at/wiki/index.php/VdW-DF_functional_of_Langreth_and_Lundqvist_et_al.) for more instructions.


## Importing a VASP Calculation

Note that VASP property calculations are run with
[`IBRION=-1`][IBRION] (in the INCAR),
molecular dynamics use `IBRION=0`
and values larger than `0` are different types
of structural energy minimization.

Let's look at importing these files interactively
before trying to script the process.

    python

    >>> import ase
    >>> from ase.calculators.vasp import Vasp
    >>> calc = Vasp(directory=".", restart=True)

Unfortunately, this threw an error since
my data did not consistently contain `vasprun.xml` files.
Even when it does, however, the `calc` object created
above cannot access all the steps of an MD trajectory
or geometry optimization.

So, I used the raw `ase.io` utilities:

    >>> from ase.io import vasp
    >>> mol = vasp.read_vasp("POSCAR")
    >>> mol
    Atoms(symbols='Co18Li89Mn27Ni45O180', pbc=True, cell=[[14.96899093742643, 0.0530040030442156, 0.0204766354319995], [-0.0083848137156942, 14.356681727895436, 0.1221912366595893], [0.1542564531462889, -0.0426485156867605, 14.020652773840293]], constraint=FixAtoms(indices=[0, 1, 2, ...]))
    >>> out = vasp.read_vasp_out("OUTCAR")
    >>> out.get_potential_energy()
    -1606.03563302
    >>> out.get_forces().shape
    (311, 3)
    >>> out.get_stresses()
    ase.calculators.calculator.PropertyNotImplementedError
    >>> out.get_charges().shape
    ase.calculators.calculator.PropertyNotImplementedError
    >>> out.get_magnetic_moments().shape
    (311,)
    >>> out.calc.kpts
    >>> out.calc

    >>> outs = vasp.read_vasp_out("OUTCAR", slice(None))
    >>> len(out)
    200

Reading through the source code of `vasp_outcar_parsers.py`
shows the data is parsed into `results` in the following class:

    >>> from ase.calculators.singlepoint import SinglePointDFTCalculator
    >>> help(SinglePointDFTCalculator)
    get_bz_k_points()
    get_fermi_level()
    get_homo_lumo()
    get_number_of_spins()
    get_property(name)
    export_properties() ~> KeyError: 'magmom'
                    cause: no _defineprop('magmom') call in ase/outputs.py
    todict() ~> {}
    >>> outs[0].calc.results.keys()
    dict_keys(['magmom', 'magmoms', 'forces', 'free_energy', 'energy'])

So, there's plenty of data available to archive in the `OUTCAR`.


### Putting it together

The following code forms the basis for a script
that ingests the OUTCAR from the directory 
where the calculation was run.
It adds some extra key-value pairs
to the database to help track where this
calculation fits into the larger project.

    import os
    import re

    from ase.io import vasp
    from ase.db import connect

    def parse_incar(fname) -> dict[str,str]:
        data = {}
        keyval = re.compile(r'^ *([^ ]+) *=([^!]*)')# (!.*)?')
        with open(fname, encoding='ascii') as f:
            for line in f.readlines():
                m = keyval.match(line)
                if m is not None:
                    data[m[1]] = m[2].strip()
        return data

    def add_to_db(db, metadata):
        params = parse_incar("INCAR")

        outs = vasp.read_vasp_out("OUTCAR", slice(None))
        for frame, out in enumerate(outs):
            homo, lumo = out.calc.get_homo_lumo()
            out.calc.parameters = params
            id = db.write(out, key_value_pairs={
              **metadata,
              "frame":     frame+1,
              "homo":      homo,
              "lumo":      lumo,
            })

    def main(argv):
        assert len(argv) >= 7, f"Usage: {argv[0]} <calc.db> <wdir> <system> <trial> <phase> <run_type> [solute]"

        dbname = argv[1]
        wdir = argv[2]
        
        metadata = {
            "system": argv[3],
            "trial": int(argv[4]),
            "phase": argv[5],
            "run_type": argv[6],
            "solute": argv[7] if len(argv) > 7 else "",
        }

        db = connect(dbname)

        os.chdir(wdir)
        add_to_db(db, metadata)

    if __name__=="__main__":
        import sys
        main(sys.argv)
    
    #for row in db.select():
    #    break
    #row.key_value_pairs
    #atoms = row.toatoms()
    #atoms.calc.results.keys() # yes, all result data is there
    #

It's best to run this python script from the command-line
for each directory.  This way all of the important metadata
can be captured for each run.


## Setting up Metadata

From the [database documentation](https://wiki.fysik.dtu.dk/ase/ase/db/db.html),
it's a good idea to setup the database's metadata using,

    ase db calc.db --set-metadata metadata.json

Use a `metadata.json` file like this one,

```
{ "title": "Surface Adsorption Study of C on Pt",
  "key_descriptions": {
    "name": ["Name", "System name", ""],
    "phase": ["Phase", "surface|gas|liquid|solid", ""],
    "trial": ["Trial", "1-based sequence number for this calculation", ""],
    "solute": ["Solute", "Name of solute/adsorbate/vacancy (if present)", ""],
    "run_type": ["Run Type", "optimization (opt), calculation (calc), molecular dynamics (md), or nudged elastic band (neb.__)", ""],
    "frame": ["Frame", "1-based step number during optimization or dynamics", ""],
    "homo": ["HOMO", "Highest occupied molecular orbital", "eV"],
    "lumo": ["LUMO", "Lowest unoccupied molecular orbital", "eV"]
  },
  "default_columns": [
    "id",
    "name",
    "phase",
    "solute",
    "run_type",
    "frame",
    "formula",
    "energy"
  ]
}
```

Note that json is very picky about formatting (double-quotes,
commas in the right places), so it's easy to create
syntax errors that are hard to spot at a glance.


## Browsing the Database

Serving the webpage notes that an additional step is
needed to actually display the molecules using jsmol:

Download `Jmol-*-binary.zip` from
https://sourceforge.net/projects/jmol/files/Jmol/,
and extract the jsmol subdirectory into ase's static folder:

    $ cd jmol-*
    $ unzip jsmol.zip
    $ mv jsmol $VIRTUAL_ENV/lib/python3.*/site-packages/ase/db/static/

Then serve using:

    $ ase db mydata.db -w

## Parallelizing

Given the basic `add_to_db(db, metadata)` function above,
it's not too difficult to create a parallel version
to import a whole list of calculations at a time.
The basic idea is to parse data files in parallel.

Although it's possible to [reserve spots in the database](https://wiki.fysik.dtu.dk/ase/ase/db/db.html#writing-rows-in-parallel),
adding to the database is fast, and reserving then writing creates
contention between threads.
Actually, reading and parsing is the bottleneck.
So, we parallelize the parsing.

The current best practice for parallelizing in python
is using a [map with async thread pools](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.Executor.map).
First, instead of calling `db.write` itself, the
`add_to_db` function should just take a directory name as input
and return a list of `[(out, key_value_pairs)]` that it would have written.
Now, the main loop can iterate over return values
from the map and write all those to the database.

```
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=8) as executor:
    for out, key_value_pairs in executor.map(parse_structures, \
                                             list_of_directories):
        db.write(out, key_value_pairs)
```

## Data Validation

Finally, a word about data validation.

It's important to understand the accuracy of your
VASP calculations.  Were these preliminary computations
used to map out a structural space with low energetic
accuracy, or a high-accuracy scan of the potential energy
surface during a rearrangement or chemical reaction?
Could the choice of pseudopotentials and energy cutoff
have created artifacts in the energy surface?

Georg Kresse has written a very nice
[presentation on this](http://wolf.ifj.edu.pl/workshop/work2004/files/talks/kresse_vasp_phonon.pdf).

[IBRION]: https://www.vasp.at/wiki/index.php/IBRION

# Authors

[David M. Rogers](https://www.olcf.ornl.gov/directory/staff-member/david-rogers/)
is a Computational Scientist in the Advanced Computing for Chemistry and Materials group, where he works to develop mathematical and computational theory jointly with methods for multiscale modeling using HPC. He obtained his Ph.D. in Physical Chemistry from University of Cincinnati in 2009 on the topic
of applying Bayes' theorem to the free energy problem with applications to multiscale modeling of fluids and interface chemistry.

This work was sponsored by the Laboratory Directed Research and Development Program of Oak Ridge National Laboratory (ORNL), managed by UT-Battelle, LLC, for the U.S. Department of Energy. ORNL is managed by UT-Battelle, LLC, for the DOE under Contract No. DE-AC05-00OR22725.
