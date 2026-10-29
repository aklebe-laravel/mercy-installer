## Mercy-Installer

This installer can be used by projects developed in [Mercy Scaffold](https://github.com/aklebe-laravel/mercy-scaffold) environment.

## Download installer

To download the installer navigate to your preferred folder like ```~/projects/mercy``` and type

```
git clone https://github.com/aklebe-laravel/mercy-installer.git .
```

or (ssh if you have permissions)

```
git clone git@github.com:aklebe-laravel/mercy-installer.git .
```

## Requirements

- Docker with the compose plugin (your user in the group `docker`)
- PHP 8.5 CLI, Composer 2.9, `git`, `unzip`
- For a private registry: `docker login <registry>` (GitHub: `ghcr.io` with a classic token, `read:packages` is enough)

Tested on a small VM: Ubuntu 24.04 LTS, 1 CPU core, 2 GiB memory, 32 GB disk (install and update).

## Project repository

The config and the business imports of your project come from its project repository (see the [Mercy Scaffold README](https://github.com/aklebe-laravel/mercy-scaffold#project)).
Clone it as `project/` into the installer folder and create the secrets of this server:

```
git clone <your-project-repo> project
# project/config/secrets.yaml (all secrets, also the env vars of the business imports)
```

The installer uses the stage `prod`: `setup.yaml` → `prod.yaml` → `secrets.yaml` → `prod.secrets.yaml`,
written to `storage/app/last-setup/setup.yaml` on every run (generated, do not edit).
Show the result with `php mercy.php config`.

## Install

1) Clone the repo
2) Clone your project repository as `project/` and create `project/config/secrets.yaml` (see above)
3) Registry login (only for a private registry, see `git.registry_source` in setup.yaml)
   ```
   docker login <registry_source>
   ```
4) Composer
   ```
   composer install
   ```

5) Run the installer
   ```
   php mercy.php install
   ```

6) Follow the instructions
7) _[optionally]_ Start ldap sync
   ```
   docker compose exec mercy.app php artisan ldap:sync-all
   ```

8) For your business imports run (renew/update/overwrite)
   ```
   docker compose exec mercy.app php artisan deploy-env:terraform-modules --module_name=local:business-imports --force
   ```

   Use this again if you have changed your business data.

## Folders

The installer has the same layout as the [Mercy Scaffold](https://github.com/aklebe-laravel/mercy-scaffold#folder-structure),
so paths are the same on dev, on the production host and inside the container:

| Path                      | Meaning                                                                         |
|---------------------------|---------------------------------------------------------------------------------|
| `project/`                | your project repository: **the only place to edit**                             |
| `resources/setup/`        | templates, copied out of the image on every `install`/`update` (do not edit)    |
| `storage/app/last-setup/` | generated: merged config and `setup-production.env` (fixed instance key, ports) |
| `compose.yaml`            | copied out of the image, it matches the image version                           |
| `.env`                    | the env of the containers (generated, values you add by hand stay)              |

Upgrading from the old layout (`install/config`, `install/templates`): run `php mercy.php update` once,
it moves `install/config/.env` to `storage/app/last-setup/setup-production.env`. Then delete the `install` folder.

## Updates

`kernel.version` in `project/config/prod.yaml` is the release to run (the image tag).
To switch to another release (or back), change it (commit it in the project repository, `git -C project pull`) and run:

```
php mercy.php update
```

The update pulls that image, copies its release files and recalculates the env values (`APP_VERSION`,
`DOCKER_REGISTRY_IMAGE`, ...) in `storage/app/last-setup/setup-production.env` and `.env`. Values you wrote by hand in `.env` stay.
Check what runs: `docker compose ps --format '{{.Service}} {{.Image}}'`.

## Destroy the whole project (inclusive dockers and images)

```
# containers, networks and volumes (the DATABASE) of this folder, images built here
php mercy.php destroy

# also the pulled images (app image, mariadb, redis) and/or all files of the folder
php mercy.php destroy --images --files
```

`destroy` uses the docker project name of the `.env` and asks you to type it. With `--files` it refuses
as long as a git repo in the folder (scaffold, `project/`, modules, themes) has uncommitted or unpushed changes.
`--force` skips both checks (for scripts).

