# 在android中使用OkHttp时的pin验证

如果在android中并使用http，那么pin的验证流程是这样的：

1. 使用Network Security Configuration中的配置验证pin
2. 使用certificatePinner验证pin
