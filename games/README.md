# 喵啊~

## 主题

https://hexo.io/themes/

https://butterfly.js.org/posts/21cfbf15/

## TODO

1. 卡牌对战的战斗模块
2. MMO的场景
3. AFK战斗
4. SLG大地图
5. effective Java
6. Java游戏服务器架构
7. XDB -> KDB
8. Netty源码
9. HikariCP源码
10. Quartz源码
11. 压测
12. 合服
13. 热更
14. 面经
15. 蜜月?

## 命令

### Hexo 常用命令

```shell
hexo new "postName" 新建文章
hexo new page "pageName" 新建页面
hexo generate 生成静态页面至public目录
hexo server 开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy 将.deploy目录部署到GitHub
hexo help 查看帮助
hexo version 查看Hexo的版本
```

### Hexo 8 及 strip-ansi 7 修复方法

```shell
# 安装最新 Node 20 LTS
nvm install 20.19.1
nvm use 20.19.1
# 删除依赖和锁文件
rm -rf node_modules package-lock.json

# 安装 Hexo 8 和固定 strip-ansi 7
npm install hexo@8 strip-ansi@7 --save-exact

# 重新安装其它依赖
npm install
```