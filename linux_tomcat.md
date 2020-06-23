# Install Tomcat on CentOS 7

1. Download Tomcat 9

    ```bash
    cd ~
    wget https://www-us.apache.org/dist/tomcat/tomcat-9/v9.0.17/bin/apache-tomcat-9.0.17.tar.gz
    ```

1. Unzip and move

    ```bash
    mkdir /opt/tomcat
    tar -xf apache-tomcat-9.0.17.tar.gz
    sudo mv apache-tomcat-9.0.17 /opt/tomcat/
    ```

1. Create a link point to tomcat dir

    ```bash
    sudo ln -s /opt/tomcat/apache-tomcat-9.0.17 /opt/tomcat/latest
    ```

1. Create an account to run Tomcat

    ```bash
    sudo useradd -r tomcat --shell /bin/false
    ```

1. Modify access rights

    ```bash
    cd /opt/tomcat/latest
    sudo chown tomcat: /opt/tomcat/latest   #一併變更owner與group為tomcat
    sudo chmod g+rwx conf   #conf dir加上group讀寫進的權限
    sudo chmod -R g+r conf   #conf dir裡所有的檔案加上group讀的權限
    sudo chmod g+rwx bin
    shdo chmod -R g+x bin
    ```

1. Setup a Systemd unit file for Apache Tomcat

    ```bash
    sudo nano /etc/systemd/system/tomcat.service
    ```

    Paste following content to the file:

    ```bash
    [Unit]
    Description=Tomcat 9 servlet container
    After=network.target

    [Service]
    Type=forking

    User=tomcat
    Group=tomcat

    Environment="JAVA_HOME=/usr/lib/jvm/jre"
    Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom"

    Environment="CATALINA_BASE=/opt/tomcat/latest"
    Environment="CATALINA_HOME=/opt/tomcat/latest"
    Environment="CATALINA_PID=/opt/tomcat/latest/temp/tomcat.pid"
    Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

    ExecStart=/opt/tomcat/latest/bin/startup.sh
    ExecStop=/opt/tomcat/latest/bin/shutdown.sh

    [Install]
    WantedBy=multi-user.target
    ```

    Then execute following commands:

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable tomcat
    sudo systemctl start tomcat

    sudo systemctl status tomcat   #check tomcat running status
    ```

1. Adjust firewall

    ```bash
    sudo firewall-cmd --zone=public --permanent --add-port=8080/tcp
    sudo firewall-cmd --reload
    ```

1. 如果不想將8080 port export出去的話, 可以使用 nginx 的 reverse proxy 反向代理功能將特定的url導向至8080. 以下範例為將url: \<server>/tomcat 導向至8080:

    ```yml
    server {
        listen        80 default_server;
        ...
        location /tomcat/ {
            proxy_pass         http://localhost:8080/;
            proxy_http_version 1.1;
            proxy_set_header   Upgrade $http_upgrade;
            proxy_set_header   Connection keep-alive;
            proxy_set_header   Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
        }
    }
    ```

1. Deploy java web application to Tomcat
    1. 打包war檔案
    1. 將war檔案複製到Tomcat/webapps目錄底下

        ```bash
        scp my-webapp.war mac@172.31.69.14:/opt/tomcat/latest/webapps
        ```

        正常狀況tomcat偵側到war檔後, 會自動將其解壓縮並將之載入. 若要手動啟動, 可使用以下指令:

        ```bash
        curl -u mac:password http://localhost:8080/manager/text/start?path=/my-webapp
        ```

    1. 若 web application 有使用 JPA, hibernate, 則web app會在一啟動時就嚐試連接資料庫. 此時若資料庫連接有問題則web app會啟動失敗.
    1. 資料庫若是在另一台電腦時, 要先確認資料庫的port是否有被防火牆擋掉. 若有的話要先開啟防火牆. 以下為win10, mssql為例:
        1. Navigate to Control Panel, System and Security and Windows Firewall.
        1. Select Advanced settings and highlight Inbound Rules in the left pane.
        1. Right click Inbound Rules and select New Rule.
        1. Add the port you need to open and click Next.
        1. Add the protocol (TCP or UDP) and the port number into the next window and click Next.
        1. Select Allow the connection in the next window and hit Next.
        1. Select the network type as you see fit and click Next.
        1. Name the rule something meaningful and click Finish.
