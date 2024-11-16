---
title: "Homelab Kubernetes cluster with Talos, Matchbox and Cilium"
date: "2024-11-16"
tags: ["kubernetes", "talos", "cilium", "homelab"]
showTags: true
draft: true
---

## Intro

Some time ago, I deployed a Kubernetes cluster as a homelab for running some workload and testing some stuff. Back then, I used Unbutu server with straight `kubeadm` to bootstrap the nodes. While it was working, it was not really fun to maintain the OS layer.

Then I came accros [Talos](https://www.talos.dev/) and started using that as the underlying Linux distro on my nodes. My first experience with Talos was the good old : download the boot assets, write them to a USB key and go from there.

Recently, I decided to improve this setup and make the nodes deploy in an automated fashion via iPXE. This write up covers the steps to get there.

* [Talos](https://www.talos.dev/): Talos Linux is a distribution dedicated to running Kubernetes. It can be deployed almost anywhere, which makes it an excellent choice for a homelab. By default, it contains only what is necessary to run Kubernetes. It doesn't provide a terminal access, everything goes through an API. It's also quite straightforward to manage and upgrade.

* [Matchbox](https://matchbox.psdn.io/): Although you could simply use a USB stick and `talosctl` to configure Talos nodes, I decided to deploy and bootstrap my nodes via iPXE. Matchbox will help in that regards by using the correct confugration file (control plane vs. worker node) depending on the MAC address.

* [Cilium](https://cilium.io/): Cilium is my cluster CNI. Aside from the networking capabilities, it can also provide [LB IPAM](https://docs.cilium.io/en/stable/network/lb-ipam/) capabilities instead of deploying an additional tool like [MetalLB](https://metallb.universe.tf/).

### Sources

A lot of what is detailed below has been sourced from different places :

* [Talos documentation](https://www.talos.dev/v1.7/introduction/)
* [Matchbox documentation](https://matchbox.psdn.io/)
* [Cilium documentation](https://docs.cilium.io/en/stable/)

## Preparation

### Matchbox networking pre-quisite

As I'm using a Raspberry PI with [Pi-hole](https://pi-hole.net/) as my DHCP server, I will skip `dnsmasq` general installation and configuration and just focus on what's needed to make it uses Matchbox as the PXE service.

Create an additionnal config file in `/etc/dnsmasq.d` folder...

```bash
vim /etc/dnsmasq.d/07-tftp-server.conf
```

...and add the following.

```
enable-tftp
tftp-root=/var/lib/tftpboot
dhcp-userclass=set:ipxe,iPXE
pxe-service=tag:#ipxe,x86PC,"PXE chainload to iPXE",undionly.kpxe
pxe-service=tag:ipxe,x86PC,"iPXE",http://matchbox.lan:8080/boot.ipxe #Make sure the hostname point to the machine matchbox is deployed to
```

You will probably need to restart `dnsmasq` service.

Next add [http://boot.ipxe.org/undionly.kpxe](http://boot.ipxe.org/undionly.kpxe) to the `tftp-root`. (in this case `/var/lib/tftpboot`)

### Matchbox installation

Download the [latest Matchbox release](https://github.com/poseidon/matchbox/releases). As I'm running Matchbox on a RPI, I'm using the ARM64 release below. Make sure to use the one that matches your architecture.

```bash
#Download latest release
wget https://github.com/poseidon/matchbox/releases/download/v0.11.0/matchbox-v0.11.0-linux-arm64.tar.gz
#Verify signature
gpg --keyserver keyserver.ubuntu.com --recv-key 2E3D92BF07D9DDCCB3BAE4A48F515AD1602065C8
gpg --verify matchbox-v0.11.0-linux-arm64.tar.gz.asc matchbox-v0.11.0-linux-arm64.tar.gz
#Untar
tar xzvf matchbox-v0.10.0-linux-arm64.tar.gz
```

Copy the binary to an appropriate location in your $PATH`

```bash
sudo cp matchbox /usr/local/bin
```

Create a dedicated user for the matchbox service.

```bash
sudo useradd -U matchbox
sudo mkdir -p /var/lib/matchbox/assets
sudo chown -R matchbox:matchbox /var/lib/matchbox
```

Copy the provided Matchbox systemd unit file.

```bash
sudo cp contrib/systemd/matchbox.service /etc/systemd/system/matchbox.service
```

Now edit the systemd service...

```bash
sudo systemctl edit matchbox
```

...and enable the gRPC API. This is needed as later we will be using [OpenTofu](https://opentofu.org/) to configure the Matchbox profiles for the Talos nodes. If you don't intend to use OpenTofu for this and configure the Matchbox profiles by other means, you can also leave it as is.

```
[Service]
Environment="MATCHBOX_ADDRESS=0.0.0.0:8080"
Environment="MATCHBOX_LOG_LEVEL=debug"
Environment="MATCHBOX_RPC_ADDRESS=0.0.0.0:8081"
```

Eventually, start and enable the service.

```bash
sudo systemctl daemon-reload
sudo systemctl start matchbox
sudo systemctl enable matchbox
```

### Talos configuration files

Now you need to generate the Talos configuration files for our machines.

`192.168.1.50` is a Virtual IP that will be shared by the controlplane nodes. It will serve as the main API address for Kubernetes.

```bash
talosctl gen config talos-k8s-metal-tutorial https://192.168.1.50:6443
created controlplane.yaml
created worker.yaml
created talosconfig
```

In `controlplane.yaml`, add the VIP that will be shared between the controlplane nodes.

```yaml
...
machine:
  network:
    # `interfaces` is used to define the network interface configuration.
    interfaces:
        - interface: eno1 # The interface name.
          dhcp: true # Indicates if DHCP should be used to configure the interface.
          # Virtual (shared) IP address configuration.
          vip:
            ip: 192.168.1.50 # Specifies the IP address to be used.
...
```

As I am using [Longhorn](https://longhorn.io) to provide local block storage to my workloads, I will aso add the following to both `workload.yaml` and `controlplane.yaml` files. If you don't need Longhorn, you can skip this step.

```yaml
...
machine:
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn # Destination is the absolute path where the mount will be placed in the container.
        type: bind # Type specifies the mount kind.
        source: /var/lib/longhorn # Source specifies the source path of the mount.
        # Options are fstab style mount options.
        options:
          - bind
          - rshared
          - rw
...
```

I will also change the default installer, as Longhorn requires some specific [system extensions](https://www.talos.dev/v1.8/talos-guides/configuration/system-extensions/) to function. For this, I use [Talos Linux Image Factory](https://factory.talos.dev) to generate a schematic ID.

![talos-factory](/img/homelab-k8s-talos-factory.png)

The schematic ID is then used to replace the default installer in both `workload.yaml` and `controlplane.yaml` files.

Note that depending on your hardware configuration, you may also need to change the `diskSelector`. This will pick any device that is plugged in and has 500GB or more. You can also directly [specify the device](https://www.talos.dev/v1.8/reference/configuration/v1alpha1/config/#Config.machine.install).

```yaml
machine:
  install:
    diskSelector:
        size: '>= 500GB' # Disk size.
    image: factory.talos.dev/installer/613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245:v1.8.2
```

Last thing, I'm going to disable `kube-proxy` as I will be replacing it with Cilium. This is only configured in the `controlplane.yaml` file.

```yaml
cluster:
  proxy:
    disabled: true # Disable kube-proxy deployment on cluster bootstrap.
```

Using `talosctl`, you can verify that the configuration files are still valid.

```bash
talosctl validate --config controlplane.yaml --mode metal
  controlplane.yaml is valid for metal mode
talosctl validate --config worker.yaml --mode metal
  worker.yaml is valid for metal mode
```

The last step is to copy both `workload.yaml` and `controlplane.yaml` files to the `assets` folder on the Matchbox host.

```bash
tree /var/lib/matchbox/
/var/lib/matchbox/
|-- assets
|   `-- talos
|       |-- controlplane.yaml
|       `-- worker.yaml
```

### Generate Matchbox TLS certificate

Because I'm using [OpenTofu](https://opentofu.org/) to configure the Matchbox profiles, I need to generate TLS certificates to authenticate with Matchbox gRPC API.

```bash
export SAN=DNS.1:matchbox.lan,IP.1:192.168.1.33 #make sure this match your environment
```

Run the [provided script](https://github.com/poseidon/matchbox/blob/main/scripts/tls/cert-gen) to generate the TLS certificates.

```bash
cd scripts/tls
./cert-gen
```

Then move the server TLS files to the Matchbox server's default location.

```bash
sudo mkdir -p /etc/matchbox
sudo cp ca.crt server.crt server.key /etc/matchbox
sudo chown -R matchbox:matchbox /etc/matchbox
```

Make sure to save the client TLS file for later (`client.crt`, `client.key` and `ca.crt`). They will be required to configure Matchbox with OpenTofu.

### Create Matchbox profiles

I will be using [OpenTofu](https://opentofu.org/) to create the matchbox profiles using the Matchbox provider and the Talos provider to get the correct schematic ID for the boot assets. The complete code is available [in this repo](https://codeberg.org/nerkho/tofu-matchbox-config).

Start by configuring the provider and copy the client TLS file that you saved earlier to a `certs` folder alongside the configuration files.

```hcl
terraform {
  required_providers {
    matchbox = {
      source  = "poseidon/matchbox"
      version = "0.5.4"
    }
    talos = {
      source  = "siderolabs/talos"
      version = "0.7.0-alpha.0"
    }
  }
}

provider "matchbox" {
  endpoint    = "matchbox.lan:8081" # make sure this match your environment
  client_cert = file("certs/client.crt")
  client_key  = file("certs/client.key")
  ca          = file("certs/ca.crt")
}
```

Also add the configuration for the Talos image factory. This will give us the schematic ID for our boot assets.

```terraform
data "talos_image_factory_extensions_versions" "image_factory" {
  # get the latest talos version
  talos_version = var.talos_version
  filters = {
    names = var.talos_extensions
  }
}

resource "talos_image_factory_schematic" "image_factory" {
  schematic = yamlencode(
    {
      customization = {
        systemExtensions = {
          officialExtensions = data.talos_image_factory_extensions_versions.image_factory.extensions_info.*.name
        }
      }
    }
  )
}
```

Create two `locals` for the boot assets. The cool trick here is to directly point to the Talos image factory. There is no need to save these on the matchbox host.
```terraform
locals {
  kernel = "https://pxe.factory.talos.dev/image/${talos_image_factory_schematic.image_factory.id}/${var.talos_version}/kernel-amd64"
  initrd = "https://pxe.factory.talos.dev/image/${talos_image_factory_schematic.image_factory.id}/${var.talos_version}/initramfs-amd64.xz"
}
```

Add the matchbox_group and matchbox_profile resource configuration for the controlplane nodes.
The `talos.config` parameter should matches with the location of the `controlplane.yaml` file on the Matchbox host. It's this parameter that tells Talos were to find the configuration file for the node.

```hcl
resource "matchbox_profile" "controlplane" {
  name   = "controlplane"
  kernel = local.kernel
  initrd = [local.initrd]
  args = [
    "initrd=initramfs.xz",
    "init_on_alloc=1",
    "slab_nomerge",
    "pti=on",
    "console=tty0",
    "console=ttyS0",
    "printk.devkmsg=on",
    "talos.platform=metal",
    "talos.config=http://matchbox.lan:8080/assets/talos/controlplane.yaml"
  ]
}

resource "matchbox_group" "controlplane" {
  for_each = var.controlplanes
  name     = each.key
  profile  = matchbox_profile.controlplane.name
  selector = {
    mac = each.value
  }
}
```

Then simply repeat the same for the worker nodes.

```hcl
resource "matchbox_profile" "worker" {
  name   = "worker"
  kernel = local.kernel
  initrd = [local.initrd]
  args = [
    "initrd=initramfs.xz",
    "init_on_alloc=1",
    "slab_nomerge",
    "pti=on",
    "console=tty0",
    "console=ttyS0",
    "printk.devkmsg=on",
    "talos.platform=metal",
    "talos.config=http://matchbox.lan:8080/assets/talos/worker.yaml"
  ]
}

resource "matchbox_group" "worker" {
  for_each = var.workers
  name     = each.key
  profile  = matchbox_profile.worker.name
  selector = {
    mac = each.value
  }
}
```

Your *.tfvars file should look something like this with the mac address for each node. This is how Matchbox will determine which profile to use.

```terraform
talos_version    = "v1.8.2"
talos_extensions = ["iscsi-tools", "util-linux-tools"]
controlplanes = {
  controlplane-node01 = "00:00:00:00:00:00"
  controlplane-node02 = "00:00:00:00:00:00"
  controlplane-node03 = "00:00:00:00:00:00"
}
workers = {
  worker-node01 = "00:00:00:00:00:00"
}
```

You can now apply the configuration.

```bash
tofu plan
tofu apply
```

If all went well, the groups and profiles will be created on the Matchbox host.

```bash
tree /var/lib/matchbox/
/var/lib/matchbox/
|-- assets
|   `-- talos
|       |-- controlplane.yaml
|       `-- worker.yaml
|-- groups
|   |-- controlplane-node01.json
|   |-- controlplane-node02.json
|   |-- controlplane-node03.json
|   `-- worker-node01.json
`-- profiles
    |-- controlplane.json
    `-- worker.json
```

## Bootstrap the nodes

This is it! You can now boot your first nodes. If all goes well, it should look similar to this :

![bootstrap](/img/homelab-k8s-bootstrap.gif)

Now you can bootstrap your cluster. You only need to run this command once against one of the controlplane node :

```bash
talosctl bootstrap -n 192.168.1.20
```

At this point, the other nodes should automatically join the cluster if they are already started.
Once the bootstrapping of all nodes is complete, most of the stuff at the top should come up green. The kubeconfig should automatically be copied to  `~/.kube` on our system, but it's also possible to retrieve it `talosctl kubeconfig -n 192.168.1.20`.

![talosdashboard](/img/homelab-k8s-talos-dashboard-2.png)

If you disabled `kube-proxy` earlier, you will notice with `kubectl get nodes` that the nodes are posting a `NotReady` state. We still need to deploy Cilium.

## Cilium

### Install Cilium

The recommended way to install Cilium is with Helm.

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

Mostly, you can re-use the recommended options from the [Talos documentation](https://www.talos.dev/v1.7/kubernetes-guides/network/deploying-cilium/#without-kube-proxy).

Just add `l2announcements.enabled=true` as Cilium [L2 announcement feature](https://docs.cilium.io/en/stable/network/l2-announcements/#l2-announcements) is required in conjunction with [LB IPAM feature](https://docs.cilium.io/en/stable/network/lb-ipam/). Those 2 options will allow Cilium to automatically assign IP addresses to LoadBalancer type services.

```bash
helm install \
    cilium cilium/cilium \
    --version 1.15.5 \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set=kubeProxyReplacement=true \
    --set=securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set=securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set=cgroup.autoMount.enabled=false \
    --set=cgroup.hostRoot=/sys/fs/cgroup \
    --set=k8sServiceHost=localhost \
    --set=k8sServicePort=7445 \
    --set=l2announcements.enabled=true
```

With [cilium-cli](https://github.com/cilium/cilium-cli), we can easily check the deployment status.

```bash
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
DaemonSet              cilium             Desired: 4, Ready: 4/4, Available: 4/4
Containers:            cilium             Running: 4
                       cilium-operator    Running: 2
Cluster Pods:          75/75 managed by Cilium
Helm chart version:
Image versions         cilium             quay.io/cilium/cilium:v1.15.5@sha256:4ce1666a73815101ec9a4d360af6c5b7f1193ab00d89b7124f8505dee147ca40: 4
                       cilium-operator    quay.io/cilium/operator-generic:v1.15.5@sha256:f5d3d19754074ca052be6aac5d1ffb1de1eb5f2d947222b5f10f6d97ad4383e8: 2
```

Once everything is green, the Kubernetes nodes should also post a `Ready` state.

### Enable L2 announcement and LB IPAM features

This is quite straight forward.

Define an IP address pool :

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "my-pool"
spec:
  cidrs:
  - start: "192.168.1.60"
    stop: "192.168.1.99"
```

Create a L2 announcement policy :

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2-announcement-policy
  namespace: kube-system
spec:
  externalIPs: true
  loadBalancerIPs: true
```

Now, whenever you create a LoadBalancer service type, it should automatically get an IP address assigned from the defined address pool and the service will be reachable from outside the cluster.

## Conclusion

I see various way this setup could be improved, but in general, I think it is a nice starting point for K8s homelab cluster with Talos. I hope this was helpful to someone out there :)
