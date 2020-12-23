# nginx.orgが提供するRPMを利用してnginxをインストールする場合の注意点
nginx.orgが提供するRPMファイルに含まれる`/usr/lib/systemd/system/nginx.service`ファイルには問題があり、終了時にUNIXドメインソケットファイルが削除されません。このため、UNIXドメインソケットでlistenしている環境でnginxを再起動しようとすると`Address already in use`エラーになります。EPELのnginxパッケージにはこの問題がありません。

```
[emerg] 103728#0: bind() to unix:/run/nginx.sock failed (98: Address already in use)
```

# 修正方法
このリポジトリにある`nginx.service`ファイルを`/etc/systemd/system/nginx.service`に格納し、`systemd daemon-reload`を実行します。

# 不具合について
nginx.serviceの中を見ると、`ExecStop`がありません。これは`systemctl stop`で実行されるコマンドです。このため`KillSignal`で指定されたSIGQUITが発行されます。SIGQUITだとソケットファイルが削除されないため、次に起動したときに`Address already in use`になってしまいます。
```
[Unit]
Description=The nginx HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
# Nginx will fail to start if /run/nginx.pid already exists but has the wrong
# SELinux context. This might happen when running `nginx -t` from the cmdline.
# https://bugzilla.redhat.com/show_bug.cgi?id=1268621
ExecStartPre=/usr/bin/rm -f /run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t
ExecStart=/usr/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
KillSignal=SIGQUIT
TimeoutStopSec=5
KillMode=mixed
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

この`nginx.service`ファイルは以下の`ExecStop`を追加し、TERMシグナルで終了するようにしてあります。TERMシグナルで終了した場合はソケットファイルが削除されるため、正常にnginxの再起動ができるようになります。
```
ExecStop=/bin/kill -s TERM $MAINPID
```

# 注記
将来的にnginx側でこの問題が修正される可能性があります。その時はなるべく対応しようと思いますが、気が付かなかったら教えてください。

# ライセンス
ライセンスはもとのnginxライセンスに従います。