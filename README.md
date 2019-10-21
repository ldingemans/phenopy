[![python-version](https://img.shields.io/badge/python-3.6+-blue.svg)](https://www.python.org/downloads/release/python-360/)
[![github-actions](https://github.com/GeneDx/phenopy/workflows/Python%20package/badge.svg)](https://github.com/GeneDx/phenopy/actions)
[![codecov](https://codecov.io/gh/GeneDx/phenopy/branch/develop/graph/badge.svg)](https://codecov.io/gh/GeneDx/phenopy)

# phenopy
`phenopy` is a Python package to perform phenotype similarity scoring by semantic similarity. `phenopy` is a
lightweight but highly optimized command line tool and library to efficiently perform semantic similarity scoring on
generic entities with phenotype annotations from the [Human Phenotype Ontology (HPO)](https://hpo.jax.org/app/).

![Phenotype Similarity Clustering](https://raw.githubusercontent.com/GeneDx/phenopy/develop/notebooks/output/cluster_three_diseases.png)

## Installation
Install using pip:
```bash
pip install phenopy
```

Install from GitHub:
```bash
git clone https://github.com/GeneDx/phenopy.git
cd phenopy
python setup.py install
```

## Command Line Usage
### Initial setup
phenopy is designed to run with minimal setup from the user, to run phenopy with default parameters (recommended), skip ahead
to the [Commands overview](#Commands-overview).  

This section provides details about where phenopy stores data resources and config files. The following occurs when
you run phenopy for the first time.
 1. phenopy creates a `.phenopy/` directory in your home folder and downloads external resources from HPO into the
  `$HOME/.phenopy/data/` directory.
 2. phenopy stores a binary version of the HPO as a [networkx](https://networkx.github.io/documentation/stable/reference/classes/multidigraph.html)
 graph object here: `$HOME/.phenopy/data/hpo_network.pickle`.
 3. phenopy creates a `$HOME/.phenopy/phenopy.ini` config file where users can set variables for phenopy to use
 at runtime.

### Commands overview
`phenopy` is primarily used as a command line tool. An entity, as described here, is presented as a sample, gene, or
disease, but could be any concept that warrants annotation of phenotype terms.

1. Score similarity of an entity defined by the HPO terms from an input file against all the OMIM diseases in
`.phenopy/data/phenotype.hpoa`. We provide a test input file in the repo.
    ```bash
    phenopy score tests/data/test.score.txt
    ```
    Output:
    ```
    #query	omim_id	score
    SAMPLE	210100	0.003.172551296022093e-05
    SAMPLE	163600	0.04450039556373293
    SAMPLE	615763	0.05497737732718229
    ...
    ```

2. Score similarity of an entity defined by the HPO terms from an input file against a custom list of entities with HPO annotations, referred to as the `--records-file`.
    ```bash
    phenopy score tests/data/test.score.txt --records-file tests/data/test.score-product.txt
    ```
    Output:
    ```
    #query  omim_id score
    SAMPLE  210100  3e-05
    SAMPLE  163600  0.0445
    SAMPLE  615763  0.05498
    ...
    ```

3. Score pairwise similarity of entities defined in the `--records-file`.

    ```bash
    phenopy score-product tests/data/test.score-product.txt --threads 4
    ```
    Output:
    ```
    118200	118200	0.7692
    118200	118300	0.5345
    118200	300905	0.2647
    ...
    ```
4. Score age-adjusted pairwise similarity of entities defined in the `--records-file`, 
    using phenotype mean age and standard deviation defined in the `--pheno_ages_file`,
    select best-match weighted average as the scoring aggregation method `--agg_score BMWA`.

    ```bash
    phenopy score-product tests/data/test.score-product-age.txt --pheno_ages_file tests/data/phenotype_age.tsv --agg_score BMWA --threads 4
    ```
    Output:
    ```
    118200  118200  0.2288
    118210  118210  0.2038
    118211  118211  0.2038
    118200  118210  0.1636
    ...
    ```
    
    The phenotype age file contains hpo-id, mean, sd as tab separated text as follows
    
    |  |  | |
    |------------|------|-----|
    | HP:0001251 | 6.0  | 3.0 |
    | HP:0001263 | 1.0  | 1.0 |
    | HP:0001290 | 1.0  | 1.0 |
    | HP:0004322 | 10.0 | 3.0 |
    | HP:0001249 | 6.0  | 3.0 |

  If no phenotype ages file is provided `--agg_score BMWA` can be selected to use default, open access literature-derived phenotype ages (~ 1,400 age weighted phenotypes).  
   ```bash
    phenopy score-product tests/data/test.score-product-age.txt  --agg_score BMWA --threads 4
   ```


## Parameters
For a full list of command arguments use `phenopy [subcommand] --help`:
```bash
phenopy score --help
```
Output:
```
    --records_file=RECORDS_FILE
        One record per line, tab delimited. First column record unique identifier, second column pipe separated list of HPO identifier (HP:0000001).
    --query_name=QUERY_NAME
        Unique identifier for the query file.
    --obo_file=OBO_FILE
        OBO file from https://hpo.jax.org/app/download/ontology.
    --disease_to_phenotype_file=DISEASE_TO_PHENOTYPE_FILE
        Disease to phenotype annoations from http://compbio.charite.de/jenkins/job/hpo.annotations.2018/
    --threads=THREADS
        Number of parallel process to use.
    --agg_score=AGG_SCORE
        The aggregation method to use for summarizing the similarity matrix between two term sets Must be one of {'BMA', 'maximum'}
    --no_parents=NO_PARENTS
        If provided, scoring is done by only using the most informative nodes. All parent nodes are removed.
    --custom_annotations_file=CUSTOM_ANNOTATIONS_FILE
        A custom entity-to-phenotype annotation file in the same format as tests/data/test.score-product.txt
    --output_file=OUTPUT_FILE
        filepath where to store the results.
```
## Library Usage
The `phenopy` library can be used as a `Python` module, allowing more control for advanced users.   

```python
import os
from phenopy import config
from phenopy.obo import restore
from phenopy.score import Scorer

network_file = os.path.join(config.data_directory, 'hpo_network.pickle')

hpo = restore(network_file)
scorer = Scorer(hpo)

terms_a = ['HP:0001882', 'HP:0011839']
terms_b = ['HP:0001263', 'HP:0000252']

print(scorer.score(terms_a, terms_b))
```
Output:
```
0.0005
```

Another example is to use the library to prune parent phenotypes from the `phenotype.hpoa`
```python
import os
from phenopy import config
from phenopy.obo import restore
from phenopy.util import export_phenotype_hpoa_with_no_parents


network_file = os.path.join(config.data_directory, 'hpo_network.pickle')
disease_to_phenotype_file = os.path.join(config.data_directory, 'phenotype.hpoa.txt')
disease_to_phenotype_no_parents_file = os.path.join(config.data_directory, 'phenotype.noparents.hpoa')

hpo = restore(network_file)
export_phenotype_hpoa_with_no_parents(disease_to_phenotype_file, disease_to_phenotype_no_parents_file, hpo)
```

### Config
While we recommend using the default settings for most users, the config file *can be* modified: `$HOME/.phenopy/phenopy.ini`.

**IMPORTANT NOTE:  
If the config variable `hpo_network_file` is defined, phenopy will try to load this stored version of the HPO and ignore
the following command-line arguments: `obo_file` and `custom_annotations_file`.**

To run phenopy with different `obo_file` or `custom_annotations_file`:
Rename or move the HPO network file: `mv $HOME/.phenopy/data/hpo_network.pickle $HOME/.phenopy/data/hpo_network.old.pickle`

To run phenopy with a previously stored version of the HPO network, simply set
`hpo_network_file = /path/to/hpo_network.pickle`.  

## Contributing
We welcome contributions from the community. Please follow these steps to setup a local development environment.  
```bash
pipenv install --dev
```

To run tests locally:
```bash
pipenv shell
coverage run --source=. -m unittest discover --start-directory tests/
coverage report -m
```  

## References
The underlying algorithm which determines the semantic similarity for any two HPO terms is based on an implementation of HRSS, [published here](https://www.ncbi.nlm.nih.gov/pubmed/23741529).
