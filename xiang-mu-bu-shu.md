---
description: 项目线上环境部署
---

# 项目部署

1.mvn package命令打包 （打包之前需要更改数据库线上地址、redis线上地址）

2.项目包上传到服务器目录

3.命令行跳转到对应目录，运行nohup java -jar xxx.jar &  \(xxx.jar对应项目包名\)

4.查看日志是否正常

