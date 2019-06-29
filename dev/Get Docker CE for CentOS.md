# Get Docker CE for CentOS

Estimated reading time: 10 minutes

To get started with Docker CE on CentOS, make sure you [meet the prerequisites](https://docs.docker.com/install/linux/docker-ce/centos/#prerequisites), then [install Docker](https://docs.docker.com/install/linux/docker-ce/centos/#install-docker-ce).

## Prerequisites

### Docker EE customers

To install Docker Enterprise Edition (Docker EE), go to [Get Docker EE for CentOS](https://docs.docker.com/install/linux/docker-ee/centos/) **instead of this topic**.

To learn more about Docker EE, see [Docker Enterprise Edition](https://www.docker.com/enterprise-edition/).

### OS requirements

To install Docker CE, you need a maintained version of CentOS 7. Archived versions aren’t supported or tested.

The `centos-extras` repository must be enabled. This repository is enabled by default, but if you have disabled it, you need to [re-enable it](https://wiki.centos.org/AdditionalResources/Repositories).

The `overlay2` storage driver is recommended.

### Uninstall old versions

Older versions of Docker were called `docker` or `docker-engine`. If these are installed, uninstall them, along with associated dependencies.

```
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```

It’s OK if `yum` reports that none of these packages are installed.

The contents of `/var/lib/docker/`, including images, containers, volumes, and networks, are preserved. The Docker CE package is now called `docker-ce`.

## Install Docker CE

You can install Docker CE in different ways, depending on your needs:

- Most users [set up Docker’s repositories](https://docs.docker.com/install/linux/docker-ce/centos/#install-using-the-repository) and install from them, for ease of installation and upgrade tasks. This is the recommended approach.
- Some users download the RPM package and [install it manually](https://docs.docker.com/install/linux/docker-ce/centos/#install-from-a-package) and manage upgrades completely manually. This is useful in situations such as installing Docker on air-gapped systems with no access to the internet.
- In testing and development environments, some users choose to use automated [convenience scripts](https://docs.docker.com/install/linux/docker-ce/centos/#install-using-the-convenience-script) to install Docker.

### Install using the repository

Before you install Docker CE for the first time on a new host machine, you need to set up the Docker repository. Afterward, you can install and update Docker from the repository.

#### SET UP THE REPOSITORY

1. Install required packages. `yum-utils` provides the `yum-config-manager` utility, and `device-mapper-persistent-data` and `lvm2`are required by the `devicemapper` storage driver.

   ```
   $ sudo yum install -y yum-utils \
     device-mapper-persistent-data \
     lvm2
   ```

2. Use the following command to set up the **stable** repository. You always need the **stable** repository, even if you want to install builds from the **edge** or **test** repositories as well.

   ```
   $ sudo yum-config-manager \
       --add-repo \
       https://download.docker.com/linux/centos/docker-ce.repo
   ```

3. **Optional**: Enable the **edge** and **test** repositories. These repositories are included in the `docker.repo` file above but are disabled by default. You can enable them alongside the stable repository.

   ```
   $ sudo yum-config-manager --enable docker-ce-edge
   ```

   ```
   $ sudo yum-config-manager --enable docker-ce-test
   ```

   You can disable the **edge** or **test** repository by running the `yum-config-manager` command with the `--disable` flag. To re-enable it, use the `--enable` flag. The following command disables the **edge** repository.

   ```
   $ sudo yum-config-manager --disable docker-ce-edge
   ```

   > **Note**: Starting with Docker 17.06, stable releases are also pushed to the **edge** and **test** repositories.

   [Learn about **stable** and **edge** builds](https://docs.docker.com/install/).

#### INSTALL DOCKER CE

1. Install the *latest version* of Docker CE, or go to the next step to install a specific version:

   ```
   $ sudo yum install docker-ce
   ```

   If prompted to accept the GPG key, verify that the fingerprint matches `060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35`, and if so, accept it.

   > Got multiple Docker repositories?
   >
   > If you have multiple Docker repositories enabled, installing or updating without specifying a version in the `yum install`or `yum update` command always installs the highest possible version, which may not be appropriate for your stability needs.

   Docker is installed but not started. The `docker` group is created, but no users are added to the group.

2. To install a *specific version* of Docker CE, list the available versions in the repo, then select and install:

   a. List and sort the versions available in your repo. This example sorts results by version number, highest to lowest, and is truncated:

   ```
   $ yum list docker-ce --showduplicates | sort -r
   
   docker-ce.x86_64            18.03.0.ce-1.el7.centos             docker-ce-stable
   ```

   The list returned depends on which repositories are enabled, and is specific to your version of CentOS (indicated by the `.el7`suffix in this example).

   b. Install a specific version by its fully qualified package name, which is the package name (`docker-ce`) plus the version string (2nd column) up to the first hyphen, separated by a hyphen (`-`), for example, `docker-ce-18.03.0.ce`.

   ```
   $ sudo yum install docker-ce-<VERSION STRING>
   ```

   Docker is installed but not started. The `docker` group is created, but no users are added to the group.

3. Start Docker.

   ```
   $ sudo systemctl start docker
   ```

4. Verify that `docker` is installed correctly by running the `hello-world` image.

   ```
   $ sudo docker run hello-world
   ```

   This command downloads a test image and runs it in a container. When the container runs, it prints an informational message and exits.

Docker CE is installed and running. You need to use `sudo` to run Docker commands. Continue to [Linux postinstall](https://docs.docker.com/install/linux/linux-postinstall/) to allow non-privileged users to run Docker commands and for other optional configuration steps.

#### UPGRADE DOCKER CE

To upgrade Docker CE, follow the [installation instructions](https://docs.docker.com/install/linux/docker-ce/centos/#install-docker-ce), choosing the new version you want to install.





read: connection reset by peer.





## 配置 Docker 加速器（https://www.daocloud.io/mirror）

### Linux

```
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
```

该脚本可以将 --registry-mirror 加入到你的 Docker 配置文件 /etc/docker/daemon.json 中。适用于 Ubuntu14.04、Debian、CentOS6 、CentOS7、Fedora、Arch Linux、openSUSE Leap 42.1，其他版本可能有细微不同。更多详情请访问文档。

### macOS

Docker For Mac

右键点击桌面顶栏的 docker 图标，选择 Preferences ，在 Daemon 标签（Docker 17.03 之前版本为 Advanced 标签）下的 Registry mirrors 列表中加入下面的镜像地址:

```
http://f1361db2.m.daocloud.io
```

点击 Apply & Restart 按钮使设置生效。

### Windows

Docker For Windows

在桌面右下角状态栏中右键 docker 图标，修改在 Docker Daemon 标签页中的 json ，把下面的地址:

```
http://f1361db2.m.daocloud.io
```

加到" `registry-mirrors`"的数组里。点击 Apply 。