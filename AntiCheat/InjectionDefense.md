

### 导入表、输入法dll注入

- 程序启动时检查exe、dll签名及证书，
- 用户进程内hook loadlibrary，当载入的dll不在白名单内的时候，拒绝载入
- 驱动内监控 用户对关键系统dll以及用户软件相关exe、dll的修改，拒绝修改？（filesystem minifilter）



对于导入表注入，由于dll加载时，程序入口函数还未执行，无法hook loadlibrary，因此只能通过驱动监控dll的载入，如果载入了不在白名单里的dll，则通知用户进程。另外安全软件通常不会修改用户文件，因此程序如果收到不明dll载入，可以直接退出，对兼容性影响较小

对于输入法等在程序启动后进行注入的方法，可通过hook loadlibray来监控