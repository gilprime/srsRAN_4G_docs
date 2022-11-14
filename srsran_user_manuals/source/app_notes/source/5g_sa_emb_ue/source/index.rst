.. Embedded 5G SA UE Application Note

.. _5g_sa_emb_ue_appnote:

Embedded 5G SA UE Application Note
==================================

Introduction
************

This application note shows how the embedded 5G SA UE can be used with a third-party 5G SA network.
In this example, we use the Amari Callbox Classic from Amarisoft to provide the 5G SA network.

The application note starts with a quick overview of the main features of the embedded 5G SA UE
implementation on the RFSoC platform. Moreover, the utilized laboratory setup and the steps to
reproduce it are provided as well (i.e., from configuring the ZCU111 prototyping board as
required, to the list of console commands used to control the system).

Embedded 5G SA UE Overview
**************************

.. image:: .imgs/5g_sa_emb_ue_lab_setup.png
  :align: center
|

Hardware Requirements
---------------------

The figure above shows the utilized laboratory setup, which comprises the following components:

  - **ZCU111 prototyping platform**: hosts the RFSoC device, which will implement the embedded
    5G SA SRS UE. Most PHY-layer functions of the embedded SA UE are FPGA-accelerated.
  - **XM500 daughterboard**: this FMC balun converter board is plugged onto the ZCU111 and
    provides external access to the ADCs/DACs in the RFSoC.
  - **Amari Callbox**: NR SDR-based UE test solution from Amarisoft. It contains both the SA mode
    gNodeb and the 5GC, colocated in the same machine. Furthermore, high-performance PCIe SDR
    cards are implementing the RF front-end of the Callbox.
  - **x86 host #1**: will provide SSH access to the ZCU111 board, in order to control the
    embedded SA SRS UE. Will also display run-time performance metrics.
  - **octoclock**: will provide a shared 10 MHz reference signal between the ZCU111 board and the
    Callbox's RF front-end.

The specific elements utilized in the laboratory setup were chosen as they are widely available,
easily configurable, and user-friendly.

Hardware Setup
--------------

The connection between the different components comprising the laboratory setup is as follows:

  * The RF front-end of the Callbox will be directly cabled with the XM500. Note that the latter
    does not include RF gain/filtering components, but enables a cabled setup via the onboard SMA
    connectors (comes equipped with suitable external filters). Additionally, a common 10 MHz
    reference signal will be shared between them (e.g., octoclock).
  * Both x86 host #1 and the ZCU111 will be connected to the same LAN (Ethernet), which will
    enable the host to access (SSH) the ZCU111 and interact with the embedded ARM (RFSoC).

Embedded 5G SA UE Features and Limitations
******************************************

.. _Features
Features
--------

The current NR SA UE implementation includes the following features:

  * DL bandwidth up to 20 MHz.
  * FPGA-accelerated L1 DSP:

     - Bulk of L1 DL DSP (PSS detection, FFT, channel estimation and PBCH/PDCCH/PDSCH).
     - L1 UL shared channel and OFDM stage (PUSCH, iFFT).
  * Run-time performance metrics provided through the console.

Limitations
-----------

The lack of a complete RF front-end introduces the following limitations:

  * A cabled setup is required, as no gain and/or RF filtering components are included in the
    XM500 daughter-board (beyond those baseline features provided by the HF/LF baluns).
    Consequently, no AGC functionalities are implemented.
  * The center frequencies supported by the specific hardware setup being utilized are
    constrained to the 10-2500 MHz range (e.g., testing has used band *n3* - ARFCN *368500: 1842.5
    MHz DL, 1747.5 MHz UL* - for the NR carrier).
  * Regarding the Tx gains, they need to be carefully fixed, for which we do recommend using the
    settings described in the configuration files provided below.

The embedded 5G SA UE implementation shares a few feature limitations with its x86 counterpart.
That is, while interoperability with a third party gNB is supported, only certain bands (i.e.,
NR configuration parameters) can actually be used. A list of key feature limitations is provided
below for the sake of thoroughness:

  * Only the 15 kHz subcarrier-spacing (SCS) is supported (including the SSB).
  * Signal bandwidth limited to 20 MHz.
  * Only DCI formats 0_0, 1_0, 0_1 and 1_1 are supported.
  * No reference signal measurements.
  * Cell search is skipped by default (fixed PCI and SSB ARFCN values are used for the NR carrier
    instead); it can be enabled for the 52 PRB configuration, but is not supported for 106 PRB.
  * PRACH is precomputed for the 106 PRB configuration (*f_idx* is fixed to *0*).

Building the embedded SA UE
***************************

The embedded 5G SA UE application can be built from the SRS FPGA repository, following the steps
described in the following.

ARM binary
----------

First, you'll need to have a Petalinux build based on the exported hardware configuration files
of the implemented Vivado project for the embedded 5G SA mode UE (you can find the related *.xsa*
file in the code repository; under the *RFdc timestamping IP section in
/lib/src/phy/ue/fpga_ue/RFdc_timestamping/petalinux_files/sa_ue_impl_files*).

The first step towards building the embedded SA UE application is to install the toolchain that
was built via *petalinux-tools*. This file is located at
*/PETALINUX_BUILD_PATH/xilinx-zcu111-2019.2/images/linux*. To install it, use the following
command::

  ./sdk.sh

You will be prompted to specify the toolchain installation path (for instace, use
*/opt/plnx_sdk_rfsoc*). When the installation finishes, set up the following environment
variables::

  . /opt/plnx_sdk_rfsoc/environment-setup-aarch64-xilinx-linux

Then, go to the path where the SRS FPGA repository is cloned locally. Then, run the following
commands, paying special attention to the *cmake* argument (which points to the *toolchain.cmake*
file linked below and for which you will need a local copy)::

  cd srsLTE_FPGA
  mkdir build && cd build
  cmake -DCMAKE_BUILD_TYPE=Release -DUSE_LTE_RATES=ON -DCMAKE_TOOLCHAIN_FILE=~/toolchain.cmake ..
  make -j12 srsue

When the build finishes, you will find the application at */srsue/src* within your local
repository.

  - :download:`toolchain.cmake file to build the UE <toolchain.cmake>`

FPGA bitstream
--------------

The latest implemented bitstream for the embedded 5G SA mode UE can be found in the same location
as the exported hardware configuration files used to build Petalinux (you can find the *.bit*
file in the code repository; under the *RFdc timestamping IP section in
/lib/src/phy/ue/fpga_ue/RFdc_timestamping/petalinux_files/sa_ue_impl_files*).

Configuration
*************

In this example, we are using the following configuration parameters:
  * Band n3:

     - FDD
     - 15 kHz
     - DL ARFCN: 368500 (1842.5 MHz)
     - UL ARFCN: 349500 (1747.5 MHz),
  * 10 MHz and 20 signal bandwidths (52 PRB and 106 PRB, respectively, for both DL and UL)
  * PCI 500
  * Two CORESETs:

     - CORESET0 (interleaved PDCCH, 48 PRB allocation, RB offset = 1 (52 PRB)/28 (106 PRB))
     - UE-specific CORESET (non-interleaved, RB offset = 0)

The next sections will detail how to apply such configuration to both UE and gNB.

Configuration files
-------------------

To reproduce the described laboratory setup, with the described features and limitations, both the
UE and the Amari Callbox need to be properly configured. Specifically, changes must be made to the
*ue.conf* file in the UE side and to the *mme.cfg* and *gnb_nsa.cfg* files in the Callbox side.

All of the modified configuration files have been included as attachments to this App Note. It is
recommended you use these files to avoid errors while changing configs manually. Any configuration
files not included here do not require modification from the default settings.

*UE configuraion file*

  - :download:`embedded 5G SA SRS UE 52 PRB configuration file <ue_52prb.conf>`
  - :download:`embedded 5G SA SRS UE 106 PRB configuration file <ue_106prb.conf>`

*Amari Callbox configuration files*

  - :download:`gNB 52 PRB configuration file <gnb-sa-fpga_52prb.cfg>`
  - :download:`gNB 106 PRB configuration file <gnb-sa-fpga_106prb.cfg>`
  - :download:`MME configuration file (common for both 52 and 106 PRB) <mme.cfg>`

**NOTE:** By default cell search is skipped, but it can be enabled (10 MHz only) through a
parameter in the UE configuration file (by setting *skip_cell_search = false*).

srsUE (ZCU111 board)
--------------------

*Use of an external reference signal in the ZCU111*

The use of an external 10 MHz reference signal ensures the accuracy of the system clock, which
will also be shared with the gNB. In order to enable the use of an external reference in the
ZCU111 board, the following actions are required:

  1. Disconnect the jumper in *J110* to power-off the 12.8 MHz TCXO that is connected by default to
     *CLKin0* of the LMK04208 PLL used to generate the ADC/DAC reference clocks in the ZCU111.
  2. Connect a 10 MHz clock reference to the *J109* SMA port in the ZCU111 (e.g., cabled output
     from octoclock).

.. image:: .imgs/zcu111_J109_J100_config.png
  :align: center
|

Note that some modifications are also required in the software end. Nevertheless, the embedded
SRS UE application is already including them. The full details are provided in the code repository
(see the *RFdc timestamping IP section in /lib/src/phy/ue/fpga_ue/RFdc_timestamping*).

*XM500 port usage*

As per FPGA design (i.e., fixed in the NR SA UE bistream), a specific set of connectors needs to
be used in the XM500 daughter-board, as indicated below:

  - The NR DL signal shall be received from ADC Tile 224, channel 1 (labelled as
    **ADC224_T0_CH1** in the board).
  - The NR UL signal shall be transmitted from DAC Tile 229, channel 3 (labelled as
    **ADC224_T1_CH3** in the board).

Moreover, one of the external DC-2500 MHz low-pass filters (**VLFX-2500+**) shipped alongisde the
XM500 needs to be placed between the Tx cable coming from the gNB and the SMA connector of the ADC
channel used in the XM500, as shown below.

.. image:: .imgs/zcu111_external_filter_detail.png
  :align: center
|

*SD card*

The bitstream and binaries implementing the embedded 5G SA mode UE are hosted in an SD card, which
is organized as detailed below:

  - **BOOT partition**: includes the embedded UE boot image (*BOOT.BIN*), which groups the FPGA
    bistream and boot binaries, the Petalinux Kernel image and the device tree.
  - **rootfs partition**: includes the root file system, which contains the user applications
    (i.e., the embedded SRS UE binary must be copied in this partition).

Build of a customized SD card is out of the scope of this application note. Nevertheless, detailed
instructions on how to do so can be found in the FPGA code repository
(see *lib/src/phy/ue/fpga_ue/srsRAN_RFSoC.md*). A ready to use image of the SD card as used by the
5G SA mode UE in this example is also available.

In case of not having physical access to the SD card in the ZCU111 used in your laboratory setup,
you can copy the the embedded SRS UE files over the network. First, run the following commands in
the ZCU111 console (i.e., the one *SSHing* the board) ::

  mkdir BOOT_mnt
  mount /dev/mmcblk0p1 BOOT_mnt

Then run the following commands in the folder containing your local copy of the embedded SRS UE
*BOOT.BIN* and device tree files (you can find them in the code repository; under the *RFdc
timestamping IP section in
/lib/src/phy/ue/fpga_ue/RFdc_timestamping/petalinux_files/sa_ue_impl_files/BOOT_BIN_files*) ::

  scp BOOT.BIN root@ZCU111_IP_ADDRESS:/home/root/BOOT_mnt/BOOT.BIN
  scp system.dtb root@ZCU111_IP_ADDRESS:/home/root/BOOT_mnt/system.dtb

Finally, run the following commands in the ZCU111 console ::

  sync
  umount BOOT_mnt
  reboot

In the *rootfs* partition we'll need to copy both the embedded SRS UE binary, the UE configuration
file and the *run* script file provided below. You can also do it over the network.

gNB and 5GC (Amari Callbox)
---------------------------

*Shared reference signal with the ZCU111*

Provide a PPS input to the Amari Callbox generated from the same reference signal source (e.g.,
octoclock) used with the ZCU111 (use of *external* sync in the gNB configuration file).

*SDR card and ports usage*

In the utilized laboratory setup (and in accordance to the attached configuration files) it was
employed the SDR card on the third slot (labelled *sdr2* in the gNB configuration file). Moreover,
a single RX RF port and a single TX RF port were used. In the case of the TX port (i.e., DL signal)
the connection passed through the external RF filter of the counterpart receive ADC channel in the
XM500 daugther-board.

Usage
*****

Following configuration, we can run the UE, gNB and 5GC. The following order should be used when
reproducing the described laboratory setup:

1. Callbox initialization
2. MME
3. gNB
4. UE
5. ping
6. iperf

Callbox initialization
----------------------

Properly initializing the Amari Callbox can be conveniently done through a series of scripts
that will make sure that all relevant configuration parameters are set as needed (e.g., CPU
governor). These scripts have been attached to this App note.

  - :download:`set of configuration scripts <callbox_init_scripts.tar.xz>`

All scripts can be executed by a single command::

  sudo ./run_all.sh

Then, make sure that the kernel module managing the SDR cards is properly loaded. To do so, run
the following command in the *trx_sdr-linux-2021-03-17* path::

  sudo ./trx_sdr-linux-2021-03-17/kernel/init.sh

MME
---

*The commands listed below are to be run on the Amari Callbox. In our setup, the LTE MME
version 2020-09-14 was used. Likewise, the TRX SDR Linux kernel module version 2021-03-17
was used.*

Make sure that the *mme.cfg* file is copied in the appropriate config folder and run the
following command in the *ltemme-linux-2020-09-14* path::

  sudo ./ltemme config/mme.cfg

The console output should be similar to::

  LTE MME version 2020-09-14, Copyright (C) 2012-2020 Amarisoft
  This software is licensed to Software Radio Systems (SRS).
  Support and software update available until 2021-10-29.

gNB
---

*The commands listed below are to be run on the Amari Callbox. In our setup, the LTE eNB/gNB
version 2021-03-17 was used.*

Make sure that the *gnb-sa-fpga_52prb.cfg* and *gnb-sa-fpga_106prb.cfg* files are copied in the
appropriate config folder. Then, run the following commands in the * lteenb-linux-2021-03-17* path
to run the 52 PRB (10 MHz) gNB::

  sudo ./lteenb config/gnb-sa-fpga_52prb.cfg

The onsole output should be similar to::

  LTE Base Station version 2021-03-17, Copyright (C) 2012-2021 Amarisoft
  This software is licensed to Software Radio Systems (SRS).
  Support and software update available until 2021-10-29.

  RF0: sample_rate=15.360 MHz dl_freq=1842.500 MHz ul_freq=1747.500 MHz (band n3) dl_ant=1 ul_ant=1
  (enb)

To run the 106 PRB (20 MHz) gNB simply use *gnb-sa-fpga_106prb.cfg* in the command above.

UE and ping
-----------

*The commands listed below are to be run on the zcu111 (i.e., through SSH via host #1). Recall that
besides the binary, you also need to copy in the SD card the *ue.conf*, *install_srsue_drivers.sh*
and *run_ue.sh* files attached in this App Note.*

To run the UE, first we'll need to load the custom srsUE DMA drivers for the ZCU111. This can
be conveniently done through a script that handles the required *insmod* calls, which has been
included attached to this App Note, as well as the drivers per se. Likewise, a script handling
the execution of the embedded 5G SA UE has also been attached.

  - :download:`embedded srsUE DMA drivers installation script <install_srsue_drivers.sh>`
  - :download:`srsUE DMA drivers <srsue_dma_kernel_drivers.tar.xz>`
  - :download:`embedded 5G SA UE execution script <run_ue.sh>`

To load the srsUE drivers use the following command::

  ./install_srsue_drivers.sh

Later the 52 PRB embedded srsUE will be executed using the following command::

  ./run_ue 52

Once the UE has been initialised you should see an output similar to the following::

  Reading configuration file ue.conf...
  WARNING: cpu0 scaling governor is not set to performance mode. Realtime processing could be compromised. Consider setting it to performance mode before running the application.

  Built in Release mode using commit 827b5c300 on branch merge_dev_june22.

  Opening 1 channels in RF device=default with args=clock=external
  Supported RF device list: RFdc file
  Trying to open RF device 'RFdc'
  metal: info:      Registered shmem provider linux_shm.
  metal: info:      Registered shmem provider ion.reserved.
  metal: info:      Registered shmem provider ion.ion_system_contig_heap.
  metal: info:      Registered shmem provider ion.ion_system_heap.
  Configuring LMK04208 to use external clock source
  tLMX configured
  RF device 'RFdc' successfully opened

  FPGA bitstream built on 0000/00/00 00:00:00:00 using commit 00000000
  Setting manual TX/RX offset to 65 samples
  Waiting PHY to initialize ... done!

To run the 106 PRB srsUE simply use *106* as input parameter in the command above.

Once the FPGA has correctly synchronized to the selected cell you should see a similar console
output during the attach procedure::

  Attaching UE...
  Random Access Transmission: prach_occasion=0, preamble_index=0, ra-rnti=0xf, tti=8811
  Random Access Complete.     c-rnti=0x4601, ta=1
  RRC Connected
  RRC NR reconfiguration successful.
  PDU Session Establishment successful. IP: 192.168.4.2
  RRC NR reconfiguration successful.

Note that an IP address is provided once the PDU session establishment is succesfully completed.
You can either start a ping from the UE (SSH session to ZCU111) or ping the UE from another
session::

  ping 192.168.4.2

Similar console outputs should then be produced::

  PING 192.168.4.2 (192.168.4.2): 56 data bytes
  64 bytes from 192.168.4.2: seq=0 ttl=64 time=33.942 ms
  64 bytes from 192.168.4.2: seq=1 ttl=64 time=113.814 ms
  64 bytes from 192.168.4.2: seq=2 ttl=64 time=33.654 ms
  64 bytes from 192.168.4.2: seq=3 ttl=64 time=33.607 ms

iperf
-----

To run an UL UDP iperf test, the first step will be starting a server in the Amari Callbox (note
that 192.168.4.1 it's the MME IP address)::

  iperf3 -s -B 192.168.4.1 -p 5003

Then we will run a client in the embedded ARM of the RFSoC (SSH to ZCU111)::

  iperf3 -c 192.168.4.1 -p 5003 -t 30 -b 40M -u

The example above targets the 52 PRB (10 MHz) SA UE configuration and, hence, the requested traffic
data-rate is limited to 40 Mbps (*-b 40M*). For 106 PRB (20 MHz) it can be doubled to 80  Mbps (*-b
80M*).

Similarly, to run a DL UDP iperf test, first a server will be started in the UE (RFSoC - note
that 192.168.4.2 is the IP address assigned to the UE by the network)::

  iperf3 -s -B 192.168.4.2 -p 5004

Then, a client will be executed in the Amari Callbox::

  iperf3 -c 192.168.4.1 -p 5003 -t 30 -b 40M -u

As before, the command above is targeting a 52 PRB (10 MHz) SA UE configuration. Hence, for 106 PRB
(20 MHz) the data-rate can be doubled too (*-b 80M*).

Finally, it is worth mentioning that by typing **t** in the console of the embedded SRS NR SA UE,
after the attach procedure is succesfully completed, the periodical display of relevant traffic
metrics as part of the displayed outputs will be enabled (below some UL iperf metrics for a 106 PRB
configuration are shown as an example)::

  Enter t to stop trace.
  ---------Signal-----------|-----------------DL-----------------|-----------UL-----------
  rat  pci  rsrp   pl   cfo | mcs  snr  iter  brate  bler  ta_us | mcs   buff  brate  bler
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   381k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   379k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   377k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    31k    0%    0.0 |  28   379k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   379k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   377k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   376k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    31k    0%    0.0 |  28   372k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   378k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   372k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   372k    75M    0%
  ---------Signal-----------|-----------------DL-----------------|-----------UL-----------
  rat  pci  rsrp   pl   cfo | mcs  snr  iter  brate  bler  ta_us | mcs   buff  brate  bler
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   372k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    31k    0%    0.0 |  28   373k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   375k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   375k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   373k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    31k    0%    0.0 |  28   378k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   380k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   380k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   379k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    30k    0%    0.0 |  28   380k    75M    0%
   nr  500     0    0   0.0 |  27    0   0.0    31k    0%    0.0 |  28   363k    75M    0%

**NOTE:** Testing of the 10 MHz (52 PRB) configuration showed that slightly higher DL throughput
can be achieved by forcing use of DCI formats 0_0 and 1_0 (set *dci_0_1_and_1_1: false* in
*gnb-sa-fpga_52prb.cfg*).

Troubleshooting and known issues
********************************

To ensure the correct behaviour of the system it is recommended that the utilized laboratory setup
is as described in this App Note. It is also very important to validate that matching configuration
parameters and files are used when executing both the gNB and UE applications. In more detail, the
following pairings are required:
  - for the 52 PRB SA network configuration use *ue_52prb.conf* (through *./run_ue 52*) and
    *gnb-sa-fpga_52prb.cfg*.
  - for the 106 PRB SA network configuration use *ue_106prb.conf* (through *./run_ue 106*) and
    *gnb-sa-fpga_106prb.cfg*.

Also, cell search is not supported under the 106 PRB configuration (i.e., make sure that the UE
configuration properly sets *skip_cell_search = true*).

It is a known issue that under heavy DL traffic conditions the DMA might enter into a deadlock from
which it cannot recover. In more detail, there seems to be a bug in either the Xilinx DMA IP core,
its related driver or the AXI interconnect management, which fails to clear a *complete bit* in the
scatter-gather descriptor memory (meant to flag a finished transaction). Therefore, the DMA IP core
will be lead to an invalid state when a new operation is requiring to use the same descriptor
memory position. From the user perspective, on-screen metrics will display *NaN* (in the *dmesg*
log there will be a message similar to *xilinx-vdma a0090000.dma: Channel 00000000eb67672b has
errors 100, cdr 77cc3200 tdr 77cc3200*). In such occurrences, it is required to stop the UE and then
unload and reload back both Xilinx DMA and SRS UE kernel drivers. A script automating the latter has
been included as attachment to this App Note.

- :download:`driver reloading script <reload_dma_drivers.sh>`

The script can be executed through the following command::

./reload_dma_drivers.sh
