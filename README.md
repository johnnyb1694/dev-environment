# Containerised Development Environments - Template :national_park:

## Background :world_map:

This repository is intended to act as a template for other developers who may wish to use
the VSCode [development containers extension](https://code.visualstudio.com/docs/devcontainers/containers).

One of the core benefits of this approach is ensuring the reproducibility of your development
environment which becomes crucial for ensuring efficiency across a multidisciplinary team.

### Structure :hammer_and_wrench:

```
.
├── backend
│   ├── Dockerfile.dev
│   ├── Dockerfile.prod
│   ├── main.py
│   ├── pyproject.toml
│   ├── README.md
│   ├── serve.sh
│   └── uv.lock
├── compose.dev.yml
├── db
│   └── secrets
│       ├── password-admin.txt
│       └── password.txt
├── frontend
│   ├── Dockerfile.dev
│   ├── Dockerfile.prod
│   ├── index.html
│   ├── package-lock.json
│   ├── package.json
│   └── serve.sh
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

### Development versus Production

Below is a copy of `./backend/Dockerfile.dev` which we will now discuss for illustrative purposes,

```docker
ARG PYTHON_VERSION=3.14

FROM python:${PYTHON_VERSION}

# Installs relevant development dependencies
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y git

# Expose '8000' inside the compose environment (which we will then map to in our compose file)
EXPOSE 8000

# NB: when 'overrideCommand = true' in devcontainer.json, this will be ignored
CMD ["tail", "-f", "/dev/null"]
```

So, the purpose of `Dockerfile.dev` is to provision a working software environment for developing
code with Python. In the main, this involves installing Python and git, which are listed above.

Note that we only include `EXPOSE 8000` so that access to port `8000` is permitted from other
containers on the same network (this is important since we will be using port `8000` to expose
our FastAPI service in development).

Also, we do not install Python dependencies until the image has been instantiated - installation of 
dependencies is handled in the `postCreateCommand` which can be found at `.devcontainer/backend/devcontainer.json`.
You could include this in your image but I've found the above to work well for my purposes.

By contrast, let's look at the production version hosted at `./backend/Dockerfile.prod`,

```docker
ARG PYTHON_VERSION=3.14.2-slim

FROM python:${PYTHON_VERSION}

# Installs relevant development dependencies
RUN apt-get update && \
    apt-get upgrade -y

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
RUN python -m pip install --no-cache-dir -r requirements.txt

# Copy the remaining source code
COPY . .

# Switch to safer 'run as' user
USER backenduser

# Expose '8000' inside the compose environment (which we will then map to in our compose file)
EXPOSE 8000

CMD ["/bin/sh", "serve.sh"]
```

There are several observable differences here:

* We opt for the 'slim' version of Python to save on space
* We provision a non-privileged user `backenduser` to run the API service mitigating security risks
* We disable the `pip` caching feature to save on space

The same idea presented above applies to the frontend configuration.

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

### Using `uv`

`uv` is a lightning-fast package management system implemented in Rust.

It is used in this project to configure the `./backend` service. 

#### Key Commands

1. To initialise a project, you type,

    ```sh
    uv init
    ```

    This will create a 'project' by default but if you want to create a 'Python package' or general 
    'System library', arguments can be provided to do that (`--package {NAME}` and `--lib {NAME}` respectively)

    As a general rule,

    * *Projects* are more suitable for web servers, scripts and CLIs
    * *Packages* are most suitable when building distributable *Python* packages (e.g. a CLI that will
    need to be published to 'PyPI' or some kind of 'Nexus' proxy). In this instance, a `src` directory
    is created and a 'package' folder is nested within for further development. In addition, a 'build
    system' will be defined so that the project can be installed into the local environment (this defaults
    to `uv_build`). Remember that when we say 'build' we typically are referring to the process of creating
    source archives called 'sdists' or 'wheels' that may contain binary modules from a compiled language
    (e.g. C++). Remember that the Python interpreter *does* compile source code to byte code at runtime,
    as much as necessary (this is why `__pycache__` folders exist). Build systems typically deal with 
    compilation in this sense as well
    * *Libraries* are very similar to packages but are not necessarily Python-specific

2. To run Python scripts (e.g. `main.py`) in the context of the local `.venv/` folder you type,

    ```sh
    uv run main.py
    ```

    Note that whenever you `run` something (as above), the project is 'locked' and 'synced' before
    invoking the requested command, which ensures that the prject is always up-to-date.

    To disable some of these actions, consider the following alternatives,

    ```sh
    # Run a command *and* disable automatic logging of dependencies to lock file
    # i.e. this will not try to update the lock file (but it will match `uv.lock` and `pyproject.toml`)
    uv run --locked main.py

    # Run a command *and* use the lock file without checking if it is up-to-date
    # i.e. this will not try to update the lock file (*and* it will not bother matching `uv.lock` and `pyproject.toml`)
    uv run --frozen main.py

    # Run a command *and* do not check whether the virtual environment is up-to-date
    # i.e. this will run with whatever is stored in `.venv/`
    uv run --no-sync main.py
    ```

3. To add a new dependency 'foo' (i.e. install the dependency into `.venv/` and register it to `pyproject.toml` & `uv.lock`),

    ```sh
    uv add foo
    ```
    
    When this action is administered, a new entry will be added to the `project.dependencies` field
    of `pyproject.toml`

    If you want to simply add a 'development' dependency (e.g. `pytest`), you'd type,

    ```sh
    uv add --dev pytest
    ``` 

    If you want to simply add an 'optional' dependency (e.g. `excel` is treated as an extra for `pandas`),

    ```sh
    uv add openpyxl --optional excel
    ```

    This will create a named list entry in `project.optional-dependencies` with an entry for `openpyxl`
    under the name `excel`.

    You can add dependencies from a different source to 'PyPI' (e.g. from GitHub) with the following syntax,

    ```sh
    uv add "httpx @ git+https://github.com/encode/httpx" --branch main
    ```

    If this action is performed an additional entry will be appended to `project.tool.uv.sources`. The sources
    supported by `uv` are numerous:

    * Index: a package resolved from an index such as PyPI (the default)
    * Git: a git repository
    * URL: a remote wheel or source distribution
    * Path: a local wheel, source distribution or project directory
    * Workspace: a member of the current workspace

4. To remove an existing dependency 'foo', simply type,

    ```sh
    uv remove foo
    ```

5. To sync a new project with an existing lock file (e.g. you're a new developer),

    ```sh
    uv sync --locked # NB: do not update `uv.lock` by default (but *do* sync the existing lock file with `pyproject.toml`)
    ```

6. To upgrade all of the Python packages within a given project, you can run,

    ```sh
    uv lock --upgrade
    ```

#### Locking & Syncing

We need to first distinguish between 'lock' and 'sync':

* Locking: is the process of resolving your project's dependencies into a lockfile - if you want to 
disable automatic locking, you can always use the `--locked` option on relevant commands
* Syncing: is the process of installing a subset of packages from the lockfile into the project environment

Note that a lock file is considered 'out of date' if any of the following are true,

* You manually add a dependency to `pyproject.toml`
* You change the version constraints for a dependency in such a way that the currently locked
version is now excluded (e.g. imagine you have `pandas==3.4.1` but you change the condition to `pandas>3.4.1`)

You can always check whether the lock file is up-to-date with the command,

```sh
uv lock --check
```

#### Key Files

When you initialise a repository with `uv`, it will create a `.venv/` folder which acts as a 
virtual environment for your project (this will be automatically excluded from git via `.gitignore`).

Important files that are created & manager by `uv` include,

* `pyproject.toml`: this encodes metadata about your Python application - it specifies the 'broad'
requirements of your project. Please note that the `project.dependencies` field in this file is used
when uploading to PyPI or building a wheel (i.e. it is framed from the client's perspective rather
than the developer's perspective which is what `uv.lock` entails)
* `uv.lock`: this is more detailed than `pyproject.toml` and contains information on *where* each 
package can be sourced from (i.e. the source `url` for the distributable bundle) along with the
full dependency tree associated with each package (i.e. the dependencies of your dependencies)
