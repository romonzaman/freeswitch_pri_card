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

#### Step2: sangoma isdn library setup
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

#### Step3: Compaile FreeTDM module
```bash
cd /usr/src/freeswitch/
```

```bash
sed -i 's/#mod_freetdm/mod_freetdm/g' modules.conf

./bootstrap || ./rebootstrap.sh
./configure
make

```

###### we need to modify two file so that compile works
nano +71 src/mod/outoftree/mod_freetdm/src/include/ftdm_os.h
```c
/* #define ftdm_set_string(x,y) {memcpy(x, y, (sizeof(x)>sizeof(y)?sizeof(y):sizeof(x))-1); x[(sizeof(x)>sizeof(y)?sizeof(y):sizeof(x))-1] = 0;} */
#define ftdm_set_string(x,y) strcpy(x, y)

```

nano +136 src/mod/outoftree/mod_freetdm/src/include/private/ftdm_core.h
```c
/* #define ftdm_set_string(x,y) {memcpy(x, y, (sizeof(x)>sizeof(y)?sizeof(y):sizeof(x))-1); x[(sizeof(x)>sizeof(y)?sizeof(y):sizeof(x))-1] = 0;} */
#define ftdm_set_string(x,y) strcpy(x, y)

```


```
cp src/mod/outoftree/mod_freetdm/mod_freetdm/mod_freetdm.c src/mod/outoftree/mod_freetdm/mod_freetdm/mod_freetdm.c.bk
sed -i 's/ftdm_set_string(caller_data.dnis.digits, dest);/strcpy(caller_data.dnis.digits, dest);/g' src/mod/outoftree/mod_freetdm/mod_freetdm/mod_freetdm.c

cp src/mod/outoftree/mod_freetdm/src/ftdm_io.c src/mod/outoftree/mod_freetdm/src/ftdm_io.c.bk
sed -i 's/ftdm_copy_string(ftdmchan->tokens\[ftdmchan->token_count/memcpy(ftdmchan->tokens\[ftdmchan->token_count/g' src/mod/outoftree/mod_freetdm/src/ftdm_io.c

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

