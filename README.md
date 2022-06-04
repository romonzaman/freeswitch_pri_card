### freeswitch_pri_card
```
setup information of PRI cards on FreeSWITCH 1.10 with ubuntu 20.04
```

#### Step1: wanpipe library setup

apt-get -y install gcc g++ automake autoconf libtool make libncurses5-dev flex bison patch libtool autoconf linux-headers-$(uname -r) libxml2-dev cmake

cd /usr/src/
wget https://ftp.sangoma.com/linux/current_wanpipe/wanpipe-current.tgz
tar xvfz wanpipe-current.tgz
cd wanpipe-<version>/
make
make install

###### detect board found

> wanrouter hwprobe
-------------------------------
| Wanpipe Hardware Probe Info |
| ------------------------------- |
1 . AFT-A101-SH : SLOT=4 : BUS=8 : IRQ=177 : CPU=A : PORT=1 : HWEC=32 : V=37
Card Cnt: A101-2=1


> wanrouter status

Devices currently active:
wanpipe1
Wanpipe Config:
Device name | Protocol Map | Adapter | IRQ | Slot/IO | If's | CLK | Baud rate |
wanpipe1 | N/A | A101/1D/A102/2D/4/4D/8| 177 | 4 | 1 | N/A | 0 |
Wanrouter Status:
Device name | Protocol | Station | Status |
wanpipe1 | AFT TE1 | N/A | Connected |

#### Step2: sangoma isdn library setup

cd /usr/src
wget https://ftp.sangoma.com/linux/libsng_isdn/libsng_isdn-current.x86_64.tgz
tar xvfz libsng_isdn-current.x86_64.tgz
cd libsng_isdn-<version>.<arch>
make install

#### Step3: Compaile FreeTDM module
```bash
cd /usr/src/freeswitch/
```
modify modules.conf and ensure mod_freetdm is enabled
```bash
mod_freetdm|https://github.com/freeswitch/freetdm.git -b master
```

```bash
./bootstrap
./configure
make
make install
```

#### Step4: create configuration file

> wancfg_fs

- Physical layer
/etc/wanpipe/wanpipeX.conf (x represents each port of your card. For analog cards, you will only see 1 for the entire card)
/etc/wanpipe/wanrouter.rc (The Sangoma driver looks into this file to begin loading each port for your card(s)

- FreeTDM Signaling layer
/usr/local/freeswitch/conf/freetdm.conf (b and d channels)

- FreeSWITCH related files
/usr/local/freeswitch/conf/autoload_configs/freetdm.conf.xml (channel details for your Sangoma card)

 > wanrouter start


#### Step 5: permission fix
```bash
echo 'SUBSYSTEM=="wanpipe", OWNER="www-data", GROUP="www-data", MODE="0660"' > /etc/udev/rules.d/30-wanpipe.rules
```

#### Configuration

nano /etc/freeswitch/autoload_configs/freetdm.conf.xml
```xml
<configuration name="freetdm.conf" description="Freetdm Configuration">
<settings>
<param name="debug" value="0"/>
<!--<param name="hold-music" value="$${moh_uri}"/>-->
<!--<param name="enable-analog-option" value="call-swap"/>-->
<!--<param name="enable-analog-option" value="3-way"/>-->
</settings>
<config_profiles>
<profile name="my_pri_nt_1">
<param name="switchtype" value="euroisdn" />
<param name="interface" value="net"/>
</profile>
</config_profiles>
<sangoma_pri_spans>
<span name="wp1" cfgprofile="my_pri_nt_1">
<param name="dialplan" value="XML"/>
<param name="context" value="public"/>
</span>
</sangoma_pri_spans>
<analog_spans>
</analog_spans>
</configuration>
```

#### troubleshooting

checking if freetdm module get any missing deps.
if require, move files to corret path.
```bash
ldd /usr/lib/freeswitch/mod/mod_freetdm.so
```

#### freeswitch commands
