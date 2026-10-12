# Differential Gene Expression Analysis (DESeq2) Service

Shared architecture, development workflows, and agent instructions:
[UNITE project documentation](https://github.com/dkfz-unite/.github/blob/main/README.md).

## General
Bulk RNA differential gene expression analysis with [DESeq2](https://bioconductor.org/packages/DESeq2), wrapped with web API.

The R script ([src/run/run.R](src/run/run.R)) is run by the [commands](https://github.com/dkfz-unite/unite-commands) web service, which runs a configured command for each request. The service is used by the [analysis service](https://github.com/dkfz-unite/unite-analysis) for [DEG analysis](https://github.com/dkfz-unite/unite-analysis/blob/main/Docs/api-analysis-deg.md): the analysis service writes the input files to the shared analysis data directory and calls the service with the analysis id as process key.

The script:
1. Reads the read counts and the sample conditions.
2. Runs DESeq2 with design `~ condition` (`DESeqDataSetFromMatrix`, `DESeq`, `results`).
3. Writes the results without rows with missing values.

R packages: `DESeq2`, expected from the base image `ghcr.io/dkfz-unite/docker-rdev-deseq2` ([src/install/install.R](src/install/install.R) installs nothing; its install line is commented out).

## Configuration
To configure the application, change environment variables as required in [commands](https://github.com/dkfz-unite/unite-commands/blob/main/README.md#configuration) web service. The [Dockerfile](Dockerfile) sets the following values:
- `UNITE_COMMAND` - command to run the analysis script (`Rscript`).
- `UNITE_COMMAND_ARGUMENTS` - command arguments (`run.R {data}/{proc}/data.tsv {data}/{proc}/metadata.tsv {data}/{proc}/results.tsv`).
  - `{data}` is replaced with `UNITE_DATA_PATH`, `{proc}` with the process key (analysis id).
- `UNITE_SOURCE_PATH` - location of the script in docker container, the working directory of the command (`/src`).
- `UNITE_DATA_PATH` - location of the data in docker container (`/mnt/data`).
- `UNITE_PROCESS_LIMIT` - maximum number of concurrent processes (`1` - process is heavy and uses a lot of CPU). If the limit is reached, the service returns `501`.

## Usage
- Place the input files to the `{proc}` subdirectory of the data directory (`/mnt/data/{proc}` in the container).
- Send a `POST` request to `/api/run?key={proc}`.
- The service runs `Rscript run.R /mnt/data/{proc}/data.tsv /mnt/data/{proc}/metadata.tsv /mnt/data/{proc}/results.tsv` and returns `200` with the script output, or `500` if the script fails.

### Input files
`data.tsv` - raw read counts. The first column `gene_id` is the gene identifier, the other columns are samples:
```tsv
gene_id	Donors_150	Donors_206	Donors_210	Donors_218
ENSG00000227232	956	1482	602	2623
ENSG00000237683	1485	109	797	159
```

`metadata.tsv` - sample conditions. The first column is the sample name (a column name of `data.tsv`), the second column `condition` is the condition label:
```tsv
sample_id	condition
Donors_150	8b4e6a2c-2b4b-4e3e-9c6d-5a4f1d3b9e0f
Donors_206	8b4e6a2c-2b4b-4e3e-9c6d-5a4f1d3b9e0f
Donors_210	2e6ee2ff-d96d-422d-8a0d-a49a412baa5c
Donors_218	2e6ee2ff-d96d-422d-8a0d-a49a412baa5c
```

The analysis service names samples `{Domain}_{id}` and uses the dataset id as condition.
Conditions are R factor levels in default (alphabetical) order; DESeq2 uses the first level as the reference, so `log2FoldChange` is the log2 fold change of the last level over the first one.

### Output files
`results.tsv` - DESeq2 results: `ID` (gene identifier), `baseMean`, `log2FoldChange`, `lfcSE`, `stat`, `pvalue`, `padj`.

The analysis service maps `ID`, `log2FoldChange` and `padj` to genes and writes `results_final.tsv`.

## Installation

### Docker Compose
The easiest way to install the application is to use docker-compose:
- [Environment configuration and installation scripts](https://github.com/dkfz-unite/unite-env)
- [DEG analysis service configuration and installation scripts](https://github.com/dkfz-unite/unite-env/tree/main/applications/unite-analysis-deg)

### Docker
The image of the service is available in our [registry](https://github.com/dkfz-unite/unite-analysis-deg/pkgs/container/unite-analysis-deg).

The image is built by the [docker-images](.github/workflows/docker-images.yml) workflow on pushes to `main`: it checks out [unite-commands](https://github.com/dkfz-unite/unite-commands), publishes it with `publish-linux-x64.sh` and `publish-linux-arm64.sh`, copies the result (`commands` executable) to `./app`, builds amd64 and arm64 images with the [Dockerfile](Dockerfile) and pushes them under one `latest` manifest.

To build an image locally, publish unite-commands for the target platform to `./app` first (the `app` directory in the repository is replaced by the workflow and is not a current build):
```bash
# in the unite-commands repository
sh publish-linux-x64.sh # or publish-linux-arm64.sh

# in this repository
rm -rf app
cp -r ../unite-commands/publish/linux-x64 app
chmod a+rx app/commands
docker build -t unite.analysis.deg:latest .
```

All application components should run in the same docker network.
To create common docker network if not yet available run the following command:
```bash
docker network create unite
```

To run application in docker run the following command:
```bash
docker run \
--name unite.analysis.deg \
--restart unless-stopped \
--net unite \
--net-alias deg.analysis.unite.net \
-p 127.0.0.1:5300:80 \
-e ASPNETCORE_ENVIRONMENT=Release \
-v unite.analysis.data:/mnt/data:rw \
-d \
unite.analysis.deg:latest
```

The volume `unite.analysis.data` is the analysis data directory shared with the analysis service. For standalone use, a host directory can be mounted instead (e.g. `-v ./data:/mnt/data:rw`).
