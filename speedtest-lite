#!/bin/sh
# Copyright 2015-2017 Neutron Soutmun
# All Rights Reserved.
#
#    Licensed under the Apache License, Version 2.0 (the "License"); you may
#    not use this file except in compliance with the License. You may obtain
#    a copy of the License at
#
#         http://www.apache.org/licenses/LICENSE-2.0
#
#    Unless required by applicable law or agreed to in writing, software
#    distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
#    WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
#    License for the specific language governing permissions and limitations
#    under the License.
#
# Inspired by speedtest-cli written in python by Matt Martz (sivel)
#   https://github.com/sivel/speedtest-cli

PROGRAMNAME="Speedtest-Lite"
VERSION="0.2.0"
AGENT="$PROGRAMNAME/$VERSION"

SPEEDTEST_CONFIG="https://www.speedtest.net/speedtest-config.php"
SPEEDTEST_SERVERS="https://www.speedtest.net/speedtest-servers-static.php"
CURL_OPTS="--connect-timeout 10 --user-agent \"$AGENT\""
NC_OPTS=""

PROG_TEMPFILE=$(which tempfile)
PROG_DATE=$(which date)
PROG_CURL=$(which curl)
PROG_NC=$(which nc)
PROG_PV=$(which pv)
PROG_BC=$(which bc)
PROG_SPARK=$(which spark)

warn_required_tool () {
  printf "%s is required\n" "$1"
  exit 1
}

# Check required tools
$PROG_DATE +%s.%N | grep -v "%N" >/dev/null || warn_required_tool "date with nanoseconds support"
test -n "${PROG_CURL}" || warn_required_tool "curl"
test -n "${PROG_NC}" || warn_required_tool "netcat (nc)"
test -n "${PROG_PV}" || warn_required_tool "pipe viewer (pv)"
test -n "${PROG_BC}" || warn_required_tool "arbitrary precision calculator language (bc)"

# Workaround if tempfile does not exists, eg. OpenWRT
if [ "x$PROG_TEMPFILE" = "x" ]; then
tempfile () {
  file="/tmp/$(tr -dc 'a-z0-9' </dev/urandom | head -c16)"
  touch $file
  echo $file
}
fi

closest_cachedir=~/.speedtest-lite
closest_cache=$closest_cachedir/closest.cache
closest_list=$closest_cachedir/closest.list
config=`tempfile`
servers=`tempfile`
filterservers=`tempfile`
closest=`tempfile`
bestlatency=`tempfile`
latency=`tempfile`
rawspeed=`tempfile`
histogram=`tempfile`
uldata=`tempfile`
ulallspeed=`tempfile`

test -d $closest_cachedir || mkdir -p $closest_cachedir

flag_simple=0
serverid=""
country=""
client_conf=""
source=""
normal=1

init () {
  if [ "x$source" != "x" ]; then
    info "The testing will originate from $source ...\n"
  fi

  info "Retrieving speedtest.net configuration...\n"
  getConfigFile

  info "Retrieving speedtest.net server list...\n"
  fetchClosestServers
}

cleanup () {
  if [ $normal -eq 0 ]; then
    printf "\nAborting...\n"
    sleep 3
  fi

  rm -f $config
  rm -f $servers
  rm -f $filterservers
  rm -f $closest
  rm -f $bestlatency
  rm -f $latency
  rm -f $histogram
  rm -f $rawspeed
  rm -f $uldata
  rm -f $ulallspeed

  if [ $normal -eq 1 ]; then
    exit 0
  else
    exit 1
  fi
}

trap "normal=0; cleanup; exit" HUP INT TERM

info () {
  if [ $flag_simple -eq 0 ]; then
    printf "$@"
  fi
}

getConfigFile () {
  response_code=$($PROG_CURL $CURL_OPTS -sw '%{http_code}' "${SPEEDTEST_CONFIG}" -o $config)

  if [ "x$response_code" != "x200" ]; then
    echo "ERROR: Could not get client configuration!!!"
    normal=0
    cleanup
  fi

  client_conf=$(getConfig client)
}

getValue () {
  # line=$1
  # attr=$2

  echo $1 | sed -n "s/.*$2=\"\([^\"]*\)\".*/\1/p"
}

getConfig () {
  # key=$1

  grep "<$1[ ]*.*>" $config
}

radians () {
  pi="4*a(1)"
  echo "($1)*${pi}/180"
}

atan2 () {
  y=$1
  x=$2
  echo "(2 * a((sqrt($x*$x + $y*$y) - $x)/$y))"
}

distance () {
  lat1=$1
  lon1=$2
  server=$3
  radius=6371 # km

  lat2=$(getValue "${server}" lat)
  lon2=$(getValue "${server}" lon)

  dlat=$(radians "(${lat2})-(${lat1})")
  dlon=$(radians "(${lon2})-(${lon1})")
  radlat1=$(radians $lat1)
  radlat2=$(radians $lat2)

  a="(s((${dlat})/2) * s((${dlat})/2) + c(${radlat1}) * c(${radlat2}) * s((${dlon})/2) * s((${dlon})/2))"
  sqrta="(sqrt($a))"
  sqrt1a="(sqrt(1-($a)))"
  c="2*$(atan2 "$sqrta" "$sqrt1a")"
  d=$(echo "($radius) * ($c)" | bc -l | awk '{printf "%0.2f", $0}')

  echo "$d $server" >> $closest
}

fetchClosestServers () {
  if [ -f ${closest_cache}.timestamp ]; then
    cache_age=$(($($PROG_DATE +%s) - $(cat ${closest_cache}.timestamp)))
    if [ $cache_age -gt 86400 ]; then
      rm -f $closest_cache
      rm -f $closest_list
    fi
  fi

  test -f $closest_cache && return

  response_code=$($PROG_CURL $CURL_OPTS -sw '%{http_code}' "${SPEEDTEST_SERVERS}" -o ${servers})

  if [ "x$response_code" != "x200" ]; then
    echo "ERROR: Could not get servers list!!!"
    normal=0
    cleanup
  fi

  clat=$(getValue "${client_conf}" lat)
  clon=$(getValue "${client_conf}" lon)

  if [ "x$country" != "x" ]; then
    cat $servers | grep -w "cc=\"${country}\"" > $filterservers
    tmpservers=$servers
    servers=$filterservers
    filterservers=$tmpservers
  fi

  num_server=$(wc -l $servers | awk '{print $1}')

  jobs_pid=

  for i in `seq 1 $num_server`; do
    line=$(awk "NR==$i{print}" $servers)
    if echo $line | grep -v "<server[ ]*.*\/>" >/dev/null; then
      continue
    fi

    distance $clat $clon "$line" &
    jobs_pid="${jobs_pid} ${!}"
    if [ $(expr $i % 20) -eq 0 ]; then
      info "\r%s %%" "$(echo "($i/$num_server * 100)" | bc -l | awk '{printf "%0.2f", $0}')"
    fi
  done

  info "\r        "
  info "\r100 %%\n"

  wait $jobs_pid 2>/dev/null
  cat $closest | sort -n > $closest_cache
  $PROG_DATE +%s > ${closest_cache}.timestamp
}

getBaseURL () {
  # url=$1

  echo $1 | sed "s/upload.php//"
}

getLatency () {
  server=$1
  baseurl=$(getBaseURL "$(getValue "${server}" url)")
  url="${baseurl}latency.txt"
  id=$(getValue "${server}" id)

  rm -f $latency

  for j in `seq 1 3`; do
    start=$($PROG_DATE +%s.%N)
    reply=$($PROG_CURL $CURL_OPTS -s "$url?x=$start")
    end=$($PROG_DATE +%s.%N)

    if [ "x$reply" = "xtest=test" ]; then
      echo "($end - $start) * 1000" | bc -l >> $latency
    else
      echo "3600" >> $latency
    fi
  done

  avg=$(cat ${latency} | awk '{sum += $1 } END { if (NR > 0) printf "%0.2f", sum / (2*NR)}')
  echo "$avg $id $server"
}

getServerById () {
  # id=$1
  grep "id=\"$1\"" $closest_cache
}

getBestServer () {
  rm -f $bestlatency

  if [ "x$serverid" = "x" ]; then
    for i in `seq 1 10`; do
      line=$(awk "NR==$i{print}" ${closest_cache})
      getLatency "$line" >> $bestlatency
    done
  else
    server=$(getServerById "$serverid")
    if [ "x$server" != "x" ]; then
      getLatency "$server" >> $bestlatency
    else
      return
    fi
  fi

  server=
  max=$(wc -l $bestlatency | awk '{print $1}')
  n=1

  for i in `seq $n $max`; do
    server=$(cat $bestlatency | sort -n | head -n${i} | tail -n1)
    host=$(getValue "${server}" host)

    test -n "$host" && break
    server=""
  done

  echo $server
}

progress () {
  data=$1
  title=$2

  test $flag_simple -eq 1 && return

  if [ -x "${PROG_SPARK}" ]; then
    echo -n "${data} " >> ${histogram}
    h=$($PROG_SPARK 0 $(cat ${histogram}))
    info "\r${title} ${h}"
  else
    info "."
  fi
}

download_test () {
  test_server=$1
  test_port=$2
  test_filesize=1000000000
  period=10

  test -z $test_server && echo "Unsupported server, no socket service, skip download test !!!" && return

  title="Testing download speed"
  info "$title"

  echo "DOWNLOAD ${test_filesize}" | \
    $PROG_NC $NC_OPTS ${test_server} ${test_port} 2>/dev/null | \
    $PROG_PV -a -f >/dev/null 2>${rawspeed} &

  n=$period
  while [ $n -gt 0 ]; do
    data=$(grep -o "[0-9.]\+[kKM]\+" ${rawspeed} | sed 's/[kK]\+//g' | sed 's/M/*1024/g' | tail -n1)
    if [ "x${data}" != "x" ]; then
      data=$(echo "scale=0; ${data}/1" | bc -l)
    fi

    progress "$data" "$title"
    sleep 1;
    n=$((n-1))

    test $n -eq 0 && test $flag_simple -eq 0 && echo
  done

  PS_OPTS=
  if ps a 2>&1 | grep "B[u]syBox" >/dev/null; then
    PS_OPTS=""
    PID_POS="\$1"
  else
    PS_OPTS="aux"
    PID_POS="\$2"
  fi

  PID=$(ps ${PS_OPTS} | grep -w ${test_server} | grep "n[c]" | awk "{print ${PID_POS}}")
  if [ -n "$PID" ]; then
    kill $PID 2>/dev/null
  fi

  speed=$(grep -o '[0-9.]\+[kKM]\+' ${rawspeed} | tail -n1 | sed 's/[kK]\+//g' | sed 's/M/*1024/g')
  test -z $speed && speed=0
  dlspeed=$(echo "scale=2; (${speed}*8.388608)/1000" | bc -l)

  printf "Download: %0.2f Mbit/s\n" $dlspeed
}

upload_test () {
  url=$1
  data10k=$(tr -dc 'A-Z0-9' </dev/urandom | head -c10000)
  period=10

  title="Testing upload speed"
  info "$title"

  echo -n "content1=" > ${uldata}
  for i in `seq 1 25`; do
    echo -n "${data10k}" >> ${uldata}
  done

  start=$($PROG_DATE +%s)
  for i in `seq 1 25`; do
    echo -n "${data10k}" >> ${uldata}

    ulspeed=$($PROG_CURL $CURL_OPTS -X POST -d @${uldata} \
      -w "%{speed_upload}" \
      -s -o /dev/null \
      $url)
    echo "${ulspeed}*8" | bc -l >> ${ulallspeed}
    progress "$ulspeed" "$title"

    now=$($PROG_DATE +%s)
    if [ $((now - start)) -gt $period ] || [ $i -eq 25 ]; then
      test $flag_simple -eq 0 && echo
      break
    fi
  done

  speed=$(cat ${ulallspeed} | awk '{sum += $1 } END { if (NR > 0) printf "%0.2f", sum / NR / 1000 /1000 }')
  test -z $speed && speed=0

  printf "Upload: %0.2f Mbit/s\n" ${speed}
}

speedtest () {
  ip=$(getValue "${client_conf}" ip)
  isp=$(getValue "${client_conf}" isp)
  info "Testing from %s (%s)...\n" "$isp" "$ip"

  if [ "x$serverid" = "x" ]; then
    info "Selecting best server based on latency...\n"
  fi

  bestServer=$(getBestServer)
  test "x$bestServer" != "x" || return

  sLatency=$(echo $bestServer | awk '{print $1}')
  sID=$(echo $bestServer | awk '{print $2}')
  sDistance=$(echo $bestServer | awk '{print $3}')
  sSponsor=$(getValue "${bestServer}" sponsor)
  sName=$(getValue "${bestServer}" name)
  sHost=$(getValue "${bestServer}" host)
  sURL=$(getValue "${bestServer}" url)

  info "Hosted by %s (%s) [%s km]: %s ms\n" "$sSponsor" "$sName" \
    "$sDistance" "$sLatency"

  sServer=$(echo $sHost | cut -d: -f1)
  sPort=$(echo $sHost | cut -d: -f2)

  if [ $flag_simple -eq 1 ]; then
    printf "Ping: %s ms\n" "$sLatency"
  fi

  download_test "$sServer" "$sPort"
  upload_test "$sURL"
}

list_servers () {
  test -f $closest_list && cat $closest_list && return

  while read line; do
    sDistance=$(echo $line | awk '{print $1}')
    sID=$(getValue "$line" id)
    sSponsor=$(getValue "$line" sponsor)
    sName=$(getValue "$line" name)
    sCountry=$(getValue "$line" country)

    echo "$sID) $sSponsor ($sName, $sCountry) [$sDistance km]" | tee -a $closest_list
  done < $closest_cache
}

usage () {
  echo "usage: $0 [-h|--help] [--simple] [--list] [--server SERVER] [--version]"
  echo "optional arguments:"
  echo "  -h, --help\t\tshow this help message and exit"
  echo "  --simple\t\tsuppress verbose output, only show basic information"
  echo "  --list\t\tdisplay a list of speedtest.net servers sorted by"
  echo "\t\t\tdistance"
  echo "  --server SERVER\tspecify a server ID to test against"
  echo "  --filter COUNTRYCODE\tspecify a country code to filter the "
  echo "\t\t\tservers, eg. TH, JP, US"
  echo "  --insecure \t\tAllow connections to speedtest.net SSL with no "
  echo "\t\t\tcertificate check"
  echo "  --version\t\tshow the version number and exit"
}

cmd="speedtest"
while [ "x$1" != "x" ]; do
  param=$(echo $1 | awk -F= '{print $1}')
  value=$(echo $1 | awk -F= '{print $2}')

  case $param in
    -h | --help)
      usage
      exit
      ;;
    --server)
      test -n "$value" || shift; value=$1
      serverid="$value"
      ;;
    --simple)
      flag_simple=1
      ;;
    --list)
      cmd="list"
      ;;
    --insecure)
      CURL_OPTS="${CURL_OPTS} --insecure"
      ;;
    --filter)
      test -n "$value" || shift; value=$1
      country=$value
      closest_cache="${closest_cache}.${country}"
      closest_list="${closest_list}.${country}"
      ;;
    --source)
      test -n "$value" || shift; value=$1
      source=$value
      CURL_OPTS="${CURL_OPTS} --interface ${source}"
      NC_OPTS="${NC_OPTS} -s ${source}"
      ;;
    --version)
      printf "%s %s\n" "$PROGRAMNAME" "$VERSION"
      exit
      ;;
  esac
  shift
done

case $cmd in
  list)
    init
    list_servers
    ;;
  *)
    init
    speedtest
    ;;
esac

cleanup
