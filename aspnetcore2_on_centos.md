# Running Asp.net core 2.x on Nginx in CentOS 7

## Install Nginx

1. 安裝 Nginx

    ```bash
    sudo yum update
    sudo yum install epel-release
    sudo yum install nginx
    ```

1. 因 CentOS 7 的防火牆預設會將http,https的port關閉, 因此得執行以下命令將之開啟

    ```bash
    sudo firewall-cmd --permanent --zone=public --add-service=http
    sudo firewall-cmd --permanent --zone=public --add-service=https
    sudo firewall-cmd --reload
    ```

1. 啟動 Nginx 服務

    ```bash
    sudo systemctl enable nginx
    sudo systemctl start nginx
    ```

    執行完成可執行 `sudo systemctl status nginx` 查看service的執行狀態:

    ```bash
    [mac@localhost ~]$ sudo systemctl status nginx
    ● nginx.service - The nginx HTTP and reverse proxy server
    Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
    Active: active (running) since Tue 2018-10-16 19:45:41 CST; 2min 22s ago
    Process: 41508 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
    Process: 41506 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 41502 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Main PID: 41510 (nginx)
        Tasks: 2
    CGroup: /system.slice/nginx.service
            ├─41510 nginx: master process /usr/sbin/nginx
            └─41511 nginx: worker process

    Oct 16 19:45:40 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
    Oct 16 19:45:41 localhost.localdomain nginx[41506]: nginx: the configuration file /etc/nginx/nginx... okOct 16 19:45:41 localhost.localdomain nginx[41506]: nginx: configuration file /etc/nginx/nginx.con...fulOct 16 19:45:41 localhost.localdomain systemd[1]: Failed to read PID from file /run/nginx.pid: Inv...entOct 16 19:45:41 localhost.localdomain systemd[1]: Started The nginx HTTP and reverse proxy server.
    Hint: Some lines were ellipsized, use -l to show in full.
    ```

1. 將檔案從windows複製到Linux

使用 `scp` 指令透過 SSH 將檔案複製到 Linux.

```scp -r todo* mac@172.31.69.12:/var/www/machung.dojo.net/```

> 安裝 Win32 OpenSSH: [https://github.com/PowerShell/Win32-OpenSSH/wiki/Install-Win32-OpenSSH].

## SELinux issues

SELinux 為 __Security-Enhanced Linux__ 的縮寫, 從 RHEL 6.6 / CentOS 6.6 之後的版本均有加上此功能. 其設定值會對 Nginx 的服務存取會有些嚴格的限制. 可參考[官方說明文件](https://www.nginx.com/blog/using-nginx-plus-with-selinux/).

Nginx 使用的 SELinux label 為 __*httpd_t*__, 最快速的解決方式就是將 httpd_t label 加入至允許清單中. 指令為 `sudo semanage permissive -a httpd_t`. 若要移除則為 `sudo semanage permissive -d httpd_t`.

如果不想直接將 httpd_t 加入允許清單, 則可以針對各別問題分別設定. 以下為一些常見的錯誤與解決方式:

### 網址為反向代理時出現 http 502

SELinux 預設是不允許 Nginx 連結至外部伺服器或是 TCP port. 解決方式為:

`setsebool -P httpd_can_network_connect on`

## 安裝 ASP.NET Core Vue template

執行 `dotnet new --install dotnet new --install Microsoft.AspNetCore.SpaTemplates::*` 安裝所有SPA Framework範本

## dotnet 執行時如何指定要綁定的 port

執行 dotnet core kestrel 時, 會下 `dotnet webapp.dll` 指令來執行. 預設 kestrel 會綁定 `http://localhost:5000, https://localhost:5001` 這兩個ports, 如果要指定綁定的port, 有以下做法:

1. ASPNETCORE_URLS 環境變數: 直接設定環境變數的值, 不同的端點(port)以 __;__ 隔開. ex: `http://localhost:8000; https://localhost:8001`
1. --urls 命令列引數: 執行 `dotnet` 指令時加上 `--urls` 參數. ex: `dotnet webapp.dll --urls http://localhost:8088`.
1. urls 主機組態索引鍵: 在外部的 json 檔案中指定 `urls` 屬性, 並透過程式碼載入此json檔案. 例如有一個 __hostsettings.json__ 檔案內容為:

    ```json
    {
        urls: "http://*:8088"
    }
    ```

    ```csharp
    public static IWebHostBuilder CreateWebHostBuilder(string[] args)
    {
        var config = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("hostsettings.json", optional: true)
            .AddCommandLine(args)
            .Build();

        return WebHost.CreateDefaultBuilder(args)
            .UseUrls("http://*:5000")
            .UseConfiguration(config) // 載入json組態檔覆寫掉上一行的 UseUrls()
            .Configure(app =>
            {
                app.Run(context =>
                    context.Response.WriteAsync("Hello, World!"));
            })
            .Build();
    }
    ```

1. UseUrls 擴充方法: 透過程式碼的方式指定:

    ```csharp
    public static IWebHostBuilder CreateWebHostBuilder(string[] args)
    {
        return WebHost.CreateDefaultBuilder(args)
            .UseUrls("http://*:5000")
            .Build();
    }
    ```

1. 修改 __appsettings.json__, 加上 `kestrel` 組態設定: `WebHost.CreateDefaultBuilder` 預設會呼叫 `serverOptions.Configure(context.Configuration.GetSection("Kestrel"))` 以載入 Kestrel 組態. Kestrel 可以使用預設的 HTTPS 應用程式設定組態結構描述. 設定多個端點, 包括 URL 和要使用的憑證.

    ```json appsettings.json
    {
        // ......,
        "Kestrel": {
            "EndPoints": {
                "Http": {
                "Url": "http://localhost:5000"
            },

            "HttpsInlineCertFile": {
                "Url": "https://localhost:5001",
                "Certificate": {
                    "Path": "<path to .pfx file>",
                    "Password": "<certificate password>"
                }
            },

            "HttpsInlineCertStore": {
                "Url": "https://localhost:5002",
                "Certificate": {
                    "Subject": "<subject; required>",
                    "Store": "<certificate store; defaults to My>",
                    "Location": "<location; defaults to CurrentUser>",
                    "AllowInvalid": "<true or false; defaults to false>"
                }
            },

            "HttpsDefaultCert": {
                "Url": "https://localhost:5003"
                },

                "Https": {
                "Url": "https://*:5004",
                "Certificate": {
                    "Path": "<path to .pfx file>",
                    "Password": "<certificate password>"
                    }
                }
            },
            "Certificates": {
                "Default": {
                    "Path": "<path to .pfx file>",
                    "Password": "<certificate password>"
                }
            }
        }
    }
    ```

    若 `appsettings.json` 中有定義 kestrel 組態, 則此組態值會覆寫掉上述四種方式的綁定.

## 設定 nginx proxy config

在 CentOS 中, nginx 的預設組態會載入 __/etc/nginx/conf.d/*.conf__ 自定組態檔:

/etc/nginx/nginx.conf

```yml
# ...

http {
    # ...

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    # ...
}
```

因此我們可以將 asp.net core 的組態依 __server_name__ 命名後放在此目錄. 例如 `machung.dojo.net.conf` 這樣. 依[官方文件說明](https://docs.microsoft.com/zh-tw/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-2.1&tabs=aspnetcore2x), `machung.dojo.net.conf` 的內容如下:

```yml
server {
    listen        80;
    server_name   machung.dojo.net;
    location / {
        proxy_pass         http://localhost:5000;
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

當 nginx 設定以上組態後, 當 http request headers 中的 __host__ 值為 __machung.dojo.net__ 時, 變會將request轉去__proxy_pass__ 設定的url來負責處理 `http://localhost:5000`.

在 dotnetcore 的 `Starpup.Configure` 中需設定 __ForwardedHeaders__:

```csharp
app.UseForwardedHeaders(new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
});
```

### 如何設定 __server_name__ 與 ip 的關係

因為我們的組態是綁定 host name, 因此我們在訪問我們的 dotnetcore 網站時, url 就必須是 `http://machung.dojo.net` 開頭, nginx 才會將request轉至 dotnetcore 處理. 那麼我們要如何在本機測試時, 去讓browser知道當我們在url上輸入 machung.dojo.net 時, 要導向 linux 這台的 ip 呢? 其實還滿簡單的, 就是編輯 C:/Windows/system32/drivers/etc/hosts 即可.

```ini
# localhost name resolution is handled within DNS itself.
#   127.0.0.1       localhost
#   ::1             localhost

172.31.69.12    machung.dojo.net
```
