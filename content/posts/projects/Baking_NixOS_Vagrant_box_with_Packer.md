---
title: "Baking NixOS Vagrant box with Packer"
date: 2025-02-13T23:22:01+01:00
tags:
  - packer
  - vagrant
  - nixos
cover:
  image: /posts/projects/images/baking_nixos_vagrant_box_with_packer.png
---

# Intro
I've never created a Vagrant box on my own, I mean not in a declarative way, once I exported a box but that doesn't really count.
This week I've been experimenting with the idea of building a Vagrant box for NixOS with Hashicorp Packer.

I've been using Nix as a package manager for MacOS since almost a month so by no means I'm an expert and I've got a long way to go but I've got a brief idea about it.

## Packer concepts
### 1. [HTTP Configuration](https://developer.hashicorp.com/packer/integrations/hashicorp/vmware/latest/components/builder/iso#http-configuration)
> Packer will create an http server serving http_directory when it is set, a random free port will be selected and the architecture of the directory referenced will be available in your builder.

This is needed for grabbing our ***.nix** configuration files but generally you will see it used especially in `boot_command` when downloading a Kickstart or Cloud-Init configuration files.

### 2. [Templating Engine - Keep ENV vars in mind](https://developer.hashicorp.com/packer/docs/templates/legacy_json_templates/engine)
I've used Packer's [shell provisioner](https://developer.hashicorp.com/packer/docs/provisioners/shell) instead of provisioning via `boot_command`, while both approaches work, waiting for your commands to be typed on the screen takes way more time than just invoking a bash script.

I rely on one of the following environment variables provided by Packer
  - If we're using VNC `boot_command` then `{{ .HTTPIP }}:{{ .HTTPPort }}` will be used to get the files stored in that directory.
  - If we're using SSH then `$PACKER_HTTP_ADDR` can be used. 
  - In order for us to keep the environment variables present when invoking the script `{{ .Vars }}` is used.
  - `{{ .Path }}` points to the path of our shell script.
  - So we end up with the following `execute_command` that is being used before invoking the script `sudo su -c '{{ .Vars }} {{ .Path }}'`

---

## Vagrant configuration
### 1. [Insecure keys](https://github.com/hashicorp/vagrant/tree/main/keys)
> These keys are the "insecure" public/private keypair we offer to base box creators for use in their base boxes so that vagrant installations can automatically SSH into the boxes.

The directory `configuration/vagrant_keys` contains the needed Vagrant **insecure keys** that should be inserted.

Packer `boot_command` adds the public keys as authorized keys (Boot command uses VNC but later on, our provision script is invoked via SSH so this step is very important).

The private key is then used to establish SSH connection to the machine.


### 2. Handle vagrant-hostname and vagrant-network generated files
Vagrant generates 2 nix configuration files when NixOS is used
```nix
# 1. vagrant-hostname configuration
{ config, pkgs, ... }:
{
  networking.hostName = "controlplane";
}

# 2. vagrant-network configuration
{ config, pkgs, ... }:
{
  networking.interfaces = {
    eth1.ipv4.addresses = [{
      address = "192.168.100.88";
      prefixLength = 24;
    }];
  };
}
```

You should take them into consideration because the nix configuration file will not include them by default and they are dynamically generated so we're not sure about the content untill the machine is provisioned, the solution is to include them with minimal configuration and let vagrant overwrite the file content.

```nix
# Importing the empty nix configuration files
imports = [
  ./vagrant-hostname.nix
  ./vagrant-network.nix
];

# Content in both files is kept minimal so that it works when baking the image when invoking nixos-install
{}
```

---

## NixOS Configuration
### 1. [Predictable Network Interface naming](https://nixos.org/manual/nixos/stable/#sec-rename-ifs)
> - By default, network cards are not given the traditional names like eth0 or eth1, whose order can change unpredictably across reboots.
> - Instead, relying on physical locations and firmware information, the scheme produces names like ens1, enp2s0, etc.

I needed to change that because Vagrant on Mac M1 always has a problem assigning static_ip, thankfully you can disable this feature with `usePredictableInterfaceNames = false;` and now I can just assign the static_ip specified in the Vagrantfile to eth1.

### 2. Temporary Vagrant user for SSH
By default NixOS comes with nixos user (which is part of the `wheel` group), this can be referenced in Packer and after we added our keys it should be fine to connect with that user, however since after the reboot, all our users will be created via `configuration.nix` file which just includes **vagrant** user (No more nixos user so we lose SSH connectivity after rebooting the machine).
```hcl
  boot_command = [
    # Add temporary vagrant user to allow for SSH connection to be established, permanent vagrant user is configured via configuration.nix
    "<wait><enter><wait10>",
    "sudo su<enter><wait>",
    "useradd vagrant --create-home<enter><wait>",
    "usermod -aG wheel vagrant<enter><wait>",
    "mkdir -pv /home/vagrant/.ssh<enter><wait>",
    "curl http://{{ .HTTPIP }}:{{ .HTTPPort }}/vagrant_keys/vagrant.pub > /home/vagrant/.ssh/authorized_keys<enter><wait>"  
  ]
```
I've decided to create a temporary vagrant user as a workaround, enable ssh via this user and on reboot still we have our new permanent vagrant user and still we are allowed to use vagrant private key to SSH into the machine since we allow vagrant insecure keys in its configuration as follows
```nix
  # I kept this part minimal, full configuration can be seen in configuration.nix file
  users.users.vagrant = {
    name            = "vagrant";
    isNormalUser = true;
    openssh.authorizedKeys.keys = [
      "ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key"
      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN1YdxBpNlzxDqfJyw/QKow1F+wvG9hXGoqiysfJOn5Y vagrant insecure public key"
    ];
  };
```

So far that's all, there should be more updates because I'm not doing any version locking for the packages used by NixOS.

## Useful Resources
1. [NixOS Vagrant boxes with NixBox](https://github.com/nix-community/nixbox)
2. [VMWare ISO Builder](https://developer.hashicorp.com/packer/integrations/hashicorp/vmware/latest/components/builder/iso)
3. [Packer plugin not working well with VMware Fusion Pro 13.0 on M1-based chipset](https://github.com/hashicorp/packer-plugin-vmware/issues/108#issuecomment-1374087471)
4. [NixOS ❄: tmpfs as root](https://elis.nu/blog/2020/05/nixos-tmpfs-as-root/)