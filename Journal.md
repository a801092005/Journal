# 2022 1/10~1/14
444 r--r--r--

600 rw-------

644 rw-r--r--

666 rw-rw-rw-

700 rwx------

744 rwxr--r--

755 rwxr-xr-x

777 rwxrwxrwx

find / -type f -iname "*123*"

tail -f var/log/message |grep *** |less

安裝
prometheu  檢查用路徑/usr/local/bin/promtool check config /etc/prometheus/prometheus.yml

新增可熱重啟curl -X POST 192.168.126.130:9090/-/reload
/etc/systemd/system/prometheus.service
  --web.enable-admin-api \
  --web.enable-lifecycle

  windows.exporter 開powershell 自訂義安裝參照自己github
  node.exporter 
  telegraf 收資料比node.exporter多

alertmanager 
  alertmanager.yml 不用寫 template
  cluster status 不會只有一台,要做兩台集群backup,保證能發信
  config.yml resolve_timeout 收值時間
  route -> routes 父類別 子類別
  發送方式用webhook最原始服務串接teams,因為沒有teams api -> 裝prometheus-msteams與default-message-card.tmpl模組
  warning 不能跟critical 重複發通知
  receviers webhook_configs url名稱 要與 msteams 上 config命名一樣之後帶上teams上的webhook

兩套service
karma GUI 

grafana dashbord GUI


# 2022 1/18~1/21
了解 OA 環境 service
建立HP ILO4 安裝nagios (超多補丁要裝)並監控 
(再另外嘗試 nagiosQL 設定)
nagios
腳本位置 /usr/local/nagios/libexec/ 

設定檔位置 /usr/local/nagios/etc/resource.cfg
確認config /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg

補丁
apt install libssl-dev
apt install zlib1g-dev
apt install expat
apt install libxml-sax-expat-incremental-perl

install perl modules:
perl -MCPAN -e 'install IO::Socket::SSL'
perl -MCPAN -e 'install Net::SSLeay'
perl -MCPAN -e 'XML::SAX::Expat'
perl -MCPAN -e 'install XML::Simple'
perl -MCPAN -e 'install Monitoring::Plugin'

設定 mail server 
apt install mailutils

# 2022 1/24~1/28
1/24~1/28
於nagios後安裝nagiosOL 要檢查php與mysql安裝環境
檢查phpinfo data.timezone 設定
安裝好nagiosQL後檢查tool -> Nagios control, Administration -> Config targets(確認全部路徑)

發送告警 Alert to ms teams 腳本 msteams.py
```
#!/usr/bin/python

import optparse
import requests

parser = optparse.OptionParser("usage: ./msteams.py -w webhook -s servicedesc -c servicestate -a hostalias -o serviceoutput -n notificationtype")
parser.add_option("-w","--webhook",dest = "webhook",help="Specify the webhook url")
parser.add_option("-s","--servicedesc",default=None,dest = "servicedesc",help="Specify the servicedesc")
parser.add_option("-c","--servicestate",default=None,dest = "servicestate",help="Specify the servicestate")
parser.add_option("-a","--hostalias",default=None,dest = "hostalias",help="Specify the hostalias")
parser.add_option("-x","--hoststate",default=None,dest = "hoststate",help="Specify the hoststate")
parser.add_option("-y","--hostoutput",default=None,dest = "hostoutput",help="Specify the hostoutput")
parser.add_option("-o","--serviceoutput",default=None,dest = "serviceoutput",help="Specify the serviceoutput")
parser.add_option("-n","--notificationtype",default=None,dest = "notificationtype",help="Specify the notificationtype")

(options, args) = parser.parse_args()

webhook = options.webhook
servicedesc = options.servicedesc
servicestate = options.servicestate
hostalias = options.hostalias
hoststate = options.hoststate
hostoutput = options.hostoutput
serviceoutput = options.serviceoutput
notificationtype = options.notificationtype


def main():
    if hoststate is None:
        print hostalias
        print servicedesc
        print servicestate
        sendServiceStateAlerts(webhook,hostalias,servicedesc,servicestate,serviceoutput,notificationtype)
    else:
        print hostalias
        print 'in else'
        sendHostStateAlerts(webhook,hostalias,hoststate,hostoutput,notificationtype)

def sendServiceStateAlerts(webhook,hostalias,servicedesc,servicestate,serviceoutput,notificationtype):
    stateColor = "#dddddd"
    exitState=3
    if (servicestate=="WARNING"):
        stateColor = "#ffff66"
        exitState=1
    elif (servicestate == "CRITICAL"):
        stateColor="#f40000"
        exitState=2
    elif(servicestate=="OK"):
        stateColor="#00b71a"
        exitState=0
    else:
        stateColor="#cc00de"
        exitState=exitState
    buildJson(webhook,servicedesc,hostalias,servicestate,exitState,stateColor)


def sendHostStateAlerts(webhook,hostalias,hoststate,hostoutput,notificationtype):
    stateColor = "#dddddd"
    exitState=3
    if (hoststate=="WARNING"):
        stateColor = "#ffff66"
        exitState=1
    elif (hoststate == "CRITICAL"):
        stateColor="#f40000"
        exitState=2
    elif(hoststate=="OK"):
        stateColor="#00b71a"
        exitState=0
    else:
        stateColor="#cc00de"
        exitState=exitState
        
    if ('dev' in hostalias):
        nagiosServer = "printingpress.phx.gapinc.dev"
    else:
        nagiosServer = "172.16.79.85"
    payload = {
        "@type": "MessageCard",
        "@context": "http://schema.org/extensions",
        "summary": hostalias + ' is ' + hoststate,
        "themeColor": stateColor,
        "sections": [
            {
                "title": hostalias + ' is ' + hoststate,
                "facts": [
                    {
                        "name":"Status Info",
                        "value":hostoutput
                    },
        #            { 
        #                "name": "Printing Press URL", 
        #                "value": "[Thruk](http://%s/thruk/#cgi-bin/extinfo.cgi?type=%s&host=%s)"%(nagiosServer,exitState,hostalias)
        #            }
                ]
            }
        ]
    }
    print payload["sections"][0]["facts"]
    postToAlerts(webhook,payload)

def buildJson(webhook,servicedesc,hostalias,servicestate,exitState,stateColor):
    if ('dev' in hostalias):
        nagiosServer = "printingpress.phx.gapinc.dev"
    else:
        nagiosServer = "printingpress.phx.gapinc.com"
    payload = {
        "@type": "MessageCard",
        "@context": "http://schema.org/extensions",
        "summary": servicedesc + ' on ' + hostalias + ' is ' + servicestate,
        "themeColor": stateColor,
        "sections": [
            {
                "title": servicedesc + ' on ' + hostalias + ' is ' + servicestate,
                "facts": [
                    {
                        "name":"Status Info",
                        "value":serviceoutput
                    },
         #           { 
         #               "name": "Printing Press URL", 
         #               "value": "[Thruk](http://%s/thruk/#cgi-bin/extinfo.cgi?type=%s&host=%s&service=%s)"%(nagiosServer,exitState,hostalias,servicedesc)
         #           }
                ]
            }
        ]
    }
    postToAlerts(webhook,payload)
    

def postToAlerts(webhook,payload):
    r = requests.post(webhook,json=payload)
    print r.status_code

main()

```
