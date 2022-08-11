# Tips

## 解决Chrome Your browser is managed
在浏览器地址栏输入：
chrome://management/ chrome://policy/ 注意到 policy URLBlocklist 禁用了安装扩张的相关rul地址，可以删除以下注册表项解决；

```
HKEY_CURRENT_USER\SOFTWARE\Policies\Google\Chrome\URLBlacklist
```
## Linux 全局代理设置
```
export http_proxy="http://10.10.100.200:8118"
export https_proxy="http://10.10.100.200:8118"
no_proxy="hocn-int-wework.mysql.rds.aliyuncs.com,dysmsapi.aliyuncs.com,wework.homag.com.cn,127.0.0.1,localhost" //支持域名通配符写法例如 .qq.com
```
取消设置
```
export http_proxy=""
export https_proxy=""
no_proxy=""
```
