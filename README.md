# Authenticating to Azure Services in a .NET Core Container

## Quick start

Run the `Docker .NET Core Launch (Development)` Configuration via Visual Studio Code.

## Setting up local development with .NET Core Container communicating with Azure

Azure pairs well with local development of .NET apps via Visual Studio and Visual Studio Code,
however containers introduce several wrinkles. Utilizng Azure resources in local development depends
on being authenticated to Azure, whether that be via logging in via Visual Studio or via the Azure CLI.
Local development of .NET core apps in a container thus requires the following to mimic development on your local machine:
- Azure CLI
- Logging on to the Azure CLI

The first issue can be solved in the Dockerfile with the following:
```
FROM base AS development
RUN apt-get update && apt-get install -y curl && curl -sL https://aka.ms/InstallAzureCLIDeb | bash
```
The above code first installs `curl`, then the Azure CLI via `curl`.

Next, the container needs to be able to 'log in' to the Azure CLI. This can't be done at build time of the Dockerfile,
and would be cumbersome to manually do at runtime. We can get around this by leveraging where the Azure CLI stores credentials
when we run `az login` on our host machine: in the `/.azure` subdirectory. By mounting this directory as a bind mount, we avoid
having to run any additional instructions at runtime to authenticate to Azure: as long as we our logged in on our host machine, our
local container will have access as well. Mounting the directory is accomplished in `tasks.json` with:
```
        {
            "type": "docker-run",
            "label": "docker-run: development",
            "dependsOn": [
                "docker-build: development"
            ],
            "dockerRun": {
                "containerName":"webapi-development",
                "ports": [{"containerPort": 5000,"hostPort": 5000}],
                "image": "webapi:development",
                "volumes": [{"localPath": "${env:HOME}/.azure", "containerPath": "/root/.azure","permissions": "rw"}]
            },
            "netCore": {
                "appProject": "${workspaceFolder}/webapi.csproj",
                "enableDebugging": true
            }
        },
```