# DBT-Final-Project-PilotDB-2
An enhanced version of PilotDB by: https://github.com/uiuc-kang-lab/PilotDB

We aim to solve some of the issues with the original PilotDB implementation as acknowledged by the original authors. More on these improvements can be found in our (fixes readme)[our-fixes/our-fixes.md].

Our adjusted version of PilotDB can be installed in the same way as the original as per the author's readme: (PilotDB-original readme)[PilotDB-original/README.md].

All of our fixes were tested via the TPC-H dataset, specifically this [version](https://github.com/aleaugustoplus/tpch-data). 

You can learn more about this dataset on the official TPC-H [website](https://www.tpc.org/tpch/) but in-short TPC-H is an an industry-standard decision support benchmark evaluates the performance of Online Analytical Processing (OLAP) and data warehousing systems by running complex queries against large volumes of data.

To assists with setting up TPC-H we have also added the following files:
1. [setup_tpch.sql](setup_tpch.sql) - which sets up the tpc-h database table schema.
2. [gen_tpch.py](gen_tpch.py) - which generates the tpc-h data.

All of benchmarking results and analysis for our improvements to PilotDB utilize the TPC-H dataset and can be found here: [benchmarking analysis](tpch-benchmark-testing-results/benchmark_analysis).


