### freeswitch_pri_card
```
setup information of PRI cards on FreeSWITCH 1.10 with ubuntu 20.04
```

#### Step1: wanpipe library setup
```bash
apt-get -y install gcc g++ automake autoconf libtool make libncurses5-dev flex bison patch libtool autoconf linux-headers-$(uname -r) libxml2-dev cmake mlocate
```

```
-- after extract, there will directory based on current version
-- we need to enter that directory and build
```
```
cd /usr/src/
wget https://ftp.sangoma.com/linux/current_wanpipe/wanpipe-current.tgz
tar xvfz wanpipe-current.tgz
mv wanpipe-*/ wanpipe
cd wanpipe/
make
make install

```


###### detect board found

> wanrouter hwprobe
```
-------------------------------
| Wanpipe Hardware Probe Info |
| ------------------------------- |
1 . AFT-A101-SH : SLOT=4 : BUS=8 : IRQ=177 : CPU=A : PORT=1 : HWEC=32 : V=37
Card Cnt: A101-2=1
```

> wanrouter status
```
Devices currently active:
wanpipe1
Wanpipe Config:
Device name | Protocol Map | Adapter | IRQ | Slot/IO | If's | CLK | Baud rate |
wanpipe1 | N/A | A101/1D/A102/2D/4/4D/8| 177 | 4 | 1 | N/A | 0 |
Wanrouter Status:
Device name | Protocol | Station | Status |
wanpipe1 | AFT TE1 | N/A | Connected |
```

#### Step2: sangoma isdn and pri library setup
```
-- after extract, there will directory based on current version
-- we need to enter that directory and build
```

```bash
cd /usr/src
wget https://ftp.sangoma.com/linux/libsng_isdn/libsng_isdn-current.x86_64.tgz
tar xvfz libsng_isdn-current.x86_64.tgz
mv libsng_isdn-*/ libsng_isdn/
cd libsng_isdn
make install

wget https://raw.githubusercontent.com/romonzaman/freeswitch_pri_card/main/ssi.x
mv ssi.x /usr/include/sng_isdn/

```

```bash
cd /usr/src
wget https://downloads.asterisk.org/pub/telephony/libpri/libpri-current.tar.gz
tar -xvzf libpri-current.tar.gz
mv libpri-*/ libpri/
cd libpri/
make
make install

```

## DAHDI card setup
#### openvox D130

```
cd /usr/src/
wget https://www.openvox.cn/pub/drivers/dahdi-linux-complete/openvox_dahdi-linux-complete-current.tar.gz
tar -xvzf openvox_dahdi-linux-complete-current.tar.gz
mv dahdi-linux-complete-*/ dahdi-linux-complete/
cd dahdi-linux-complete
make
make install 
make install-config

```

```
dahdi_hardware
```

```
modprobe dahdi
modprobe opvxd115
dahdi_genconf
```

By running "dahdi_genconf", it will generate /etc/dahdi/system.conf and etc/asterisk/dahdi-channels.conf automatically.


A part of system.conf which is one of the basic channel configuration files is displayed.
```
 # Span 1: D115/D130/0/1 "D115/D130 (E1|T1) Card 0 Span 1" (MASTER)
 span=1,1,0,ccs,hdb3,crc4
 # termtype: te
 bchan=1-15,17-31
 dchan=16
 #echocanceller=mg2,1-15,17-31
 # Global data
 loadzone        = us
 defaultzone     = us
```

> dahdi_cfg -v
```
[root@localhost ~]# dahdi_cfg -v DAHDI Tools Version - 2.6.1
DAHDI Version: 2.6.1 Echo Canceller(s): Configuration ======================
SPAN 1: CCS/HDB3 Build-out:
31 channels to configure.

Setting echocan for channel 1 to none
Setting echocan for channel 2 to none
Setting echocan for channel 3 to none
....
....
Setting echocan for channel 30 to none
Setting echocan for channel 31 to none
```

#### Step3: Compaile Freeswitch
```bash
cd /usr/src/freeswitch/
sed -i 's/#mod_freetdm/mod_freetdm/g' modules.conf
./bootstrap || ./rebootstrap.sh
./configure
make
make install

```

###### compile will fail. we will do patch now to make compile ok

```bash

cp src/mod/outoftree/mod_freetdm/mod_freetdm/mod_freetdm.c src/mod/outoftree/mod_freetdm/mod_freetdm/mod_freetdm.c.bk
sed -i 's/ftdm_set_string(caller_data.dnis.digits, dest);/strcpy(caller_data.dnis.digits, dest);/g' src/mod/outoftree/mod_freetdm/mod_freetdm/mod_freetdm.c

cp src/mod/outoftree/mod_freetdm/src/ftdm_io.c src/mod/outoftree/mod_freetdm/src/ftdm_io.c.bk
sed -i 's/ftdm_copy_string(ftdmchan->tokens\[ftdmchan->token_count/memcpy(ftdmchan->tokens\[ftdmchan->token_count/g' src/mod/outoftree/mod_freetdm/src/ftdm_io.c

cd src/mod/outoftree/mod_freetdm/
./configure --with-libpri
cd ../../../../

sed -i  's/snprintf(caller_data->aniII, 5/snprintf(caller_data->aniII, sizeof(caller_data->aniII)/g' src/mod/outoftree/mod_freetdm/src/ftmod/ftmod_libpri/ftmod_libpri.c

./bootstrap || ./rebootstrap.sh
./configure

make
make install

cp /usr/local/freetdm/mod/* /usr/lib/freeswitch/mod/
cp /usr/local/freeswitch/mod/mod_freetdm.so /usr/lib/freeswitch/mod/
cp /usr/lib64/libsng_isdn.so* /lib/x86_64-linux-gnu/

```

#### Step4: create configuration file

> wancfg_fs

- this will create configuration files.
  - **Physical layer**
/etc/wanpipe/wanpipe1.conf
/etc/wanpipe/wanrouter.rc

  - **FreeTDM Signaling layer**
/etc/freeswitch/freetdm.conf (b and d channels)

  - **FreeSWITCH related files**
/etc/freeswitch/autoload_configs/freetdm.conf.xml (channel details for your Sangoma card)

 > wanrouter start


#### Step 5: permission fix
```bash
echo 'SUBSYSTEM=="wanpipe", OWNER="www-data", GROUP="www-data", MODE="0660"' > /etc/udev/rules.d/30-wanpipe.rules
```

#### Configuration

cat /etc/freeswitch/autoload_configs/freetdm.conf.xml
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

load freetd module
> fs_cli
```
load mod_freetdm

```

#### troubleshooting

1) mod_freetdm.so file location is /usr/lib/freeswitch/mod/mod_freetdm.so
if it is other place , move to this location

checking if freetdm module get any missing deps.
if require, move files to corret path.
```bash
	mv /usr/local/freetdm/mod/* /usr/lib/freeswitch/mod/
```

2) mod_freetdm failed to open span card.
   > reboot -f

#### freeswitch commands
> load mod_freetdm

> reload mod_freetdm

> module_exists mod_freetdm

#### dialplan

-- Inbound
```xml
<extension name="freetdm-wp1-55" continue="false" uuid="b9e01e26-53b5-46fd-893e-08ffd7f0600d">
	<condition field="${freetdm_span_name}" expression="wp1"/>
	<condition field="destination_number" expression="^55$">
		<action application="export" data="call_direction=inbound" inline="true"/>
		<action application="set" data="domain_uuid=ac2322ec-52aa-4b90-82c9-6f571a02b09f" inline="true"/>
		<action application="set" data="domain_name=192.168.1.118" inline="true"/>
		<action application="set" data="channel_number=${freetdm_chan_number}"/>
		<action application="info" data=""/>
		<action application="pre_answer" data=""/>
		<action application="set" data="instant_ringback=true"/>
		<action application="set" data="ringback=%(2000,4000,440.0,480.0)"/>
		<action application="sleep" data="1000"/>
		<action application="answer" data=""/>
		<action application="transfer" data="55  XML 192.168.1.118"/>
	</condition>
</extension>
```


-- outbound
```xml
<extension name="freetdm.d11" continue="false" uuid="7a32cba3-e312-4adb-91aa-11b21282d1cf">
	<condition field="${user_exists}" expression="false"/>
	<condition field="destination_number" expression="^(\d{11})$">
		<action application="set" data="sip_h_X-accountcode=${accountcode}"/>
		<action application="export" data="call_direction=outbound"/>
		<action application="unset" data="call_timeout"/>
		<action application="set" data="hangup_after_bridge=true"/>
		<action application="set" data="effective_caller_id_name=${outbound_caller_id_name}"/>
		<action application="set" data="effective_caller_id_number=${outbound_caller_id_number}"/>
		<action application="set" data="inherit_codec=true"/>
		<action application="set" data="ignore_display_updates=true"/>
		<action application="set" data="callee_id_number=$1"/>
		<action application="set" data="continue_on_fail=1,2,3,6,18,21,27,28,31,34,38,41,42,44,58,88,111,403,501,602,607"/>
		<action application="bridge" data="freetdm/1/a/$1"/>
	</condition>
</extension>
```

### fusionpnx modules

- ensure freetdm module is setup as true for default. so that it autoload after restart







```bash
cd /usr/src/freeswitch/

cd src/mod/outoftree/mod_freetdm/
./configure --with-libpri

sed -i  's/snprintf(caller_data->aniII, 5/snprintf(caller_data->aniII, sizeof(caller_data->aniII)/g' src/mod/outoftree/mod_freetdm/src/ftmod/ftmod_libpri/ftmod_libpri.c

cd ../../../../

make
make install

cp /usr/local/freetdm/mod/* /usr/lib/freeswitch/mod/
cp /usr/local/freeswitch/mod/mod_freetdm.so /usr/lib/freeswitch/mod/

```

cat /etc/freeswitch/autoload_configs/freetdm.conf.xml
```xml
<configuration name="freetdm.conf" description="Freetdm Configuration">

....

<libpri_spans>
<span name="wp1">
<param name="node" value="cpe"/>
<param name="switch" value="euroisdn"/>
<param name="opts" value="none"/>
<param name="dp" value="unknown"/>
<param name="debug" value="all"/>
<param name="dialplan" value="XML"/>
<param name="context" value="public"/>
</span>
</libpri_spans>

...

```

nano /etc/freeswitch/freetdm.conf
```
[span zt wp1]
trunk_type => e1
group=1
b-channel => 1:1-15
b-channel => 1:17-31
d-channel => 1:16
```