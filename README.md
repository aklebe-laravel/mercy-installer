# Mercy-Installer

Installs and updates a [Mercy Scaffold](https://github.com/aklebe-laravel/mercy-scaffold) application on a production
server. The application runs as a Docker image built with `./mercy build`; the installer pulls that image, configures it
from your project repository and keeps it up to date.

The installer contains no application code and no configuration of its own:

- **the image** brings the application, `compose.yaml` and the env templates (copied out of the image on every run, so
  they always match the release),
- **your project repository** (`project/`) brings the config (stage `prod`) and the business imports,
- **the installer** brings the `./mercy` CLI ([mercy-php-deployment](https://github.com/aklebe-laravel/mercy-php-deployment)).

## Contents

- [Requirements](#requirements)
- [Install](#install)
- [Update and rollback](#update-and-rollback)
- [Configuration](#configuration)
- [Commands](#commands)
- [Folder structure](#folder-structure)
- [Remove an installation](#remove-an-installation)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Requirements

| Requirement                       | Notes                                                                          |
|-----------------------------------|--------------------------------------------------------------------------------|
| Docker with the compose plugin    | your user in the group `docker`                                                |
| PHP 8.5 CLI, Composer 2.9         | the CLI runs on the host                                                       |
| `git`, `unzip`                    |                                                                                |
| access to your project repository | e.g. your SSH key added at GitHub / your git server                            |
| a registry token                  | only for a private image; GitHub `ghcr.io`: classic token with `read:packages` |

Tested on Ubuntu 24.04 LTS with 1 CPU core, 2 GiB memory and 32 GB disk.

## Install

```bash
# 1. the installer
mkdir -p ~/apps/my-product && cd ~/apps/my-product
git clone https://github.com/aklebe-laravel/mercy-installer.git .

# 2. your project repository and the secrets of this server
git clone <your-project-repository> project
cp project/config/secrets.example.yaml project/config/secrets.yaml   # if your project has one, then fill it in

# 3. install
composer install
./mercy install                  # with instances: ./mercy install --instance=<name>
```

The install checks PHP and the platform, logs in to the registry if needed (it asks for user and token), pulls the
image of `kernel.version`, copies the release files out of it, creates the env, starts the containers, generates the
app key, syncs Nginx Proxy Manager (if enabled) and runs migrations and module deployments. It ends with the status
of the containers.

Then load your project data:

```bash
docker compose exec mercy.app php artisan ldap:sync-all      # only with LDAP: users and groups first
docker compose exec mercy.app php artisan deploy-env:terraform-modules --module_name=local:business-imports --force
```

Run the business imports again whenever your project data changes. Open a new terminal to get tab completion (`./mercy update --instance=<tab>`).

## Update and rollback

The release to run is `kernel.version` of the production stage (`project/config/prod.yaml` or the instance). To switch
to a new release, or back to an older one:

```bash
# change kernel.version in the project repository, commit and push, then on the server:
git pull && git -C project pull
./mercy update
```

`update` pulls the image, copies its release files, recalculates the env values (`APP_VERSION`,
`DOCKER_REGISTRY_IMAGE`, ...), restarts the containers, updates the proxy hosts and runs the migrations. Values you
added to `.env` by hand stay.

Check what runs:

```bash
./mercy                                                   # status and next step
docker compose ps --format '{{.Service}} {{.Image}}'
```

A rollback to an older release works the same way, but database migrations are not reverted. Make a backup before
updates that change the database.

## Configuration

The config comes from the project repository, stage `prod`:

Mercy defaults → `setup.yaml` (and its `extends`, e.g. the network) → `prod.yaml` → the instance → `secrets.yaml`
→ `prod.secrets.yaml`.

The merged result is written to `storage/app/last-setup/` on every run (generated, do not edit). Show it with:

```bash
./mercy config                 # values the project sets, with their origin (secrets masked)
./mercy config --all           # including the Mercy defaults
```

Details of the config model (layers, instances, networks, secrets) are in the
[scaffold README](https://github.com/aklebe-laravel/mercy-scaffold#project-repository), all keys in the
[Mercy defaults](https://github.com/aklebe-laravel/mercy-php-deployment/blob/master/resources/defaults/setup.yaml).

Typical production values (in `prod.yaml` or the instance):

```yaml
extends: [ ../networks/home.yaml ]
kernel:
  domain: "my-product.example.com"
  ip: "192.0.2.10"           # host ip, used by Nginx Proxy Manager on another machine
  version: "1.4.0"           # the release to run
```

Keep a copy of `secrets.yaml` in your password manager: it exists only on this server.

## Commands

| Command                   | Purpose                                                                                |
|---------------------------|----------------------------------------------------------------------------------------|
| `./mercy`                 | status: project, instance, config warnings, secrets, containers, URL and the next step |
| `./mercy install`         | first installation                                                                     |
| `./mercy update`          | switch to `kernel.version`, refresh env and release files                              |
| `./mercy config [filter]` | effective config with the origin of every value (`--all`, `--redundant`, `--instance`) |
| `./mercy destroy`         | remove the installation, see below                                                     |

`--info` lists the steps of `install` and `update`; `./mercy update <step>` continues at a step after an error. All
steps: [mercy-php-deployment](https://github.com/aklebe-laravel/mercy-php-deployment#commands-and-steps).

Useful Docker commands in the installer folder:

```bash
docker compose logs -f mercy.app
docker compose exec mercy.app php artisan mercy:info
docker compose exec mercy.app php artisan config:clear
```

## Folder structure

The installer has the same layout as the scaffold, so paths are the same on dev, on the server and inside the container:

| Path                      | Meaning                                                                         |
|---------------------------|---------------------------------------------------------------------------------|
| `mercy`                   | the CLI                                                                         |
| `project/`                | your project repository: **the only place to edit**                             |
| `resources/setup/`        | templates, copied out of the image on every `install`/`update` (do not edit)    |
| `storage/app/last-setup/` | generated: merged config and `setup-production.env` (fixed instance key, ports) |
| `compose.yaml`            | copied out of the image, matches the release                                    |
| `.env`                    | env of the containers (generated; values you add by hand stay)                  |

Upgrading from the old layout (`install/config`, `install/templates`): run `./mercy update` once, it moves
`install/config/.env` to `storage/app/last-setup/setup-production.env`. Then delete the `install` folder.

## Remove an installation

```bash
./mercy destroy                    # containers, networks, volumes (the DATABASE), proxy hosts
./mercy destroy --images --files   # also the pulled images and all files of the folder
```

`destroy` reads the project name from Docker and asks you to type it. With `--files` it refuses while a git repository
in the folder (installer, `project/`) has uncommitted or unpushed changes. `--force` skips both checks.

## Troubleshooting

| Situation                               | What to do                                                                                         |
|-----------------------------------------|----------------------------------------------------------------------------------------------------|
| a step failed                           | fix the cause, then `./mercy update <step>` (or `install <step>`)                                  |
| image not found / access denied         | check `kernel.version` and `git.registry_*` (`./mercy config git`), then `docker login <registry>` |
| a setting seems to be ignored           | `./mercy config <key>` and the warnings about unknown keys                                         |
| "an instance is required"               | `./mercy update --instance=<tab>`                                                                  |
| business data missing after the install | run the business imports (see [Install](#install))                                                 |

## License

MIT, see [LICENSE.md](LICENSE.md).
