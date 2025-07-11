---
title: Linux_墙的配置
typora-root-url: ../..
categories:
  - [Linux]
tags:
  - null 
date: 2022/9/2
update:
comments:
published:
---

# 墙的配置

```sh
# filename: setProxy.sh
MY_PROXY_IP_="192.168.2.249"
MY_PROXY_PORT_="10810"
MY_PROXY_SECRET_=""
MY_PROXY_LOCATION_TEST_URL_="myip.ipip.net/"
function f_proxy_setup() {
    if [ "$#" -eq "2" ];
    then
        MY_PROXY_IP_="$1"
        MY_PROXY_PORT_="$2"
    fi  
    MY_PROXY_LOCATION_=socks5://${MY_PROXY_IP_}:${MY_PROXY_PORT_}
    export all_proxy="$MY_PROXY_LOCATION_"
    echo $all_proxy
}

function f_proxy_down() {
    echo "($all_proxy)" down.
    export all_proxy=""
}

function f_proxy_location() {
    echo ALL_PROXY: $all_proxy
    curl $MY_PROXY_LOCATION_TEST_URL_
}
```

使用：

```
$ . setProxy.h "xxx.xxx.xxx.xxx" "xxxx"
$ f_proxy_setup
$ f_proxy_location
$ f_proxy_down
$ f_proxy_location
```

