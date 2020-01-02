# WarVOX

*Notice*: WarVOX is currently unsupported and unmaintained. YMMV.
WarVOX is released under a BSD-style license. See docs/LICENSE for more details.

 - [Installing](#installing)

## Installing
Note: This install process was edited and tested to be working (Ubuntu 16.04) on 12/31/2019.

WarVOX requires a Linux operating system, preferably Ubuntu or Debian.
WarVOX requires PostgreSQL 9.1 or newer with the "contrib" package installed for integer array support.

To get started, install the OS-level dependencies:
```
	$ sudo apt-get install gnuplot lame build-essential libssl-dev libcurl4-openssl-dev \
	  postgresql postgresql-contrib postgresql-common git-core curl libpq-dev sox
```
Install the GPG Key so RVM will sucessfully install
```
	$ gpg --keyserver hkp://pool.sks-keyservers.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
```

Install RVM to obtain Ruby 2.2.5 or later
```
	$ curl -L https://get.rvm.io | bash -s stable --autolibs=3 --rails
```

After RVM is installed you need to run the rvm script provided
```
	$ source /usr/local/rvm/scripts/rvm
```

Install Bundler, version 1.15.3
```
	$ gem install bundler -v '=1.15.3'
```

Clone this repository to the location you want to install WarVOX:
```
	$ git clone git://github.com/the-toph/warvox.git /opt/warvox
```

Configure WarVOX:
```
	$ cd /opt/warvox
	$ bundle install
	$ make
```

At the end of the Make, you should see "WarVOX Installation Verifier" If not, verify your installation:
```
	$ bin/verify_install.rb
```

If no errors, install Postgres:

```
apt-get install postgresql postgresql-contrib
```

Configure the PostgreSQL account for WarVOX:

```
	$ sudo su - postgres
	$ createuser -s warvox
	$ createdb warvox -O warvox
	$ psql
	psql> alter user warvox with password 'randompass';
	psql> exit
	$ exit
```

Copy the example database configuration to database.yml:

```
	$ cp config/database.yml.example config/database.yml
```

Copy the example secrets configuration to secrets.yml:
```
	$ cp config/secrets.yml.example config/secrets.yml
```
Create a new secrect token:
```
	$ rake secret > config/session.key
```
Modify config/database.yml to include the password set previously

Initialize the WarVOX database:
```
	$ make database
```

Add an admin account to WarVOX
```
	$ bin/adduser admin randompass
```
To get assets (images, etc) to show up, you need to first compile assets in production environment:
```
	$ RAILS_ENV=production bundle exec rake assets:precompile
```
This will compile all static assets into `public` folder.

Next, you need to enable the `RAILS_SERVE_STATIC_FILES` environment variable through the terminal:
```
	$ export RAILS_SERVE_STATIC_FILES=true
```
Note: If you do not add this to your .env file, you will need to export this variable before each start of Warvox.

Start the WarVOX daemons:
```
	$ bin/warvox
```

or to bind WarVox to all interfaces:
```
	$ bin/warvox --address 0.0.0.0
```

Access the web interface at http://127.0.0.1:7777/
At this point you can configure a new IAX2 provider, create a project, and start making calls.

## Configuring Asterisk

You may use Asterisk as a gateway between Warvox and your SIP Provider (Warvox -> Asterisk via IAX2 -> Your SIP Provider via SIP)
Basic configuration of Asterisk goes beyond the scope of this document...

Example iax.conf configuration:

```
[warvox2]
context=wardialer ;Use the context you have configured for WarVOX, or see the example dialplan I use on my install.
requirecalltoken=no
type=friend
permit=1.2.3.4 ;Change me to the IP Address or Hostname of your WarVOX installation.
host=dynamic
secret=s3cur3p@$$w0rd ;Change this to a secure password.
port=4569
disallow=all
allow=ulaw
```

Example extensions.conf (dialplan) configrations:

Do you have more than one SIP provider, and you'd like to faux-load-balance calls over those providers?
If so, this is what I personally use:
```
[wardialer]
exten => _XXXXXXXXXX,1,Goto(wardialer,1${EXTEN},1)
;
exten => _1XXXXXXXXXX,1,NoOp(Wardialing ${EXTEN})
 same => n,Goto(${RAND(10,13)})
;
 same => 10,Goto(80)
 same => 11,Goto(90)
 same => 12,Goto(100)
 same => 13,Goto(110)
;
 same => 80,Dial(SIP/${EXTEN}@sip-provider-1)	;Change me to a Trunk/Peer name defined in sip.conf
 same => n,Goto(fail)
;
 same => 90,Dial(SIP/${EXTEN}@sip-provider-2)	;Change me to a Trunk/Peer name defined in sip.conf
 same => n,Goto(fail)
;
 same => 100,Dial(SIP/${EXTEN}@sip-provider-3)	;Change me to a Trunk/Peer name defined in sip.conf
 same => n,Goto(fail)
;
 same => 110,Dial(SIP/${EXTEN}@sip-provider-4)	;Change me to a Trunk/Peer name defined in sip.conf
 same => n,Goto(fail)
;
 same => n(fail),Congestion(5)
 same => n,Hangup
```

Do you have one single SIP provider to use? Use this instead:
```
[wardialer]
exten => _XXXXXXXXXX,1,Goto(wardialer,1${EXTEN},1)
;
exten => _1XXXXXXXXXX,1,NoOp(Wardialing ${EXTEN})
 same => n,Dial(SIP/${EXTEN}@my-sip-provider)	;Change me to a Trunk/Peer name defined in sip.conf
 same => n,Congestion(5)
```
