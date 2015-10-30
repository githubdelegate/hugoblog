+++
categories = ["android"]
date = "2014-06-19T14:20:20+08:00"
draft = false
slug = "ifeng-xmss"
title = "报警短信助手 (Android)"

+++

少年，你是不是经常被大量的报警所淹没。
少年，你是不是经常在茫茫短信中漏掉十分重要的报警。
少年，你是不是Android手机。

运维人员每天会处理大量短信报警，经常会有漏报警，报警处理不及时，不重要短信频频打扰我们生活的问题。报警助手对报警进行拦截，实行分级别管理，对不同级别报警，采用不同的处理和提醒方式，让你能够高效准确的处理报警，节省更多的时间投入到工作、学习、及世界杯中。

本应用基于小米运维部开源的应用进行了二次开发，并得到了开发人员的授权。我只是站在巨人的肩膀上做了点小小的优化：
1. UI调整，放弃小米的shi黄色，改用企业蓝。
2. 短信格式解析调整。
3. 改进应用退出方式，使得退出更加优雅。
4. 支持本地的报警级别分析。
5. 编写自有的升级接口。

举个例子，我们最常见的按两下返回键退出程序的实现代码如下：

```java
	@Override
	public boolean onKeyDown(int keyCode, KeyEvent event) {  
        if (keyCode == KeyEvent.KEYCODE_BACK) {  
            if (isQuit == false) {  
                isQuit = true;  
                Toast.makeText(getBaseContext(), "再按一次退出，建议放在后台运行", Toast.LENGTH_SHORT).show();  
                TimerTask task = null;  
                task = new TimerTask() {  
                    @Override  
                    public void run() {  
                        isQuit = false;  
                    }  
                };  
                timer.schedule(task, 2000);  
            } else {  
                finish();  
                System.exit(0);  
            }  
        }  
        return true;  
	} 
```

在小米的开源代码中没有找到他们的服务器端升级接口的代码实现，阅读客户端的代码，揣测里面至少有两个字段：

```php
<?php

//state:s是否升级，0是没有新版本
//url:升级新版apk的下载地址

$update_url = "http://yourserver/alertsystem/apk/alertsystem_v2.0.3.apk";
$update_ver = 3;

if( isset($_GET["ver"])  &&  !empty($_GET["ver"])){
        if($update_ver > $_GET["ver"]){
                $res["url"] = $update_url;
                $res["stat"] = 1;
        }else{
                $res["url"] = "";
                $res["stat"] = 0;
        }
}else{
                $res["url"] = "";
                $res["stat"] = 0;
}
echo json_encode($res);
```

下面上几张效果图：

![](/images/2014/1403158738.png)

![](/images/2014/1403158707.png)

![](/images/2014/1403158761.jpg)

开源地址：https://github.com/xiaomi-sa/alertsystem