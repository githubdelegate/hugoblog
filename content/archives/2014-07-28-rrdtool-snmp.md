+++
categories = ["dev"]
date = "2014-07-28T15:32:27+08:00"
draft = false
slug = "rrdtool-snmp"
title = "利用 rrdtool 和 snmp 来绘制流量图"

+++

#### 事出必有因
---

之前一直是在基于Cacti做一些流量统计的事情，虽然也做了些修改，但是感觉还是比较受限制。

![old sys pic](/images/2014/1400812152.png)

上面这张图 虽然你从表面上看不出来它是基于Cacti的，但是后端的数据收集是Cacti来做的，我只是重写了展示，当然他也是利用rrdtool来画图，走snmp协议。哦，也许你已经看到了 CactiEZ的水印，好吧，我们继续。

我为什么要自己做这样一件事，而不是继续用cacti。cacti非常强大，我必须承认这一点。不过我是这样想的：

1.  自己做可以了解数据采集和绘图的整个过程，可以学到底层更多的东西。
2.  虽然一开始功能上很难做到cacti这么强大，由于是自己写，会非常灵活，可定制性强，可以满足很多需求。
3.  天生不安分的性格。


#### 开工
---

首先，我在一个文件中写上我所有要采集数据的IP，然后定时去通过snmp协议去采集数据，存储到rrd文件中，最后通过rrdtool来绘出所需要的图。

``` log
#Web Server
10.0.28.22
10.0.28.23
10.0.28.24
10.0.28.25
10.0.28.26
10.0.28.27
10.0.28.28
10.0.28.29
10.0.28.30
10.0.28.31
10.0.28.32
10.0.28.33
10.0.28.34
10.0.28.35
#Database
10.0.28.36
10.0.28.37
10.0.28.38
```

因为我这边的机器只有内网激活了，并且都在eth0上，所以这里我以eth0为例。

``` shell
# 首先取得 eth0 接口的 ifIndex
index=$(snmpwalk -IR localhost RFC1213-MIB::ifDescr |grep eth0|cut -d '=' -f 1|cut -d '.' -f 2)

# 再通过 snmp 协议取得 ififInOctets 和 ifOutOctets 的值
# 由于在 /etc/snmp.conf 中配置了 defVersion 和 defCommunity ，所以 snmpget 命令不用指定这两个参数
eth0_in=$(snmpget -IR -Os localhost ifInOctets.${index}|cut -d ':' -f 2|tr -d '[:blank:]')
eth0_out=$(snmpget -IR -Os localhost ifOutOctets.${index}|cut -d ':' -f 2 |tr -d '[:blank:]')
echo ${eth0_in}
echo ${eth0_out}

# 需要我要用这些数据来更新 eth0.rrd ，注意 update 时的 timestamp 我们用的是 N
rrdtool updatev /home/libk/eth0.rrd N:${eth0_in}:${eth0_out}
```

下面是一个完整的脚本：

``` shell
#!/bin/sh

BASE_PATH='/data/web/htdocs/net'
RRD_PATH='/data/web/htdocs/net/rrd'
FLOW_LOG="/data/web/htdocs/net/script/flow-ip.log"
community="public"

function checkip {
        dot=`echo $1 | awk -F '.' '{print NF-1}'`
        if [ $dot -ne 3 ]; then
                return 1
        fi

        count=0
        for var in `echo $1 | awk -F. '{print $1, $2, $3, $4}'`
        do
                echo $var | grep "^[0-9]*$" >/dev/null
                if [ $? -ne 0 ]; then
                        return 1
                fi

                if [ $var -ge 0 -a $var -le 255 ] ; then
                        ((count=count+1))
                        continue
                else
                        return 1
                fi
        done

        if [ $count -eq 4 ]; then
                return 0
        else
                return 1
        fi
}

function create_rrd_flow_graph() {
}

update_rrd_flow_graph() {
    if [ ! -f  ${RRD_PATH}/${line}/eth0.rrd ]; then
        mkdir -p ${RRD_PATH}/${line}/
        create_rrd_flow_graph
        echo "create rrd file"
    fi

    n=`date "+"%s""`
    index=$(snmpwalk -v2c -c ${community} ${line}  RFC1213-MIB::ifDescr |grep eth0|cut -d '=' -f 1|cut -d '.' -f 2)
    eth0_in=$(snmpget -v2c -c ${community} ${line} ifInOctets.${index} | cut -d ':' -f 4 | tr -d '[:blank:]')
    eth0_out=$(snmpget -v2c -c ${community} ${line} ifOutOctets.${index} | cut -d ':' -f 4 | tr -d '[:blank:]')
    /usr/local/rrdtool/bin/rrdtool updatev ${RRD_PATH}/${line}/eth0.rrd $n:${eth0_in}:${eth0_out} >> /tmp/rrd.log

    index1=$(snmpwalk -v2c -c ${community} ${line}  RFC1213-MIB::ifDescr |grep eth1|cut -d '=' -f 1|cut -d '.' -f 2)
    eth1_in=$(snmpget -v2c -c ${community} ${line} ifInOctets.${index1} | cut -d ':' -f 4 | tr -d '[:blank:]')
    eth1_out=$(snmpget -v2c -c ${community} ${line} ifOutOctets.${index1} | cut -d ':' -f 4 | tr -d '[:blank:]')
    /usr/local/rrdtool/bin/rrdtool updatev ${RRD_PATH}/${line}/eth1.rrd $n:${eth1_in}:${eth1_out} >> /tmp/rrd.log

}

function create_graph() {
rm -f ../graph/${line}-eth0.png
rm -f ../graph/${line}-eth1.png
/usr/local/rrdtool/bin/rrdtool graph ${BASE_PATH}/graph/${line}-eth0.png \
--imgformat=PNG \
--start=-1200 \
--base=1000 \
--title=" Title - ${line} - eth0" \
--height=135 \
--width=500 \
--alt-autoscale-max \
--lower-limit=0 \
--vertical-label='位/秒' \
--slope-mode \
--font TITLE:10:"${BASE_PATH}/font/msyhbd.ttf" \
--font AXIS:8:"${BASE_PATH}/font/msyhbd.ttf" \
--font LEGEND:8:"${BASE_PATH}/font/msyhbd.ttf" \
--font UNIT:8:"${BASE_PATH}/font/msyhbd.ttf" \
--watermark="" \
DEF:a="${RRD_PATH}/${line}/eth0.rrd":traffic_in:AVERAGE \
DEF:b="${RRD_PATH}/${line}/eth0.rrd":traffic_out:AVERAGE \
CDEF:cdefa=a,8,* \
CDEF:cdefe=b,8,* \
CDEF:cdefi=a,UN,INF,UNKN,IF \
AREA:cdefa#00BF47FF:"流入"  \
GPRINT:cdefa:LAST:"当前\:%8.2lf %s"  \
GPRINT:cdefa:AVERAGE:"平均\:%8.2lf %s"  \
GPRINT:cdefa:MAX:"最大\:%8.2lf %s\n"  \
LINE1:cdefe#0D006AFF:"流出"  \
GPRINT:cdefe:LAST:"当前\:%8.2lf %s"  \
GPRINT:cdefe:AVERAGE:"平均\:%8.2lf %s"  \
GPRINT:cdefe:MAX:"最大\:%8.2lf %s\n"  \
AREA:cdefi#8F9286FF:"" >> /tmp/rrd.log

/usr/local/rrdtool/bin/rrdtool graph ${BASE_PATH}/graph/${line}-eth1.png \
--imgformat=PNG \
--start=-1200 \
--base=1000 \
--title="title - ${line} - eth1" \
--height=135 \
--width=500 \
--alt-autoscale-max \
--lower-limit=0 \
--vertical-label='位/秒' \
--slope-mode \
--font TITLE:10:"${BASE_PATH}/font/msyhbd.ttf" \
--font AXIS:8:"${BASE_PATH}/font/msyhbd.ttf" \
--font LEGEND:8:"${BASE_PATH}/font/msyhbd.ttf" \
--font UNIT:8:"${BASE_PATH}/font/msyhbd.ttf" \
--watermark="" \
DEF:a="${RRD_PATH}/${line}/eth1.rrd":traffic_in:AVERAGE \
DEF:b="${RRD_PATH}/${line}/eth1.rrd":traffic_out:AVERAGE \
CDEF:cdefa=a,8,* \
CDEF:cdefe=b,8,* \
CDEF:cdefi=a,UN,INF,UNKN,IF \
AREA:cdefa#00BF47FF:"流入"  \
GPRINT:cdefa:LAST:"当前\:%8.2lf %s"  \
GPRINT:cdefa:AVERAGE:"平均\:%8.2lf %s"  \
GPRINT:cdefa:MAX:"最大\:%8.2lf %s\n"  \
LINE1:cdefe#0D006AFF:"流出"  \
GPRINT:cdefe:LAST:"当前\:%8.2lf %s"  \
GPRINT:cdefe:AVERAGE:"平均\:%8.2lf %s"  \
GPRINT:cdefe:MAX:"最大\:%8.2lf %s\n"  \
AREA:cdefi#8F9286FF:"" >> /tmp/rrd.log
}

while read -r line
do
    echo $line

    checkip ${line}
        case $? in
                1)
                        continue
                        ;;
        esac

    update_rrd_flow_graph
    #create_graph
done < $FLOW_LOG
```

数据采集完了 当然就是画图了。我直接写了个 graph.php  专门来输出图像：

``` php
<?php

header("Content-type: image/png");
ob_end_clean();
          ....
```
只需这样便可以调用：

``` html
<img width="500" height="120" src="./graph.php?s=86400&amp;ip=10.0.28.36">
```


而且参数可以自己定义，想怎么改就怎么改。配色和界面也做了些调整，最后上下效果图：

![](/images/2014/1400812474.png)

![](/images/2014/1400812491.png)






