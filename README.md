# Local Environment

## Helpers

### Updating of Linux Kernel

[Update Linux Kernel version](https://askubuntu.com/questions/1388115/how-do-i-update-my-kernel-to-the-latest-one)

Check Kernel version:

```bash
uname -r
```

Install the shell script which automatically checks and install the latest kernel:
```bash
wget https://raw.githubusercontent.com/pimlie/ubuntu-mainline-kernel.sh/master/ubuntu-mainline-kernel.sh
sudo install ubuntu-mainline-kernel.sh /usr/local/bin/
```

Check if a newer kernel version is available:

```bash
sudo ubuntu-mainline-kernel.sh -c
```

List available Kernel versions:

```bash
sudo ubuntu-mainline-kernel.sh -r
```

Install the latest Kernel:

```bash
sudo ubuntu-mainline-kernel.sh -i
```

Install the needed Kernel version:
```bash
sudo ubuntu-mainline-kernel.sh -i 6.5.11
```