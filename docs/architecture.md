# Architecture

## Application runtime

```mermaid
flowchart LR
    user[Browser / HTTP client]
    subgraph azure["Microsoft Azure"]
        subgraph appservice["Linux App Service - pythonwebapp98600 (Production)"]
            ingress[App Service HTTP ingress]
            gunicorn[Gunicorn web server]
            flask[Flask application - app.py]
            pages[Dashboard template - templates/index.html]
            assets[Static CSS and JavaScript]
            endpoints["Routes: /, /health, /api/info"]
            ingress --> gunicorn
            gunicorn --> flask
            flask --> endpoints
            endpoints --> pages
            pages --> assets
        end
    end
    user -->|"HTTPS requests"| ingress
    endpoints -->|"HTML / JSON responses"| ingress
```

The Flask app serves the dashboard and its static assets, plus health and
application metadata endpoints. The application has no database or other
external runtime service configured.

## Build and deployment

```mermaid
flowchart LR
    source["Source changes on main<br/>or manual dispatch"]
    subgraph actions["GitHub Actions - Ubuntu runners"]
        build["Build job<br/>Checkout repository<br/>Set up Python 3.14<br/>Install requirements.txt"]
        artifact["python-app artifact<br/>(without antenv/)"]
        deploy["Deploy job<br/>Download artifact"]
        oidc["Azure login<br/>GitHub OIDC"]
        action["azure/webapps-deploy@v3"]
        build -->|"upload"| artifact
        artifact -->|"needs: build"| deploy
        deploy --> oidc
        oidc --> action
    end
    azure["Azure App Service<br/>pythonwebapp98600<br/>Production slot"]
    source --> build
    action -->|"deploy application package"| azure
```

The deployment workflow is
`.github/workflows/main_pythonwebapp98600.yml`. It uploads the repository
contents as an artifact, excluding the build virtual environment. Azure login
uses OIDC credentials stored as GitHub Actions secrets. The App Service
deployment build behavior depends on the `SCM_DO_BUILD_DURING_DEPLOYMENT`
setting; the workflow comments describe Oryx as the default when that setting
is enabled.
