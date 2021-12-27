# 一个转盘

没啥特别的抽奖转盘。可解决诸如中午吃啥的问题。

[演示](https://wheel.bakaya.ro/example)

## 依赖

[nodejs](https://nodejs.org/)，反正往新了装就是了。

## 打包

```
npm ci
npm run build
```

## 部署

纯静态前端，由于路由使用HTML5 History 模式，需要后台配置支持。[详见](https://router.vuejs.org/zh/guide/essentials/history-mode.html)

nginx的例子：

<pre>
location / {
  root <i>你的部署路径</i>;
  try_files $uri $uri/ /index.html;
}
</pre>