# srsRAN Project
#### This README file is also the script that does all of the things.  You can run it with this command: -
#### curl -L https://raw.githubusercontent.com/philrod1/srsRAN-installer/main/README.md | bash
#### Alternatively, you can click on the 🖉 symbol in Github and copy the raw markdown.
#### You could also run each section by using the copy option

## Run this script after a clean install of the OAIC RIC
#### My RIC installer can be found here: https://github.com/philrod1/oaic-ric-installer
### Tested using ubuntu-20.04.1-legacy-server-amd64.iso image
#### Hypervisor details: KVM, qemu-system-x86_64, Q35, BIOS
#### Guest system: 4 cores, 16G RAM, 150G storage, default networking (NAT)
#### Host system: Ubuntu 23.10, Kernel 6.5.0, Intel i7-1370P, 64G RAM


## Setup Useful Aliases

    export myip=`hostname  -I | cut -f1 -d' '`
    export E2NODE_PORT=5006
    export E2TERM_IP=`sudo kubectl get svc -n ricplt --field-selector metadata.name=service-ricplt-e2term-sctp-alpha -o jsonpath='{.items[0].spec.clusterIP}'`

## Do apt Stuff

    message () { echo -e "\e[1;93m$1\e[0m"; }
    message "Refresh apt"
    sudo add-apt-repository ppa:ettusresearch/uhd
    sudo apt update
    sudo apt upgrade -y     
    sudo apt install -y libuhd-dev libuhd4.1.0 uhd-host libzmq3-dev
    sudo apt install -y pax openssh-server build-essential cmake libfftw3-dev libmbedtls-dev libboost-program-options-dev libconfig++-dev libsctp-dev libtool autoconf ccache
    mkdir $HOME/srs_logs


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


## Build and install srsRAN E2 agent

    message "Cloning the srsRAN-e2 project"
    cd ~/oaic/srsRAN-e2
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
    sudo mkdir -p /root/.config/srsran
    cd /usr/local/share/srsran
    sudo pax -rw -pe -s/.example// . /root/.config/srsran/


## Generate Start Scripts

    message "Generate SRS start scripts"
cat > ~/startEPC.sh <<EOF
#!/bin/bash
sudo srsepc
EOF
    
    cat > ~/startENB.sh <<EOF
#!/bin/bash
    
    sudo srsenb \
    --enb.n_prb=50 \
    --enb.name=enb1 \
    --enb.enb_id=0x19B \
    --rf.device_name=zmq \
    --rf.device_args="fail_on_disconnect=true,tx_port0=tcp://*:2000,rx_port0=tcp://localhost:2001,tx_port1=tcp://*:2100,rx_port1=tcp://localhost:2101,id=enb,base_srate=23.04e6" \
    --ric.agent.remote_ipv4_addr=${E2TERM_IP} \
    --log.all_level=warn \
    --ric.agent.log_level=debug \
    --log.filename=stdout \
    --ric.agent.local_ipv4_addr=${myip} \
    --ric.agent.local_port=${E2NODE_PORT}
EOF
    
    cat > ~/startUE.sh <<EOF
#!/bin/bash
sudo ip netns add ue1
sudo srsue --gw.netns=ue1
EOF


## Start the Services 

    #message "Starting the Services"
    #cd
    #mkdir srs_logs
    #nohup bash startEPC.sh > ~/srs_logs/epc.log 2>&1 & echo $! > ~/srs_logs/epc.pid
    #nohup bash startENB.sh > ~/srs_logs/enb.log 2>&1 & echo $! > ~/srs_logs/enb.pid
    #nohup bash startUE.sh > ~/srs_logs/ue.log 2>&1 & echo $! > ~/srs_logs/ue.pid
    #message "Done."
    
## Alternative to nohup
#### I personally prefer screen to nohup.  I use variations of this command: -
#### screen -S epc /bin/bash -c './startEPC.sh 2>&1 | tee srs_logs/epc.log'
    

## More Information
#### The three services are running in the background using nohup.  The PID of the respective nohup processes are stored in files inside the srs_logs directory.  This information is pretty much useless, as the actual PID of the services will be something like 1-4 greater than the stored PID.  If you want to kill a service, you could simply do `sudo killall srsue` for example.  I might work on a more elegant solution at some point.
