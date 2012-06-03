kalua - build mesh-networks _without_ pain
==========================================

* community: http://wireless.subsignal.org
* monitoring: http://intercity-vpn.de/networks/dhfleesensee/
* documentation: [API](http://wireless.subsignal.org/index.php?title=Firmware-Dokumentation_API)


needing support?
join the [club](http://blog.maschinenraum.tk) or ask for [consulting](http://bittorf-wireless.de)


how to build this from scratch on a debian server
-------------------------------------------------

- to avoid problems in openwrt and kernel config we suggest to take the changes manually instead of copying config giles

	# be root user
	apt-get update
	LIST="build-essential libncurses5-dev m4 flex git git-core zlib1g-dev unzip subversion gawk python libssl-dev quilt screen"
	for PACKAGE in $LIST; do apt-get -y install $PACKAGE; done

	# now login as non-root user
	git clone git://nbd.name/openwrt.git
	git clone git://nbd.name/packages.git
	cd openwrt
	git clone git://github.com/andibraeu/weimarnetz.git
	
	make menuconfig				# simply select exit, (just for init)
	make package/symlinks
	
	weimarnetz/openwrt-build/mybuild.sh initial_settings	#copy some files
	# start from here, if you only update vour build environment
	weimarnetz/openwrt-build/mybuild.sh gitpull
	./scripts/feeds update -a		# update openwrt feeds (luci, x-wrt and packages) 
	
	echo "TP-LINK TL-WR1043ND">> KALUA_HARDWARE
	make menuconfig
<pre>
	==> Target System ---> Atheros AR7xxx/AR9xxx
	==> Subtarget ---> Generic
	==> Target Profile ---> TP-LINK TL-WR1043N/ND
</pre>	
make kernel_menuconfig

<pre>
	==> Device Drivers ---> [*] Staging Drivers ---> [*] Compressed RAM block device support
</pre>
	make menuconfig

<pre>
        ==> Base system ---> [-] firewall
        ==> Base system ---> busybox ---> Linux System Utilities ---> [*] mkswap
                                                                 ---> [*] swaponoff
        ==> LuCi ---> Modules ---> [*] luci-mod-freifunk (remove firewall dependency in luci Makefile before)
                 ---> Applications ---> [*] luci-app-olsr
                                   ---> [*] luci-app-olsr-services
                 ---> Themes ---> [*] luci-theme-bootstrap
                 ---> Translations ---> [*] luci-i18n-german
        ==> Network ---> VPN ---> [*] vtun (copy vtun-Makefile before to compile without ssl and lzo)
                    ---> Firewall ---> [*] iptables-mod-ipopt
                    ---> Routing and Redirection ---> [*] ip
                                                 ---> [*] olsrd
                                                                ---> [*] olsrd-mod-arprefresh/olsrd-mod-dyn-gw/olsrd-mod-nameservice/olsrd-mod-txtinfo/olsrd-mod-watchdog
                    ---> [*] uhttpd
                    ---> uhttpd ---> [*] uhttpd-mod-tls
                    ---> [-] wpad-mini
        ==> Languages ---> lua ---> [*] libiwinfo-lua

	weimarnetz/openwrt-build/mybuild.sh applymystuff "ffweimar" "adhoc" "42"
	weimarnetz/openwrt-build/mybuild.sh make 		# needs some hours
	
	# flash your image via TFTP
	FW="/path/to/your/baked/firmware_file"
	IP="your.own.router.ip"
	while :; do atftp --trace --option "timeout 1" --option "mode octet" --put --local-file $FW $IP && break; sleep 1; done
</pre>

how to do a sysupgrade via wifi
---------------------------------

* usage
    * login via ssh
    * prepare the router by calling _firmware_wget_prepare_for_lowmem_devices
    * fetch/copy firmware image to /tmp/fw
    * call _firmware_burn 

how to development directly on a router
------------------------------------------

	opkg update
	opkg install git

	echo  >/tmp/gitssh.sh '#!/bin/sh'
	echo >>/tmp/gitssh.sh 'logger -s "$0: $*"'
	echo >>/tmp/gitssh.sh 'ssh -i /etc/dropbear/dropbear_dss_host_key $*'

	chmod +x /tmp/gitssh.sh
	export GIT_SSH="/tmp/gitssh.sh"		# dropbear needs this for public key authentication

	git config --global user.name >/dev/null || {
		git config --global user.name "Firstname Lastname"
		git config --global user.email "your_email@youremail.com"
		git config --edit --global
	}

	mkdir -p /tmp/dev; cd /tmp/dev
	git clone <this_repo>
	weimarnetz/openwrt-build/mybuild.sh build_ffweimar_update_tarball
	cd /; tar xvzf /tmp/tarball.tgz; rm /tmp/tarball.tgz

	cd /tmp/dev/weimarnetz
	git add <changed_files>
	git commit -m "decribe changes"
	git push ...

piggyback kalua on a new router model without building from scratch
-------------------------------------------------------------------

	# for new devices, which are flashed with a plain openwrt
	# from http://downloads.openwrt.org/snapshots/trunk/ do this:

	# plugin ethernet on WAN, to get IP via DHCP, wait
	# some seconds, connect via LAN with 'telnet 192.168.1.1' and

	# look with which IP was given on WAN, then do:
	ifconfig $(uci get network.wan.ifname) | fgrep "inet addr:"
	/etc/init.d/firewall stop
	/etc/init.d/firewall disable
	exit

	# plugin ethernet on WAN and connect to the router
	# via 'telnet <routers_wan_ip>', then do:
	opkg update
	opkg install ip bmon netperf
	opkg install olsrd olsrd-mod-arprefresh olsrd-mod-watchdog olsrd-mod-txtinfo olsrd-mod-nameservice
	opkg install uhttpd uhttpd-mod-tls px5g
	opkg install kmod-ipt-compat-xtables iptables-mod-conntrack iptables-mod-conntrack-extra iptables-mod-extra
	opkg install iptables-mod-filter iptables-mod-ipp2p iptables-mod-ipopt iptables-mod-nat iptables-mod-nat-extra
	opkg install iptables-mod-ulog ulogd ulogd-mod-extra

	# build full kalua-tarball on server
	weimarnetz/openwrt-build/mybuild.sh build_ffweimar_update_tarball full

	# copy from server to your router
	scp user@yourserver:/tmp/tarball.tgz /tmp/tarball.tgz
	# OR take this prebuilt one:
	wget -O /tmp/tarball.tgz http://46.252.25.48/tarball_full.tgz
	# decompress:
	cd /; tar xvzf /tmp/tarball.tgz; rm /tmp/tarball.tgz

	# execute config-writer
	/etc/init.d/apply_profile.code