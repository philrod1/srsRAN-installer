# srsRAN Project
#### This README file is also the script that does all of the things.  You can run it with this command: -
#### curl -L https://raw.githubusercontent.com/philrod1/srsRAN-installer/main/README.md | bash
#### Alternatively, you can click on the 🖉 symbol in Github and copy the raw markdown.
#### You could also run each section by using the copy option

## Start with a fresh install of Ubuntu 22.04
### Tested using ubuntu-22.04.3-server-amd64.iso image
#### Hypervisor details: KVM, qemu-system-x86_64, Q35, BIOS
#### Guest system: 4 cores, 16G RAM, 150G storage, default networking (NAT)
#### Host system: Ubuntu 23.10, Kernel 6.5.0, Intel i7-1370P, 64G RAM


## Setup Useful Aliases

    export myip=`hostname  -I | cut -f1 -d' '`


## Do apt Stuff

    message () { echo -e "\e[1;93m$1\e[0m"; }
    message "Refresh apt"
    sudo apt update
    sudo apt upgrade -y
    sudo add-apt-repository ppa:ettusresearch/uhd
    sudo apt-get update
    sudo apt-get install -y libuhd-dev uhd-host         
    sudo apt install -y openssh-server build-essential cmake libfftw3-dev libmbedtls-dev libboost-program-options-dev libconfig++-dev libsctp-dev libtool autoconf ccache libzmq3-dev


## Install asn1c Compiler

    message "Installing asn1c compiler"
    git clone https://gitlab.eurecom.fr/oai/asn1c.git
    cd asn1c
    git checkout velichkov_s1ap_plus_option_group
    autoreconf -iv
    ./configure
    make -j`nproc`
    sudo make install
    sudo ldconfig


## Build and install srsRAN E2

    message "Cloning the srsRAN-e2 project"
    cd
    git clone https://github.com/openaicellular/srsRAN-e2.git
    message "Fixing the code"
    sed -i '6i #include <cstddef>' srsRAN-e2/srsenb/hdr/ric/e2ap.h
    cd srsRAN-e2
    mkdir build
    export SRS=`realpath .`
    cd build
    message "... cmake"
    cmake ../ -DCMAKE_BUILD_TYPE=RelWithDebInfo \
    -DRIC_GENERATED_E2AP_BINDING_DIR=${SRS}/e2_bindings/E2AP-v01.01 \
    -DRIC_GENERATED_E2SM_KPM_BINDING_DIR=${SRS}/e2_bindings/E2SM-KPM \
    -DRIC_GENERATED_E2SM_GNB_NRT_BINDING_DIR=${SRS}/e2_bindings/E2SM-GNB-NRT
    message "... make"
    make -j`nproc`
    message "... install"
    sudo make install
    message "... load"
    sudo ldconfig
    message "Install configs"
    srsran_install_configs.sh user --force

## Install the applications
