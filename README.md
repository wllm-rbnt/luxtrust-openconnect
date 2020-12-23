This HOWTO is about modifying GNUTLS to make OpenConnect work with
LuxTrust/Gemalto smartcards on GNU/Linux. It shows you how to patch the 
GNUTLS library, build and install the new package and how to invoke OpenConnect
to make a successful connection to a remote VPN gateway.

This is written for Debian 10 aka Buster. The fix is simple, ugly but
functionnal, and will probably break some other applications.

# LuxTrust Middleware setup

Install LuxTrust middleware and fix dependencies:

	$ sudo dpkg -i Gemalto_Middleware_Debian_64bit_7.2.0-b04.deb luxtrust-middleware-1.2.1-64.deb                                                      
	$ sudo apt install -f

# GNUTLS Patching

## Fetch the contents of this repo

	$ git clone https://github.com/wllm-rbnt/luxtrust-openconnect.git

## Prepare for compilation

Prepare the compilation environment:

	$ mkdir deb
	$ cd deb
	$ sudo apt install devscripts fakeroot build-essential

## Adjust and add source repository to /etc/apt/source.list

Add src targets to your APT source.list file:

	deb-src http://deb.debian.org/debian buster main
	deb-src http://deb.debian.org/debian-security/ buster/updates main
	deb-src http://deb.debian.org/debian buster-updates main

Install GNUTLS build dependencies and sources:

	$ sudo apt update
	$ sudo apt build-dep libgnutls30
	$ apt source libgnutls30

That last command will create a new directory named `gnutls28-3.x.y`. 
Don't be surprised, that IS the correct directory for `libgnutls30`.

## Apply patch

Apply the GNUTLS fix:

	$ cd gnutls28-3.x.y
	$ git apply path_to/luxtrust-openconnect/0001-Fix-LuxTrust-OpenConnect.patch 

## Build GNUTLS

Add a local suffix to the version (e.g. `-conostix`) and compile GNUTLS 
(documentation build and tests disabled):

	$ dch -l '-conostix'
	$ dpkg-buildpackage -rfakeroot -b
	
This will most likely end with an error message about not having been able to sign 
the packages. That's to be expected and won't keep you from installing them in the next step.

We added a suffix to the package version so that we can easily hold or uninstall them in the future.

## Install packages

Install the produced packages and any missing dependencies:

	$ sudo dpkg -i $(ls -1 ../*.deb | grep -v dbgsym | xargs)
	$ sudo apt install -f
	
## Marking packages as held

You might want to mark the recently marked packages as held to prevent a further `apt ugprade`
from upgrading to an unpatched version

	$ sudo apt-mark hold `dpkg -l | awk '/conostix/{print $2"="$3}'`

# Reverting

## Reinstall Debian's version

If required, reinstall GNUTLS original packages from Debian's repositories
(here's where the version suffix comes in handy):

	$ sudo apt install $(dpkg -l | awk '/conostix/{gsub("-conostix","",$3);print $2"="$3}')

# Connection

## OpenConnect Installation/Invocation

Install OpenConnect:

	$ sudo apt install openconnect

Locate LuxTrust/Gemalto PKCS11 URIs associated with your smartcard:

	$ p11tool --list-tokens
	[...]
	Token 5:
		URL: pkcs11:model=Classic%20V3;manufacturer=Gemalto%20S.A.;serial=0123456789ABCDEF;token=GemP15-1
	    	Label: GemP15-1
	    	Type: Hardware token
	    	Manufacturer: Gemalto S.A.
	    	Model: Classic V3
	    	Serial: 0123456789ABCDEF
	[...]

Then list the certs on that token. What you're looking for are the certificate and the
private key that have `User Cert Auth` as a label:

	$ p11tool --login --list-all "pkcs11:model=Classic%20V3;manufacturer=Gemalto%20S.A.;serial=0123456789ABCDEF;token=GemP15-1"
	Token 'GemP15-1' with URL 'pkcs11:model=Classic%20V3;manufacturer=Gemalto%20S.A.;serial=0123456789ABCDEF;token=GemP15-1' requires user PIN
	Enter PIN: 012345
	[...]
	Object 6:
		URL: pkcs11:model=Classic%20V3;manufacturer=Gemalto%20S.A.;serial=0123456789ABCDEF;token=GemP15-1;id=%00%01%02%03%04%05%06%07%08%09%0a%0b%0c%0d%0e%0f%10%11%12%13;object=User%20Cert%20Auth;object-type=cert
		Type: X.509 Certificate
    	Label: User Cert Auth
    	ID: 00:01:02:03:04:05:06:07:08:09:0a:0b:0c:0d:0e:0f:10:11:12:13
 	[...]
	Object 10:
		URL: pkcs11:model=Classic%20V3;manufacturer=Gemalto%20S.A.;serial=0123456789ABCDEF;token=GemP15-1;id=%00%01%02%03%04%05%06%07%08%09%0a%0b%0c%0d%0e%0f%10%11%12%13;object=User%20Cert%20Auth;object-type=private
		Type: Private key
    	Label: User Cert Auth
		Flags: CKA_WRAP/UNWRAP; CKA_PRIVATE; CKA_SENSITIVE;
    	ID: 00:01:02:03:04:05:06:07:08:09:0a:0b:0c:0d:0e:0f:10:11:12:13
	[...]

Launch OpenConnect with the right URIs (`-c` for the cert, `-k` for the key) and some debug options:

	$ sudo /usr/sbin/openconnect --disable-ipv6 --printcookie --dump-http-traffic -v -c 'pkcs11:model=Classic%20V3;manufacturer=Gemalto%20S.A.;serial=0123456789ABCDEF;token=GemP15-1;id=%00%01%02%03%04%05%06%07%08%09%0a%0b%0c%0d%0e%0f%10%11%12%13;object=User%20Cert%20Auth;object-type=cert' -k 'pkcs11:model=Classic%20V3;manufacturer=Gemalto%20S.A.;serial=0123456789ABCDEF;token=GemP15-1;id=%00%01%02%03%04%05%06%07%08%09%0a%0b%0c%0d%0e%0f%10%11%12%13;object=User%20Cert%20Auth;object-type=private' https://subdomain.domain.tld

## After successful connection

Tweak your routing table to suit your needs, and restore your original
(pre-connection) resolv.conf if required:

	$ sudo ip r del default
	$ sudo ip r add <remote_subnet> dev tunX
	$ sudo ip r add default via <original_local_network_gateway>
	$ sudo cp /var/run/vpnc/resolv.conf-backup /etc/resolv.conf

where `remote_subnet` is the network you're trying to reach on the VPN and
`original_local_network_gateway` is your local gateway.

