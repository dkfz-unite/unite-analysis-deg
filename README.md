# Differential Gene Expression Analysis (DESeq2) Service

## General
DESeq2 analysis application wrapped with web API.

## Configuration
To configure the application, change environment variables as required in [commands](https://github.com/dkfz-unite/unite-commands/blob/main/README.md#configuration) web service:
- `UNITE_COMMAND` - command to run the analysis package (`Rscript`).
- `UNITE_COMMAND_ARGUMENS` - command arguments (`run.R {data}/{proc}_data.tsv {data}/{proc}_metadata.tsv {data}/{proc}_results.tsv`).
- `UNITE_SOURCE_PATH` - location of the source code in docker container (`/src`).
- `UNITE_DATA_PATH` - location of the data in docker container (`/mnt/analysis`).
- `UNITE_LIMIT` - maximum number of concurrent jobs (`1` - process is heavy and uses a lot of CPU).

## Installation

### Docker Compose
The easiest way to install the application is to use docker-compose:
- Environment configuration and installation scripts: https://github.com/dkfz-unite/unite-environment
- DESeq2 analysis service configuration and installation scripts: https://github.com/dkfz-unite/unite-environment/tree/main/applications/unite-analysis-deg

### Docker
[Dockerfile](Dockerfile) is used to build an image of the application.
To build an image run the following command:
```
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
-e UNITE_COMMAND=Rscript \
-v ./data:/mnt/analysis:rw \
-d \
unite.analysis.deg:latest
```

## Usage
- Place the data files `{proc}_data.tsv` and `{proc}_metadata.tsv` in the `./data` directory on the host machine.
- Send a POST request to the `localhost:5300/api/run?key=[key]` endpoint, where `[key]` is the process key.
- Analysis will run the command `Rscript` with the arguments `run.R {data}/{proc}_data.tsv {data}/{proc}_metadata.tsv {data}/{proc}_results.tsv` where `{proc}` is the process key.
  - All entries of `{data}` will be replaced with the path to the data location in docker container (In the example `./data` on the host machine will be mounted to `/mnt/analysis` in container).
  - All entries of `{proc}` will be replaced with the process key.
- Analysis will try to find the files `{proc}_data.tsv` and `{proc}_metadata.tsv` in the data location and use them as input.
- Analysis will save the results to the file `{proc}_results.tsv` in the data location.
