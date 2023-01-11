### freeswitch_pri_card
```
setup information of PRI cards on FreeSWITCH 1.10 with ubuntu 20.04
```

#### Step1: dependancy, wanpipe library, sangoma isdn and pri library setup, dahdi driver
```
echo "#### installing\n"
echo "####"
apt-get update -y
apt-get -y install gcc g++ automake autoconf libtool make libncurses5-dev flex bison patch libtool autoconf linux-headers-$(uname -r) libxml2-dev cmake mlocate

```


```
lspci -n

dahdi_hardware

dahdi_cfg -v

```

## Step 2: DAHDI card setup


### Digium TE12x Single-Span T1/E1 Card

##### with sphreaknet script
```
cd /usr/src/
wget http://downloads.digium.com/pub/telephony/dahdi-linux-complete/dahdi-linux-complete-current.tar.gz
tar -xvzf dahdi-linux-complete-current.tar.gz
mv dahdi-linux-complete-*/ dahdi-linux-complete/
cd dahdi-linux-complete
make
make install 
make install-config

```

```
cd /usr/src
wget https://docs.phreaknet.org/script/phreaknet.sh
chmod +x phreaknet.sh
./phreaknet.sh make

```

```
phreaknet dahdi

```

##### with digium documents

```
cd /usr/src

wget https://downloads.asterisk.org/pub/telephony/libpri/libpri-current.tar.gz
tar -xvzf libpri-current.tar.gz
mv libpri-*/ libpri/
cd libpri/
make
make install

cd /usr/src/
wget https://downloads.asterisk.org/pub/telephony/dahdi-linux-complete/dahdi-linux-complete-current.tar.gz
tar -xvzf openvox_dahdi-linux-complete-current.tar.gz
mv dahdi-linux-complete-*/ dahdi-linux-complete/
cd dahdi-linux-complete
make
make install 
make config

```

#### Step3: Compaile Freeswitch

```bash
cd /usr/src/freeswitch/
sed -i '/#mod_freetdm/d' modules.conf
echo "" >> modules.conf
echo "mod_freetdm|https://github.com/romonzaman/freetdm.git -b master" >> modules.conf
```


```bash
#./bootstrap || ./rebootstrap.sh
./configure
make
make install

```

###### 

```bash

cp /usr/local/freetdm/mod/* /usr/lib/freeswitch/mod/
cp /usr/local/freetdm/mod/* /usr/local/freeswitch/mod/
cp /usr/local/freeswitch/mod/mod_freetdm.so /usr/lib/freeswitch/mod/
cp /usr/lib64/libsng_isdn.so* /lib/x86_64-linux-gnu/

```

#### Step 4: permission fix
```bash
sed -i 's/asterisk/www-data/g' /etc/udev/rules.d/dahdi.rules

```

#### Configuration
```bash
cat  <<EOT > /etc/freeswitch/autoload_configs/freetdm.conf.xml
<configuration name="freetdm.conf" description="Freetdm Configuration">
<settings>
<param name="debug" value="0"/>
<!--<param name="hold-music" value="$${moh_uri}"/>-->
<!--<param name="enable-analog-option" value="call-swap"/>-->
<!--<param name="enable-analog-option" value="3-way"/>-->
</settings>

<config_profiles>
</config_profiles>

<sangoma_pri_spans>
</sangoma_pri_spans>

<analog_spans>
</analog_spans>

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

</configuration>
EOT

cat  <<EOT > /etc/freeswitch/freetdm.conf
[span zt wp1]
name=wp1
trunk_type => E1
group=wp1
b-channel => 1-15
b-channel => 17-31
d-channel => 16
EOT


```

#### reboot server
```bash
reboot -f

```


load freetd module
> fs_cli
```
load mod_freetdm

```

```
ftdm list

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


####### test
###
```
cd /usr/local/src/
wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-17-current.tar.gz
wget https://downloads.asterisk.org/pub/telephony/dahdi-linux-complete/dahdi-linux-complete-3.1.0+3.1.0.tar.gz
wget https://downloads.asterisk.org/pub/telephony/libpri/libpri-1.6.0.tar.gz

git clone -b next https://github.com/asterisk/dahdi-tools dahdi-tools
git clone -b next https://github.com/asterisk/dahdi-linux dahdi-linux
```

#Asterisk Dependencies
```
apt -y install git curl wget libnewt-dev libssl-dev libncurses5-dev libsqlite3-dev build-essential libjansson-dev libxml2-dev uuid-dev autoconf libedit-dev
apt install -y libtool

cd /usr/local/src/
tar -zxvf dahdi-linux-complete-3.1.0+3.1.0.tar.gz
cp dahdi-linux-complete-3.1.0+3.1.0/linux/drivers/dahdi/firmware/dahdi-fw*.tar.gz dahdi-linux/drivers/dahdi/firmware/
cd dahdi-linux
make
make install
cd ../dahdi-tools
./bootstrap.sh
libtoolize --force
aclocal
autoheader
automake --force-missing --add-missing
autoconf
./configure
make
make install
cd ../dahdi-linux-complete-3.1.0+3.1.0
make install-config
#check dahdi driver is loaded
lsmod |grep dahdi
#If driver is not loaded/shown
modprobe wct4xxp
#Now Dahdi driver should be displayed in following command
lsmod |grep dahdi
#enable dahdi
systemctl enable dahdi
systemctl start dahdi
systemctl status dahdi
dahdi_genconf -vv
dahdi_cfg -vv
```


### CentOS 6
```
yum update -y
sed -i s/SELINUX=enforcing/SELINUX=disabled/g /etc/selinux/config
reboot

yum install -y make wget openssl-devel ncurses-devel  newt-devel libxml2-devel kernel-devel gcc gcc-c++ sqlite-devel libuuid-devel
```

Download the source tarballs. These commands will get the current release of DAHDI 2.6, libpri 1.4 and Asterisk 11.
```
cd /usr/src/
wget https://downloads.asterisk.org/pub/telephony/dahdi-linux-complete/dahdi-linux-complete-2.6.2+2.6.2.tar.gz
tar -zxvf dahdi-linux-complete-2.6.2+2.6.2.tar.gz
cd dahdi-linux-complete-2.6.2+2.6.2
make
make install
make install-config

/etc/init.d/dahdi status
/etc/init.d/dahdi start

dahdi_genconf -vv
dahdi_cfg -vv


wget https://downloads.asterisk.org/pub/telephony/libpri/old/libpri-1.4.15.tar.gz
tar-xvzf libpri-1.4.15.tar.gz
cd libpri-1.4.15
make
make install


```
