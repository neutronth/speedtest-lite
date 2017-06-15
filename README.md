# speedtest-lite
shell script command line Internet bandwidth testing using speedtest.net.
It is written in shell script and external programs for the test and calculation.

Inspired by **speedtest-cli** written by Matt Martz (sivel) - https://github.com/sivel/speedtest-cli

INSTALL
=======

```
$ sudo apt-get install curl netcat-openbsd bc pv
$ wget https://raw.githubusercontent.com/neutronth/speedtest-lite/master/speedtest-lite
```

USAGE
=====

* Get help
```
$ ./speedtest-lite --help

usage: ./speedtest-lite [-h|--help] [--simple] [--list] [--server SERVER] [--version]
optional arguments:
  -h, --help            show this help message and exit
  --simple              suppress verbose output, only show basic information
  --list                display a list of speedtest.net servers sorted by
                        distance
  --server SERVER       specify a server ID to test against
  --filter COUNTRYCODE  specify a country code to filter the 
                        servers, eg. TH, JP, US
  --insecure            Allow connections to speedtest.net SSL with no 
                        certificate check
  --source ADDRESS      bind the testing connections to ADDRESS
                        the testing will originate from this ADDRESS
  --version             show the version number and exit
```

* Just test
```
$ ./speedtest-lite

Retrieving speedtest.net configuration...
Retrieving speedtest.net server list...
Testing from True Internet (114.109.97.xxx)...
Selecting best server based on latency...
Hosted by STS Group (Bangkok) [406.18 km]: 20.21 ms
Testing download speed...........
Download: 42.26 Mbit/s
Testing upload speed..........................
Upload: 3.28 Mbit/s
```

* Less verbose test
```
$ ./speedtest-lite --simple

Ping: 20.40 ms
Download: 43.20 Mbit/s
Upload: 3.22 Mbit/s
```

* Filter servers by country
```
$ ./speedtest-lite --filter JP

Retrieving speedtest.net configuration...
Retrieving speedtest.net server list...
Testing from True Internet (114.109.97.xxx)...
Selecting best server based on latency...
Hosted by Too late (Ishikari) [4675.45 km]: 368.75 ms
Testing download speed..........
Download: 4.52 Mbit/s
Testing upload speed.................
Upload: 1.50 Mbit/s
```

* List servers
```
$ ./speedtest-lite --list

Retrieving speedtest.net configuration...
Retrieving speedtest.net server list...
1936) Lao Telecom (Vientiane, Lao PDR ) [208.44 km]
1803) Beeline Lao (Vientiane, Lao PDR ) [208.44 km]
6220) omputer Engineer Dept, RMUTT, Thailand (Pathum Thani, Thailand ) [384.36 km]
1219) STS Group (Bangkok, Thailand ) [406.18 km]
3147) AIS (Bangkok, Thailand ) [406.18 km]
5920) PEA (Bangkok, Thailand ) [406.18 km]
6283) Ncic (Bangkok, Thailand ) [406.18 km]
3855) dtac (Bangkok, Thailand ) [406.18 km]
...
...
```

* Test against specify server
```
$ ./speedtest-lite --server 1803

Retrieving speedtest.net configuration...
Retrieving speedtest.net server list...
Testing from True Internet (114.109.97.xxx)...
Hosted by Beeline Lao (Vientiane) [208.44 km]: 59.80 ms
Testing download speed..........
Download: 40.88 Mbit/s
Testing upload speed.......................
Upload: 3.24 Mbit/s
```

* Test with source IP address binding

  The multiple uplinks environment, each link could be tested by specify
  the link IP address as source IP address.

```
$ ./speedtest-lite --source 192.168.1.191

The testing will originate from 192.168.1.191 ...
Retrieving speedtest.net configuration...
Retrieving speedtest.net server list...
Testing from 3BB (183.88.56.xxx)...
Selecting best server based on latency...
Hosted by AIS (Bangkok) [1.21 km]: 12.26 ms
Testing download speed..........
Download: 63.82 Mbit/s
Testing upload speed.......................
Upload: 14.61 Mbit/s
```

* Test and show simple speed histogram

  See: https://github.com/holman/spark

```
$ wget https://raw.githubusercontent.com/holman/spark/master/spark
```
```
$ ./speedtest-lite

Retrieving speedtest.net configuration...
Retrieving speedtest.net server list...
Testing from True Internet (114.109.97.xxx)...
Selecting best server based on latency...
Hosted by STS Group (Bangkok) [406.18 km]: 19.34 ms
Testing download speed ▁▇████████
Download: 42.17 Mbit/s
Testing upload speed ▁▁▁▁▁▁▁▁▁▁█▆▆▆▆▆▆▆▆▆▆▆▆▆
Upload: 3.33 Mbit/s
```

Contributing
============

Contributions are welcome.

Happy Hacking!
