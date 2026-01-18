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

* The `Dockerfile.dev` files enclosed within the `frontend` & `backend` folders, provide the base infrastructure
* Then, the `devcontainer.json` files supplement the aformentioned files with developer-specific tooling (e.g. VSCode extensions)

Note that although this setup is highly simplified (i.e. our backend and frontend consists of two files
collectively), it is still robust enough to use on any serious development initiative as we are merely
providing the *canvas*, not the *implementation* (e.g. `React` + `FastAPI` being one approach).

## Notes

* Note that `npm` is a *package manager* service (whereas `npx` is a tool to *execute* packages)
* 'CORS' is what is known as 'Cross-Origin Resource Sharing' (or 'CORS' for short) and it is a browser
security rule,
    * Browsers consider two URLs to be from 'different origins' (i.e. representing different entities)
    if either the protocol, domain or port differ
    * In our case, `localhost:8000` and `localhost:8080` are from different origins since the port number differs
    * The browser acts as a 'security officer' of sorts, by default, by stipulating that any resources that are requested from 'outside origins' be
    stamped with an explicit header (`Access-Control-Allow-Origin`) that permits contact with said origins