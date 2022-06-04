### freeswitch_pri_card
```
setup information of PRI cards on FreeSWITCH 1.10 with ubuntu 20.04
```

#### Step1: wanpipe library setup
```bash
apt-get -y install gcc g++ automake autoconf libtool make libncurses5-dev flex bison patch libtool autoconf linux-headers-$(uname -r) libxml2-dev cmake
```

```
cd /usr/src/
wget https://ftp.sangoma.com/linux/current_wanpipe/wanpipe-current.tgz
tar xvfz wanpipe-current.tgz
cd wanpipe-<version>/
make
make install
```


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
```bash
cd /usr/src
wget https://ftp.sangoma.com/linux/libsng_isdn/libsng_isdn-current.x86_64.tgz
tar xvfz libsng_isdn-current.x86_64.tgz
cd libsng_isdn-<version>.<arch>
make install
```

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

###### troubleshooting on compile
nano +1657 src/mod/outoftree/mod_freetdm/mod_freetdm.c
```c
        if (!zstr(dest)) {
-               ftdm_set_string(caller_data.dnis.digits, dest);
+               strcpy(caller_data.dnis.digits, dest);
        }
```

nano +1335 src/mod/outoftree/mod_freetdm/src/ftdm_io.c
```c
                for (i = 0; i < count; i++) {
                        if (strcmp(tokens[i], token)) {
-                               ftdm_copy_string(ftdmchan->tokens[ftdmchan->token_count], tokens[i], sizeof(ftdmchan->tokens[ftdmchan->token_count]));
+                               memcpy(ftdmchan->tokens[ftdmchan->token_count], tokens[i], sizeof(ftdmchan->tokens[ftdmchan->token_count]));
                                ftdmchan->token_count++;
                        }
                }
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
mv /usr/local/freetdm/mod/* /usr/lib/freeswitch/mod/
```

#### freeswitch commands



#### dialplan

-- outbound
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


-- inbound
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

