# osbuild + ostree native container lab

Hands-on lab for using osbuild and ostree native containers

In this repo, you'll find instructions for using osbuild to generate a Fedora IoT artifact, creating a new base container image from the ostree commit, and creating customized container images to rebase your Fedora IoT system to.

## Prerequisites

- Fedora virtual machine(s)
- Quay.io account
- General understanding of osbuild
- General understanding of containers

You'll need one or more Fedora virtual machines: one system for installing osbuild and building artifacts. Another system for installing Fedora IoT; this could be another VM or a separate physical device.  (Your osbuild system could also be a physical system, but it assumed most folks will be using a VM)

**NOTE:** your osbuild system should have at least 20G of unused, available disk space in order to successfully build this lab.

We'll refer to the system where osbuild runs as the "builder" system and the system where Fedora IoT will be installed as your "destination" system.

## Step 1: Install osbuild

On your "builder" system, follow the [instructions](https://www.osbuild.org/guides/image-builder-on-premises/installation.html) for installing osbuild and osbuild-composer.

```bash
sudo dnf install osbuild-composer composer-cli
sudo systemctl enable --now osbuild-composer.socket
sudo usermod -a -G weldr <user>
newgrp weldr
```

## Step 2: Create empty blueprint for ostree commit

On your "builder" system, create an empty blueprint that can be used to create an ostree commit.

Your empty TOML file should look something like this:

```toml
name = "minimal"
description = "minimal fedora iot blueprint"
version = "1.0.0"
modules = []
groups = []
distro = "fedora-38"
```

Push the blueprint with `composer-cli`:

`composer-cli blueprints push minimal.toml`

## Step 3: Create base ostree commit

Start the build process for the ostree commit; we're choosing the `iot-commit` artifact because we want to be able to easily access the resulting ostree repo on-disk and via HTTP:

`composer-cli compose start-ostree minimal iot-commit`

Wait the necessary time for the build to complete; time will vary based on the specs of your "builder" system.

To watch the status of the compose use:

`watch -n 15 composer-cli compose info <uuid>`

When the build has completed, fetch the tarball, explode it to disk, and serve it up via HTTP.  This might be best done in a separate terminal so as not to flood your working terminal with log messages.

```bash
composer-cli compose image <uuid>
tar -xf <uuid>.tar
cd repo
python3 -m http.server
```

**HINT**: the `commit.tar` artifact should look simiar to this:

```bash
$ ls -lh 64cd0719-fd07-4b32-a116-0192cecdada3-commit.tar
-rw-------. 1 core weldr 785M Jul 27 15:36 64cd0719-fd07-4b32-a116-0192cecdada3-commit.tar
```


Note the location of the `repo` directory on the disk, you will need this location when encapsulating the ostree commit into a container image.

## Step 4: Create blueprint for installer artifact

Create a blueprint that you can use to create an installer artifact.  You should include some basic user customizations so that you can login to your Fedora IoT system later.

```toml
name = "iso"
description = "blueprint to create iot-installer"
version = "1.0.0"
distro = "fedora-38"

[[customizations.user]]
name = "core"
key = "ssh-rsa ...."
password = "..."
groups = ["wheel"]
```

And push the blueprint with `composer-cli`

`composer-cli blueprints push iso.toml`

## Step 5: Create installer artifact

Start the build process for the installer artifact; the ostree repo should be served via HTTP on port 8080 from the previous steps.

`composer-cli compose start-ostree --url http://localhost:8000/ iso iot-installer`

Wait for the build to complete and then fetch the ISO to be used for installation:

```bash
watch -n 15 composer-cli compose info <uuid>
composer-cli compose image <uuid>
```

## Step 6: Install Fedora IoT to separate VM/device

Using the ISO you created in the previous steps, install Fedora IoT in a VM or on your "destination" system.

The Anaconda installer should walk you through the necessary steps; they will not be covered as part of this lab.

Ensure that you are able to boot your "destination" system into Fedora IoT and successfully SSH to the IoT system.

## Step 7: Create/push ostree native container

On your "builder" system, create a new base image from the ostree commit using the `rpm-ostree` command.

`rpm-ostree compose container-encapsulate --repo=</path/to/ostree/repo> fedora/38/x86_64/iot containers-storage:quay.io/<username>/fedora-iot-base:38`

Optionally tag your image as `:latest` and push the image reference(s) to the registry; the `podman login` command can be retrieved from the settings page of your Quay.io account.

```bash
podman login -u="<username>" -p="<password>" quay.io
podman tag quay.io/<username>/fedora-iot-base:38 quay.io/<username>/fedora-iot-base:latest
podman push quay.io/<username>/fedora-iot-base:38
podman push quay.io/<username>/fedora-iot-base:latest
```

You should go to the settings page for your new container image on Quay.io and mark the repository as public.

## Step 8: Create customized Containerfile

Create a Containerfile that starts with your new base image and use `rpm-ostree install` to install a new package.  An example is provided below:

```Dockerfile
FROM quay.io/<username>/fedora-iot-base:38
RUN rpm-ostree install strace && \
    ostree container commit
```

**NOTE**: it is recommended (required?) to use `ostree container commit` as your final step, as it performs finalization steps that are needed for ostree systems.  (For more information, see <https://github.com/ostreedev/ostree-rs-ext/issues/159>)

You could create this Containerfile on your "builder" system or any other system where `podman` is installed.

## Step 9: Build/push customized container image

With the Containerfile, you can now build and push a new container image.

```bash
podman build -t quay.io/<username>/fedora-iot-custom:latest -f Containerfile .
podman push quay.io/<username>/fedora-iot-custom:latest
```

You'll need to go to the settings page on Quay.io for the custom image you just pushed and mark the repository as public.

## Step 10: Rebase Fedora IoT system to container image

On your "destination" system where Fedora IoT has been installed, rebase the system to your new customized image.

`sudo rpm-ostree rebase --experimental ostree-unverified-registry:quay.io/<username>/fedora-iot-custom:latest`

When you reboot your "destination" system, you should be in a new deployment and the package that you had installed in your Containerfile should be available.

## Step 11: Enable automatic updates

Let's enable automatic updates driven by `rpm-ostree` and watch what happens when we update the container image on the registry.

On your "destination" system, configure the automatic update mechanism of `rpm-ostree` to stage the updates. This allows the user to have control when they want to have the updates applied.

```bash
sudo sed -i '/AutomaticUpdatePolicy=none/c\AutomaticUpdatePolicy=stage' /etc/rpm-ostreed.conf
sudo rpm-ostree reload
```

And adjust the timer for the updates to happen every minute so we don't have to wait around for the timer to fire.  Create a drop-in file to override the timer settings and then reload systemd:

```bash
sudo mkdir -p /etc/systemd/system/rpm-ostreed-automatic.timer.d/
sudo tee /etc/systemd/system/rpm-ostreed-automatic.timer.d/override.conf > /dev/null << EOF
[Timer]
OnActiveSec=60
EOF
sudo systemctl daemon-reload
sudo systemctl enable rpm-ostreed-automatic.timer --now
```

## Step 12: Add configuration file to Containerfile

We can embed configuration files (or any files really) as part of the customized container image. This streamlines the process of doing OS updates and configuration changes via the same mechanism.

In this example, we are adding a `dnf` repo to the system. Create the `/etc` directory structure and add in a `.repo` file.

```bash
mkdir -p etc/yum.repos.d/
pushd etc/yum.repos.d/
curl -LO https://copr.fedorainfracloud.org/coprs/g/redhat-et/microshift/repo/fedora-38/group_redhat-et-microshift-fedora-38.repo
popd
```

Create a new Containerfile to copy in the `.repo` file.

```Dockerfile
FROM quay.io/<username>/fedora-iot-custom:latest
COPY etc /etc
```

Your directory structure should look like this:

```bash
 tree .
.
├── Containerfile.update
└── etc
    └── yum.repos.d
        └── group_redhat-et-microshift-fedora-38.repo

3 directories, 2 files
```

## Step 13: Build/push updated container image

Now build and push the updated image; we'll use the same `:latest` tag so that the automatic update mechanism picks up the new container image.

```bash
podman build -t quay.io/<username>/fedora-iot-custom:latest -f Containerfile.update .
podman push quay.io/<username>/fedora-iot-custom:latest
```

## Step 14: Observe available update

On the destination Fedora IoT system, you should be able to see the updated container image being fetched by the automatic update mechanism.  Or the update has already been fetched and staged.

`rpm-ostree status`
