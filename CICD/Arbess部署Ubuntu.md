# Tiklab-Arbess部署

## 服务配置





## 服务启动

```properties
cd /mnt/dockerImageData/cicd/tiklab/tiklab-arbess/bin
./startup.sh
```



## UI 配置修改

- **webpack.prod.js**
  - **移除 optimize-css-assets-webpack-plugin 增加  css-minimizer-webpack-plugin**



## UI 启动打包

```properties
cd /mnt/dockerImageData/cicd/tiklab/tiklab-arbess-ui
# 安装
npm install
# 开发
npm run arbess-start
# 打包
npm run prod
```

