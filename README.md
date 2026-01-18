# Containerised Development Environments - Template :national_park:

## Background :world_map:

This repository is intended to act as a template for other developers who may wish to use
the VSCode [development containers extension](https://code.visualstudio.com/docs/devcontainers/containers).

One of the core benefits of this approach is ensuring the reproducibility of your development
environment which becomes crucial for ensuring efficiency across a team.

## Structure :hammer_and_wrench:

```
.
├── .devcontainer # supplements base infrastructure with developer-specific tooling (e.g. VSCode extensions)
│   ├── backend
│   │   └── devcontainer.json
│   └── frontend
│       └── devcontainer.json
├── backend
│   ├── Dockerfile.dev # base infrastructure for backend development (e.g. `python`, `pip`, ...)
│   └── main.py
├── compose.dev.yml
├── frontend
│   ├── Dockerfile.dev # base infrastructure for frontend development (e.g. `http-server`, `npm`, ...)
│   └── index.html
└── README.md
```

The way this setup works is as follows,

* The `Dockerfile.dev` files enclosed within the `frontend` & `backend` folders, provide the base infrastructure (e.g.
it will provide a working version of the language 'Python' and other development tools like 'git'). This file is basically
your 'definition' of how you want the base environment to look, whether you are connected to the container or not
* Then, the `devcontainer.json` files supplement the aformentioned files with developer-specific tooling (e.g. VSCode extensions)
and custom 'startup' logic. This file is basically your 'definition' of how you want your development environment
to look *on top of* whatever is provided by your basic `Dockerfile`

Note that although this setup is highly simplified (i.e. our backend and frontend consists of two files
collectively), it is still robust enough to use on any serious development initiative as we are merely
providing the *canvas*, not the *implementation* (e.g. `React` + `FastAPI` being one approach).

### Further Explanation

Below is a copy of `./backend/Dockerfile.dev` which we will now discuss for illustrative purposes,

```docker
ARG PYTHON_VERSION=3.14.2-slim

FROM python:${PYTHON_VERSION}

# Installs relevant development dependencies
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y git

# Setup a working directory
WORKDIR /app

# Setup a default user to 'run as'
ARG UID=10001
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    backenduser

# Copy requirements file
COPY requirements.txt .

# Install packages
RUN python -m pip install -r requirements.txt

# Copy the remaining source code
COPY . .

# Switch to safer 'run as' user
USER backenduser

# Expose '8000' inside the compose environment (which we will then map to in our compose file)
EXPOSE 8000

# NB: when 'overrideCommand = true' in devcontainer.json, this will be ignored
CMD ["tail", "-f", "/dev/null"]
```

## Miscellaneous

### CORS

'CORS' is what is known as 'Cross-Origin Resource Sharing' (or 'CORS' for short) and it is a browser
security rule,

* Browsers consider two URLs to be from 'different origins' (i.e. representing different entities)
if either the protocol, domain or port differ
* In our case, `localhost:8000` and `localhost:8080` are from different origins since the port number differs
* The browser acts as a 'security officer' of sorts, by default, by stipulating that any resources that are requested from 'outside origins' be
stamped with an explicit header (`Access-Control-Allow-Origin`) that permits contact with said origins

Note that CORS is implemented in our `FastAPI` backend via the logic outlined in `./backend/main.py`.

Without it, there would be no way to access the backend from our frontend via the host's browser.