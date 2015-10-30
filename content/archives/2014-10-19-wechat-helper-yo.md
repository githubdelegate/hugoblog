+++
categories = ["tool"]
date = "2014-10-19T23:26:04+08:00"
draft = false
slug = "wechat-helper-yo"
title = "微信运维助手 - 小优"

+++

#### 1.起初

---

很久没自己做东西了，恰恰这段时间又忙着各系统的上线，自己学习的时间也少了，总之是自己越来越懒了。前端时间想写个运维方向的iOS应用，但是提交应用每年要向Apple交银子，实在不舍，也没有好的点子，就搁置了。恰好有个同学在搞互联网微信营销，于是想着能不能结合微信做点东西，于是有了这篇文章，算是个入门，高手略过。


#### 2.CMDB信息查询服务接入微信

---

申请账号什么的我就不赘述了，重点在于通过微信公众账号接通自己的服务，为用户提供服务。我打算实现的功能是：用户提交一个IP，然后返回这个IP的服务器信息，包括机身码、机房位置、管理卡IP、负责人以及服务器内跑的服务等。方便运维人员不在公司的时候能够快速查询信息。

9月底和桐桐沟通开通查询CMDB的API接口，桐桐很快就搞定了。于是开始着手业务逻辑编码了。先把数据请求流程图贴出来吧：

![](/images/2014/1413732304.jpg)

我们要做的就是接收用户发出的指令，然后查询信息，返回给用户，处理用户信息的核心代码如下：

``` php

 public function responseMsg()
    {
        //get post data, May be due to the different environments
        $postStr = $GLOBALS["HTTP_RAW_POST_DATA"];

		//extract post data
        if (!empty($postStr)){
	        /* libxml_disable_entity_loader is to prevent XML eXternal Entity Injection,
	           the best way is to check the validity of xml by yourself */
	        libxml_disable_entity_loader(true);
	        $postObj = simplexml_load_string($postStr, 'SimpleXMLElement', LIBXML_NOCDATA);
	        $fromUsername = $postObj->FromUserName;
	        $toUsername = $postObj->ToUserName;
	        $keyword = trim($postObj->Content);
	        $time = time();
	        $textTpl = "<xml>
	                    <ToUserName><![CDATA[%s]]></ToUserName>
	                            <FromUserName><![CDATA[%s]]></FromUserName>
	                            <CreateTime>%s</CreateTime>
	                            <MsgType><![CDATA[%s]]></MsgType>
	                            <Content><![CDATA[%s]]></Content>
	                            <FuncFlag>0</FuncFlag>
	                            </xml>";
	        if(!empty( $keyword ))
	        {
	                $msgType = "text";
	                $ProcessMessage = new ProcessMessage();
	                $contentStr = $ProcessMessage->Process($keyword);
	                $resultStr = sprintf($textTpl, $fromUsername, $toUsername, $time, $msgType, $contentStr);
	                echo $resultStr;
	        }else{
	                echo "Input something...";
	        }

        }else {
                echo "";
                exit;
        }
    }

```

为了保持代码逻辑性、模块性和扩展性，我另外写了一个类ProcessMessage用于处理用户提交的信息，方便后期的功能添加。代码如下：

``` PHP

class ProcessMessage
{
        public function Process($mes)
    	{
                //IP地址处理
                if(preg_match("/^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$/", $mes)) {
                        return $this->Return_IP_Info($mes);
                }else{
                        return "Hello World";
                }
        }

        private function Return_IP_Info($ip){
                $pageContents = json_decode($res = HttpClient::quickPost('http://yuwei.baidu.com/api/server', array('ip' => $ip)));
                $report = json_decode($pageContents->report_info);
                $result = "主机名:".$report->hostName;
                if($res == "None"){
                        $result = "你确定公司有这台机器么，那可能没加进CMDB";
                }
                return $result;
        }
}

```

具体的实现代码如上了，如果匹配到用户提交的信息是IP地址，那么就调用 Return_IP_Info 方法，考虑到信息安全性，我们将这个方法设置为private。在这个类中我们可以扩展更多的功能，比如报警，远程执行命令等。大家可以任意发挥了。


#### 2.Screen Shots

![](/images/2014/1413732329.PNG)
![](/images/2014/1413732358.PNG)
