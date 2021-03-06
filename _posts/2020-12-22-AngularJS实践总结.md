---
layout: post
title: AngularJS实践总结
excerpt: "工作稳定下来后，新公司的技术栈是AngularJS，现在已做完一个项目，提出一些总结教训和学习经验。"
categories: [leonids]
comments: true
---
技术栈是AngularJS、Electron，组件用的是AntdV10.2。需要支持国际化。
#写在前面的一些感想
Angular.json其实就相当于从前的一些webpack.config.js配置文件，它会规定一些dev配置以及production配置，实际上就是一个封装的webpack，需要仔细阅读源码把每一个属性的作用的搞懂后才不至于在部署打包的时候踩坑。
## 修改ng-zorro-antd组件的文案
测试阶段发现了Upload组件的upload图标鼠标移动上去后的文案是简体的下载文件，实际上应该要展示繁体的。按理来说在App.module.ts那里设置了国际化语言后这里的文案应该不需要再重新管了，但是由于搭建项目不是我负责的，我接手的时候项目整体框架已经搭建完毕，根据文档所述也有诸如
{% highlight js%}
{ provide: NZ_I18N, useValue: zh_TW },
{% endhighlight %}
这样的配置，实际实践的时候并没有生效，查阅issue后初步认为是源码的错误导致的，如果升级到V11版应该可以解决这个问题，但是因为测试阶段已近尾声，且组件升级后Angular必须也要升级到V11版，风险太大，阅读Upload组件源码后确定了临时的解决方案：
{% highlight js%}
// 配置Upload组件文案的临时解决方案，如升级到ng-zorro的v11版可在provider中配置
Object.assign(zh_TW.Upload, {downloadFile: '下載檔案'});
{% endhighlight %}
key值在ng-zorro-antd/i18n文件夹内可以找到。
## 部署环境变量
根据当前开发环境的变量是production或development决定相应的逻辑: 
{% highlight js%}
import { AppConfig } from './../environments/environment';
if (AppConfig.environment === 'LOCAL' && !AppConfig.production) {
  // do something
}
{% endhighlight %}
## 修改ng-zorro-antd主题色
因为Antd的主题色用的是less编译，但本项目用的是sass，虽然Angular.json中会定义是用sass还是less编译，但是似乎只能二选一，其实对于当前这个项目来说用sass和less并没有什么差异，甚至连作用域都没用到。。但是没办法项目已经搭建
好了，遂研究如何修改主题色。不知不觉地就走进了一个误区，想着法要让sass和less在这一个项目中共存，几经搜索都没找到很权威的说法，虽然最后找到了一个插件，但是这个插件的实现需要另外写webpack配置文件，既然我都写
webpack配置文件了，那还不如直接就在webpack里改样式呢，又简单又直接！具体配置如下：
{% highlight js%}
"build": {
          "builder": "@angular-builders/custom-webpack:browser",
          "options": {
            "customWebpackConfig": {
                          "path": "./webpack.config.js",
                          "mergeStrategies": {
                            "module.rules": "append"
                          },
                          "replaceDuplicatePlugins": true
                        }
            }
          }
     
{% endhighlight %}
webpack配置文件:
{% highlight js%}
module.exports = {
  module: {
    rules: [
      {
        test   : /\.less$/,
        loader: 'less-loader',
        options: {
          lessOptions: {
            modifyVars: { // 修改主题变量
              'primary-color': '#2F54EB',
              'desk-background': '#EEEFF1',
              'danger-color': '#CF1322',
              'btn-border-radius-base': '4px',
              'font-family': '-apple-system, Microsoft YaHei, BlinkMacSystemFont, Segoe UI, Roboto, Helvetica Neue, Arial,Noto Sans, sans-serif, Apple Color Emoji, Segoe UI Emoji, Segoe UI Symbol, Noto Color Emoji',
            },
            javascriptEnabled: true
          },
        }
      }
    ],
  },
  target: 'electron-renderer'
};
{% endhighlight %}
## @ngx-translate的同步异步问题
ngx-translate要根据key值获取翻译的结果有两个方法，一个是：
{% highlight js%}
this.translate.get(['endBtn', ...keys] ).subscribe()
{% endhighlight %}
方法，一个是：
{% highlight js%}
this.translate.instant('yourKey')
{% endhighlight %}
这两个方法的区别就是Observe对象那个方法是异步获取的，另一个是同步获取的，因翻译值都是存储在json文件中，读取文件本身是个异步的过程，所以保险起见还是使用subscribe方法读取，实际使用过程中确实偶有发生没有取到翻译值的问题
## Electron应用中下载文件
{% highlight js%}
const link = document.createElement('a');
link.href = 'yourFileUrl'';
link.download = name;
link.click();
{% endhighlight %}
## 与Electron通信
{% highlight js%}
this.electronService.ipcRenderer.on('setting', (event, action) => { // 监听Electron的返回，监听必须写在发送前面
Object.assign(this, { startAuto: action.startAuto, shortcutKey: action.shortcutKey, version: action.version });
});
this.electronService.ipcRenderer.send('getConfig');
{% endhighlight %}
## CMDB应用（待补充）
字段都是客户自己配置的，且客户要求实现字段要实现一些简单公式功能，例如sum(A1, B1)，或大于x(数字)时颜色显示为红色，或计算某数组某个值的总和。经研究使用new Function功能可以运行js脚本，用特殊符号（我是用[*A*]）去标记参数，然后用正则表达式替换掉参数传给new Function。初步验证了一版这个方案是可行的，等彻底完成一个重要模块后再总结，感觉这个功能还是挺有挑战性的~
