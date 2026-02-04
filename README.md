# OpenCitations OCI Service

The **Open Citation Identifier (OCI)** is a globally unique persistent identifier for bibliographic citations, created and maintained by [OpenCitations](https://opencitations.net). This service provides a resolution mechanism that takes an OCI and returns information about that citation.

## Formal Definition

A formal description and definition of an OCI is given in:

> Silvio Peroni, David Shotton (2019). *Open Citation Identifier: Definition*. Figshare.
> [https://doi.org/10.6084/m9.figshare.7127816](https://doi.org/10.6084/m9.figshare.7127816)

## Supplier Prefixes

OCIs have been created for open citations within various bibliographic databases. The prefix indicates the data source:

| Prefix | Supplier | Identifier Type | Example |
|--------|----------|-----------------|---------|
| `010` | [Wikidata](https://www.wikidata.org) | Wikidata Identifier (QID) | `oci:01027931310-01022252312` |
| `020` | [Crossref](https://crossref.org) | Digital Object Identifier (DOI) | `oci:02001010806360107050663080702026306630509-02001010806360107050663080702026305630301` |
| `040` | [Dryad](https://datadryad.org/) | Digital Object Identifier (DOI) | `oci:0400500060136132734101337001215332523320931-040010009033616142314291812283601030037013701090905` |
| `06[1-9 digits]0` | [OpenCitations](https://w3id.org/oc) | OpenCitations Meta Identifier (OMID) | `oci:06101801781-06180334099` |

# Configuration

## Environment Variables

The service requires the following environment variables. These values take precedence over the ones defined in `conf.json`:

- `BASE_URL`: Base URL for the service
- `SPARQL_ENDPOINT_INDEX`: URL for the internal Index SPARQL endpoint
- `SPARQL_ENDPOINT_META`: URL for the internal Meta SPARQL endpoint
- `LOG_DIR`: Directory path where log files will be stored
- `SYNC_ENABLED`: Enable/disable static files synchronization (default: false)
- `INDEX_BASE_URL`: Base URL used to construct redirect links for resources served by the application

For instance:

```env
# API Configuration
BASE_URL=api.opencitations.net
LOG_DIR=/home/dir/log/
SPARQL_ENDPOINT_INDEX=http://qlever-service.default.svc.cluster.local:7011  
SPARQL_ENDPOINT_META=http://virtuoso-service.default.svc.cluster.local:8890/sparql
SYNC_ENABLED=true
INDEX_BASE_URL=https://w3id.org/oc
```

> **Note**: When running with Docker, environment variables always override the corresponding values in `conf.json`. If an environment variable is not set, the application will fall back to the values defined in `conf.json`.

## Static Files Synchronization

The application can synchronize static files from a GitHub repository. This configuration is managed in `conf.json`:

```json
{
  "oc_services_templates": "https://github.com/opencitations/oc_services_templates",
  "sync": {
    "folders": [
      "static",
      "html-template/common"
    ],
    "files": [
      "test.txt"
    ]
  }
}
```

- `oc_services_templates`: The GitHub repository URL to sync files from
- `sync.folders`: List of folders to synchronize
- `sync.files`: List of individual files to synchronize

When static sync is enabled (via `--sync-static` or `SYNC_ENABLED=true`), the application will:
1. Clone the specified repository
2. Copy the specified folders and files
3. Keep the local static files up to date

> **Note**: Make sure the specified folders and files exist in the source repository.

# Running Options

## Local Development

For local development and testing, the application uses the built-in web.py HTTP server:

Examples:
```bash
# Run with default settings
python3 oci_oc.py

# Run with static sync enabled
python3 oci_oc.py --sync-static

# Run on custom port
python3 oci_oc.py --port 8085

# Run with both options
python3 oci_oc.py --sync-static --port 8085
```

The application supports the following command line arguments:

- `--sync-static`: Synchronize static files at startup and enable periodic sync (every 30 minutes)
- `--port PORT`: Specify the port to run the application on (default: 8080)

## Production Deployment (Docker)

When running in Docker/Kubernetes, the application uses **Gunicorn** as the WSGI HTTP server for better performance and concurrency handling:


You can change these variables in the Dockerfile:
- **Server**: Gunicorn with gevent workers
- **Workers**: 2 concurrent worker processes
- **Worker Type**: gevent (async) for handling thousands of simultaneous requests
- **Timeout**: 1200 seconds (to handle long-running SPARQL queries)
- **Connections per worker**: 800 simultaneous connections


The Docker container automatically uses Gunicorn and is configured with static sync enabled by default.

> **Note**: The application code automatically detects the execution environment. When run with `python3 oci_oc.py`, it uses the built-in web.py server. When run with Gunicorn (as in Docker), it uses the WSGI interface.

You can customize the Gunicorn server configuration by modifying the `gunicorn.conf.py` file.

## Dockerfile

You can change these variables in the Dockerfile:

```dockerfile
# Base image: Python slim for a lightweight container
FROM python:3.11-slim

# Define environment variables with default values
# These can be overridden during container runtime
ENV BASE_URL="oci.opencitations.net" \
    LOG_DIR="/mnt/log_dir/oc_oci" \
    SPARQL_ENDPOINT_INDEX="http://qlever-service.default.svc.cluster.local:7011" \
    SPARQL_ENDPOINT_META="http://virtuoso-service.default.svc.cluster.local:8890/sparql" \
    SYNC_ENABLED="true" \
    INDEX_BASE_URL="https://w3id.org/oc" \
    USE_INTERNAL_OC_ENDPOINT="true" \
    API_INTERNAL_ENDPOINT="http://oc-api-service.default.svc.cluster.local" \
    SPARQL_INTERNAL_ENDPOINT="http://oc-sparql-service.default.svc.cluster.local"


# Ensure Python output is unbuffered
ENV PYTHONUNBUFFERED=1
# Install system dependencies required for Python package compilation
# We clean up apt cache after installation to reduce image size
RUN apt-get update && \
    apt-get install -y \
    git \
    python3-dev \
    build-essential

# Set the working directory for our application
WORKDIR /website

# Clone the specific branch (api) from the repository
# The dot at the end means clone into current directory
RUN git clone --single-branch --branch main https://github.com/opencitations/oc_oci .

# Install Python dependencies from requirements.txt
RUN pip install -r requirements.txt

# Expose the port that our service will listen on
EXPOSE 8080

# Start the application with gunicorn for production
CMD ["gunicorn", "-c", "gunicorn.conf.py", "oci_oc:application"]
```