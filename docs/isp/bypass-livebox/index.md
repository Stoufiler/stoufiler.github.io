# How to replace Orange LIVEBOX by UDM PRO SE ?

## What do you need ?

- 1 [UDM PRO SE](https://eu.store.ui.com/products/dream-machine-se){target=_blank}
- 1 [LEOX GPON STICK LXT-010S-H](https://www.leolabs.pl/lxt-010s-h.html){target=_blank}
- 1 [SC-APC / SC-UPC Cable](https://amzn.to/3lh8xCA){target=_blank}

You can order the GPON on [xbest.pl](http://xbest.pl/index.php?p4878,sfp-gpon-stick-leox-lxt-010s-h-1-25-2-5g-sm-sc-20km-tx1310-rx1490-ddm-class-b){target=_blank},
 it costs me around 90€ with delivery

## STEP 1 : Create configuration network

On the UDM PRO SE :

- Create a new network named : `CONF LEOX`
- Uncheck Auto-Scale Network
- Set Host Address to : `192.168.100.2`
- Set Netmask to `29` to allow 5 IPs
- Set DHCP Mode to : `DHCP Server`
- Set DHCP Rans to :
  - Start : `192.168.100.2`
  - Stop : `192.168.100.6`

<figure markdown>
  [![Step 1](screenshots/network-configuration.png){: style="width:400px"}](screenshots/network-configuration.png)
</figure>

## STEP 2 : Connect the GPON Adapter for configuration

!!! warning
    GPON should be plugged into LAN port (PORT 11)

On the UDM PRO SE :

- Set Port Profile of GPON to network freshly created : `CONF LEOX`
- Apply changes

<figure markdown>
[![Step 2](screenshots/port-emplacement.png){: style="width:400px"}](screenshots/port-emplacement.png)
</figure>

## STEP 3 : Allow your computer to access GPON configuration

!!! info inline end
    Because we add a DHCP Server on this network, your computer will
    automatically receive an IP Address on the same subnet as GPON

In order to be able to communicate with GPON,
you need to be on the same subnet as GPON

On the UDM PRO SE :

- Set Port Profile of your computer to `CONF LEOX` port
- Apply changes

<figure markdown>
[![Step 3](screenshots/computer-vlan-gpon.png){: style="width:400px"}](screenshots/computer-vlan-gpon.png)
</figure>

## STEP 4 : Upgrade GPON to latest version

- Go to [http://192.168.100.1/upgrade.asp](http://192.168.100.1/upgrade.asp)
- Fill the basic authentication with these values :

  - **Username :** `leox`
  - **Password :** `leolabs_7`

- [Download the firmware](firmware/V3.3.4L4rc4)
- Select the firmware on your laptop
- Click on `Upgrade` button
- Confirm you want to upgrade

<figure markdown>
[![Step 4](screenshots/upgrade-gpon.png){: style="width:400px"}](screenshots/upgrade-gpon.png)
</figure>

## STEP 5 : Verify that version is latest one

Once you upgrade to latest version, you should verify that

- Go to [http://192.168.100.1/status.asp](http://192.168.100.1/status.asp)
- Firmware version should be : `V3.3.4L4rc4`

<figure markdown>
[![Step 5](screenshots/check-version.png){: style="width:400px"}](screenshots/check-version.png)
</figure>

## STEP 6 : Connect to GPON via telnet

On the UDM PRO SE

- Install telnet package

```bash
apt update
apt install telnet
```

- Initiate a telnet connection to GPON

```bash
telnet 192.168.100.1
Trying 192.168.100.1...
Connected to 192.168.100.1.
Escape character is '^]'.
LXT-010S-H login: leox
Password: leolabs_7
#
```

You are now connected into the GPON

## STEP 7 : Get informations from Livebox using and Open Source tool

- Download this tool : [LiveboxMonitor](https://github.com/p-dor/LiveboxMonitor)
- Execute it
- Fill IP, Username and Password
- Wait at least 1 minute for fetching all data
- Click on tab `Stats/Info Livebox`
- Click on `Export...` button

<figure markdown>
[![Step 7](screenshots/LiveboxMonitor.png){: style="width:400px"}](screenshots/LiveboxMonitor.png)
</figure>

- Open exported file and follow STEP 8

## STEP 8 : Configure the GPON

- Go back to GPON configuration
- Write following lines according to your data

```bash
flash set GPON_PLOAM_PASSWD DEFAULT012
flash set OMCI_TM_OPT 0
flash set OMCI_OLT_MODE 1
flash set GPON_SN SMBSXXXXX             # Numéro de série
flash set PON_VENDOR_ID SMBS            # ID Vendeur
flash set HW_HWVER SMBSXXXXXXX          # Version Matériel
flash set OMCI_SW_VER1 SAHEXXXX         # Version 0 Logiciel ONT
flash set OMCI_SW_VER2 SAHEXXXX         # Version 1 Logiciel ONT
```

- OK perfect, verify GPON is available

```bash
diag gpon get onu-state
# IT SHOULD RETURN 05
gpon get onu-state
ONU state: Operation State(O5)
```

!!! info
    - If you have `01` for `Operation State` please check your cable
    - If you have `05` for `Operation State` it seems to be OK we can continue

Now it's time to find the `OltVendorId`

```bash
omcicli mib get 131
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
OltG
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
=================================
EntityId: 0x00
OltVendorId: ALCL
EquipId:
Version:
ToDInfo:
 Sequence number of GEM superframe: 0x0
 Timestamp: secs 0, nanosecs 0
=================================
```

You should have something in `OltVendorId`,
if you have nothing please check previous steps.

Ok so now my `OltVendorId` is `ALCL` which is Alcatel

Tell the GPON that `OMCC_VER` is `128`

```bash
flash set OMCC_VER 128
```

| OltVendorID  | OMCC_VER  |
|:-----------: |:--------: |
| ALCL         | 128       |
| HWTC         | 135       |

!!! success "GPON Configuration is done"

## STEP 9 : Configure the UDM PRO SE

!!! info "Be sure to running `Unifi Network` version >= 7.4.145"

- Go to UDM PRO SE -> Network -> Settings -> Internet
- Select second internet source

<figure markdown>
  [![Step 9.1](screenshots/9.1.png){: style="width:400px"}](screenshots/9.1.png)
</figure>

!!! warning "Fill everything I mentionned"

- Give a name : `LEOX GPON`
- Set expected ISP Speeds : `Download 300Mbps | Upload 300Mbps`
- Enable and set VLAN ID : `832`
- Enable and set MAC Address Clone : `XX.XX.XX.XX.XX.XX` _# LIVEBOX MAC ADDRESS_
- DHCP Client Options :
  - 60 : `sagem`
  - 77 : `FSVDSL_livebox.Internet.softathome.Livebox5` _# If
you have Livebox 3 replace `5` by `3`_
  - 90 : Calculate it by using this
  [jsfiddle](https://jsfiddle.net/kgersen/3mnsc6wy/){target=_blank} ;
  Let `Salt` and `Byte` to default
  <iframe width="100%" height="300"
  src="//jsfiddle.net/kgersen/3mnsc6wy/embedded/result/dark/"></iframe>

- DHCP CoS : `6`
- IPv4 Connection : `DHCPv4`
- IPv6 Connection : `Disabled`

Apply your changes

You should optain something like this :

<figure markdown>
  [![Step 9.2](screenshots/9.2.png){: style="width:400px"}](screenshots/9.2.png)
</figure>

## STEP 10 : Test in production

- Move the GPON from port `11` to port `10`
- Wait a bit
- You should see an IP address on Network -> Settings -> Internet :

<figure markdown>
  [![Step 10](screenshots/10.png){: style="width:400px"}](screenshots/10.png)
</figure>

!!! success "You are now connected to internet without using Orange Livebox anymore"

!!! Warning
    If you have a private IP like `172.16.152.14` there is a misconfiguration.
    Check all steps and try again
