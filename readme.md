# An [Open Container Initiative](https://opencontainers.org/)(OCI) [FreeBSD](https://www.freebsd.org/) container image.

## Requirements

Run this container image with [containerd](https://github.com/containerd/containerd/blob/main/BUILDING.md) and [nerd-ctl](https://github.com/containerd/nerdctl/blob/master/docs/freebsd.md) on [FreeBSD](https://www.freebsd.org/).

### Build and Install the Requirements

Create a `Projects` folder, install [Go](https://go.dev/), and build and install [containerd](https://github.com/containerd/containerd), [runj](https://github.com/samuelkarp/runj), [umoci](https://github.com/opencontainers/umoci) and [nerdctl](https://github.com/containerd/nerdctl.git):

```sh
mkdir ~/Projects && cd ~/Projects

sudo pkg install -y go

git clone https://github.com/jackvz/containerd
cd containerd
git checkout v1.5.0
go build ./cmd/containerd
go build ./cmd/ctr
sudo install -o 0 -g 0 containerd ctr /usr/local/bin
cd ..

git clone https://github.com/jackvz/runj
cd runj
make && sudo make install
cd ..

git clone https://github.com/jackvz/umoci
cd umoci
go build ./cmd/umoci
sudo install -o 0 -g 0 umoci /usr/local/bin
cd ..

git clone https://github.com/jackvz/nerdctl
cd nerdctl
go build ./cmd/nerdctl
sudo install -o 0 -g 0 nerdctl /usr/local/bin
cd ..
```

Note: [BuildKit](https://github.com/moby/buildkit), the Dockerfile-agnostic builder toolkit, is not yet ported to FreeBSD, but [`umoci`](https://github.com/opencontainers/umoci) can be used to modify and repack container images.

### Configure the Requirements

Set up [ZFS](https://docs.freebsd.org/en/books/handbook/zfs/) on a disk partition, and configure [containerd](https://github.com/containerd/containerd) and [nerdctl](https://github.com/containerd/nerdctl.git):

```sh
sudo zpool create -f zroot /dev/[disk-partition-eg-da0s3]
sudo zfs create -o mountpoint=/var/lib/containerd/io.containerd.snapshotter.v1.zfs zroot/containerd
sudo df
sudo ls -al /var/lib/containerd/

sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
sudo sh -c 'echo "" > /etc/containerd/config.toml'
sudo sh -c 'echo "version = 2" >> /etc/containerd/config.toml'
sudo sh -c 'echo "[plugins]" >> /etc/containerd/config.toml'
sudo sh -c 'echo "[plugins.\"io.containerd.snapshotter.v1.zfs\"]" >> /etc/containerd/config.toml'
sudo sh -c 'echo "root_path = \"/var/lib/containerd/io.containerd.snapshotter.v1.zfs\"" >> /etc/containerd/config.toml'
sudo cat /etc/containerd/config.toml

sudo mkdir -p /etc/nerdctl
sudo sh -c 'echo "" > /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "debug = true" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "debug_full = true" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "address = \"unix:////run/containerd/containerd.sock\"" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "#namespace = \"k8s.io\"" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "snapshotter = \"zfs\"" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "#cgroup_manager = \"cgroupfs\"" >> /etc/nerdctl/nerdctl.toml'
sudo sh -c 'echo "#hosts_dir = [\"/etc/containerd/certs.d\", \"/etc/docker/certs.d\"]" >> /etc/nerdctl/nerdctl.toml'
sudo cat /etc/nerdctl/nerdctl.toml
```

## Build and Deploy

Use [FreeBSD](https://www.freebsd.org/).

Set the FreeBSD release version:

```sh
export OCI_FREEBSD_VER=$(freebsd-version)
```

Get the FreeBSD root filesystem and create an OCI image with [runj](https://github.com/samuelkarp/runj):

```sh
runj demo download --output rootfs.txz
runj demo oci-image --input rootfs.txz
```

For now, run `containerd` in the background. In production, run it as a daemon. @todo: [Practical rc.d scripting in BSD](https://docs.freebsd.org/en/articles/rc-scripting/index.html)

```sh
sudo containerd > containerd.log 2>&1 &
export CONTAINERD_PID=$!
```

Import, tag and run the image with [containerd ctl](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#interacting-with-containerd-via-cli):

```sh
sudo ctr image import --snapshotter zfs --index-name freebsd:$OCI_FREEBSD_VER image.tar
sudo ctr image tag freebsd:$OCI_FREEBSD_VER docker.io/jackvanzyl1/freebsd:$OCI_FREEBSD_VER

sudo ctr run \
    --runtime wtf.sbk.runj.v1 \
    --rm \
    freebsd:$OCI_FREEBSD_VER \
    my-container \
    sh -c 'echo "Hello from the container!"'
```

Deploy the image to [DockerHub](https://hub.docker.com/) with [nerdctl](https://github.com/containerd/nerdctl):

```sh
nerdctl login registry-1.docker.io
nerdctl image push jackvanzyl1/freebsd:13.0-RELEASE
```

Stop the `containerd` process:

```sh
kill $CONTAINERD_PID
export CONTAINERD_PID=
```
