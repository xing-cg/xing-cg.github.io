---
title: git_config
categories:
  - - Linux
  - - git
tags:
  - 
date: 2022/9/3
updated:
comments:
published:
---

# 解决git clone慢的问题

https://zhuanlan.zhihu.com/p/148110705

```
git config --global http.proxy 127.0.0.1:11112
git config --global https.proxy 127.0.0.1:11112
git config --global --unset http.proxy
git config --global --unset https.proxy
```

# 局域网配置代理脚本

```sh
MY_PROXY_IP_="172.20.83.70" #有墙的主机ip
MY_PROXY_PORT_="11113"      #墙的端口号
MY_PROXY_SECRET_=""
MY_PROXY_LOCATION_TEST_URL_="https://myip.ipip.net/"
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

