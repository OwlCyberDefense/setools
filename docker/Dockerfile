FROM ubuntu:14.04

# define variables
ENV setools_repo https://github.com/TresysTechnology/setools
ENV setools_rev HEAD

# install build dependencies
RUN apt-get update && \
    apt-get install -y sudo git \
                       gcc gettext flex bison \
                       python python-setuptools python-all-dev \
                       libbz2-dev wget make libpcre3-dev python3-all-dev \
                       python-pip python3-pip \
                       # Install SELinux userspace utilities dependencies
                       libaudit-dev libcap-ng-dev libustr-dev libpcre3-dev

# install external build dependencies
RUN cd /tmp && \
    # Install newer swig
    wget http://prdownloads.sourceforge.net/swig/swig-2.0.12.tar.gz && \
    tar -xzf swig-2.0.12.tar.gz && cd swig-2.0.12 && \
    ./configure && make && make install

# install runtime dependencies
RUN apt-get install -y python-networkx && \
    pip2 install tox && \
    pip2 install mock

# create an user 'docker'
RUN adduser --disabled-password --gecos '' docker && \
    adduser docker sudo && \
    echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# switch to 'docker' user
USER docker
WORKDIR /home/docker

# clone repositories
RUN git clone ${setools_repo} && \
    cd ./setools/ && \
    git reset --hard ${setools_rev} && \
    # Download current SELinux userspace tools and libraries
    wget https://github.com/SELinuxProject/selinux/archive/20150202.tar.gz && \
    tar -xzf 20150202.tar.gz && mv selinux-20150202 selinux-src && \
    patch -p1 -d selinux-src < patches/libsepol-2.4-mls-semantic-level-expand.patch && \
    # Ubuntu 12.04 coreutils is too old to provide "ln --relative" :(
    sed 's/ln -sf --relative /ln -sf /' -i selinux-src/libsepol/src/Makefile && \
    sed 's/ln -sf --relative /ln -sf /' -i selinux-src/libselinux/src/Makefile && \
    # Drop sepolicy to break setools3 dependence (sepolicy isn't used hereanyway)
    sed -i -e 's/sepolicy//' selinux-src/policycoreutils/Makefile && \
    # Compile and install SELinux toolchain
    sudo make CFLAGS="-O2 -pipe -fPIC -Wall" -C selinux-src install && \
    # Ubuntu 12.04's flex generates a redundant decl in libqpol
    sed -i -e "/Wwrite-strings/s/,/, '-Wno-redundant-decls',/" setup.py && \
    python setup.py build && \
    sudo python setup.py install

# Try to reduce size of container.
RUN sudo apt-get clean && sudo rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
