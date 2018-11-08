title: SRS服务器 二 保存及拉取文件数据
author: Cyrus
tags:
  - SRS
categories:
  - rtmp
date: 2018-10-31 22:35:00
---
在SRS服务器一中讲到了SRS服务器的搭建，现在讲一下SRS服务器保存推流文件的设置。

### 一、SRS服务器保存推流文件的设置

1、找到srs.conf文件配置
```
cd srs/trunk/
vi conf/srs.conf
```

2、修改如下：
```
# main config for srs.
# @see full.conf for detail config.

listen              1935;
max_connections     1000;
srs_log_tank        file;
srs_log_file        ./objs/srs.log;
http_api {
    enabled         on;
    listen          1985;
}
http_server {
    enabled         on;
    listen          8080;
    dir             ./objs/nginx/html;
}
stats {
    network         0;
    disk            sda sdb xvda xvdb;
}
vhost __defaultVhost__ {
    dvr {		//dvr保存数据的配置
        enabled    on;
#        dvr_path    ./objs/nginx/html/[app]/[stream].[timestamp].flv;
        dvr_path    ./objs/nginx/html/[app]-[stream].flv
        dvr_plan    session;
        dvr_duration    30;
        dvr_wait_keyframe    on;
        time_jitter    full;
    }
}
```
其中dvr_path就是文件保存地址,保存文件必须为flv格式。在推流过程中，文件会保存为xxx.flv.tmp，停止推流后自动修改为xxx.flv。
obs推流：
![](obs_set.png)
vlc播放：
![](vlc_play.png)
推流时：
![](pushing.png)
推流后：
![](pushed.png)

直接播放保存的flv文件(点播)：
![](play_file.png)
注：只有存放在***/objs/nginx/html***目录下的***flv格式文件***才能播放。另推流时暂存的xxx.flv.tmp文件也可以用这种方式播放，即<font color=ff0000>播放的文件，只跟文件存放数据的格式有关，跟文件名无关</font>。



