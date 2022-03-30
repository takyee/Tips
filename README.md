# Tips

## Chrome Your browser is managed
在浏览器地址栏输入：
chrome://management/ chrome://policy/ 注意到 policy URLBlacklist 禁用了安装扩张的相关rul 地址，可以删除以下注册表项；

```
HKEY_CURRENT_USER\SOFTWARE\Policies\Google\Chrome\URLBlacklist
```
