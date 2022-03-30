# Tips

## Chrome Your browser is managed
在浏览器地址栏输入：
chrome://management/ chrome://policy/ 注意到 policy URLBlocklist 禁用了安装扩张的相关rul地址，可以删除以下注册表项解决；

```
HKEY_CURRENT_USER\SOFTWARE\Policies\Google\Chrome\URLBlacklist
```
