---
title: hexoblog
date: 2022-04-20 12:44:03
tags:
---

# Hexo 博客搭建

> https://hexo.io/zh-cn/docs/index.html

1. 安装 nodejs（https://github.com/nodesource/distributions#debinstall），如果直接使用apt安装的版本太低！所以要先添加官方的源，但是官方的源很慢，也可以选择[下载预编译包](https://nodejs.org/zh-cn/download/)，然后手动添加

   ```bash
   curl -fsSL https://deb.nodesource.com/setup_17.x | sudo -E bash -
   sudo apt-get install -y nodejs
   ```

2. 安装 hexo，下面的步骤安装的目录为node所在的路径，还无法在其他目录下使用，因此添加软链接

   ```bash
   npm install -g hexo-cli
   sudo ln -s /home/yogurt/AppImage/node-v16.14.2-linux-x64/bin/hexo /usr/local/bin/hexo
   ```

3. 初始化博客文件夹

   ```bash
   hexo init hexo
   cd hexo
   ```

4. 安装主题，

   > https://zhangc233.github.io/categories/
   >
   > https://github.com/jerryc127/hexo-theme-butterfly
   >
   > 应用主题参考：https://butterfly.js.org/posts/21cfbf15/#%E5%AE%89%E8%A3%9D

   ```bash
   git clone -b master https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly

   # 还需要安装一些插件，要在博客的主文件夹中运行，不要用sudo！
   npm install hexo-renderer-pug hexo-renderer-stylus --save
   sed -i 's/landscape/butterfly/g' _config.yml
   ```

5. 主题相关的配置查看官方给的系列说明：https://butterfly.js.org/posts/4aa8abbe/#%E7%B6%B2%E7%AB%99%E8%B3%87%E6%96%99

6.

## 汇总脚本：

```bash
sudo pwd	# 先输一次密码
# ------------------
# 安装 nodejs
wget https://nodejs.org/dist/v16.14.2/node-v16.14.2-linux-x64.tar.xz
unar node-v16.14.2-linux-x64.tar.xz
mv node-v16.14.2-linux-x64 ~/AppImage/
sudo ln -s /home/yogurt/AppImage/node-v16.14.2-linux-x64/bin/node /usr/local/bin/node
sudo ln -s /home/yogurt/AppImage/node-v16.14.2-linux-x64/bin/npm /usr/local/bin/npm

# 安装 hexo
npm install -g hexo-cli
sudo ln -s /home/yogurt/AppImage/node-v16.14.2-linux-x64/bin/hexo /usr/local/bin/hexo

# 初始化博客文件夹
hexo init hexo
cd hexo

# 安装主题
git clone -b master https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
npm install hexo-renderer-pug hexo-renderer-stylus --save

# 应用主题
sed -i 's/landscape/butterfly/g' _config.yml

hexo new page tags
hexo new page categories
hexo new page link

```


