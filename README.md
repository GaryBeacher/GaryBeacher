# 背景
学活线现有项目接入的监控服务为阿里云arms服务，如下图：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/620070/1614567866250-cfc61ca7-de6d-4ba8-909d-5c03fffdbe51.png#align=left&display=inline&height=261&margin=%5Bobject%20Object%5D&name=image.png&originHeight=755&originWidth=2553&size=172302&status=done&style=stroke&width=884)
Arms前端性能监控的JS错误诊断诊断页面，可以看到错误发生的次数、影响用户数以及错误分布情况等信息，开发者可以根据这些统计数据更好地进行性能优化。
现在监控SDK 的设计是针对arms的功能性补充。
根据业务我们可以知道主要场景是这样的，基础业务板块在运行过程中出现的错误异常情况，然后根据抓取到的内容，分析用户行为，以便于复现异常发生的情况。
# 为什么要自己做
目前市场中的前端项目异常监控系统，功能繁多，有的会绑定性能监控，用户画像等功能，我们可能需要的只是其中某一项功能而已。还有存在另一种现象，一位前端工程师，肯定会不止一次去解决一些顽固的线上问题，可能某些错误或者异常现象是由于某个用户的误操作引起的，我们也想方设法复现用户的bug，但是结果可能都不太理想。 因为它发生于用户的一系列操作之后。错误的原因可能源于机型，网络环境，复杂的操作行为等等，甚至某些错误可能仅仅是因为接口返回的数据类型有问题而导致页面白屏。

所以我们是否又必要去针对我们所需要的内容去设计一套和我们本身业务更加匹配的SDK监控系统。
# 基础设计流程

## SDK设计流程
> 本次调研设计报告，时间较为充足，在原本计划中是希望能满足补充arms的功能，也就是希望能在获取错误的时候可以针对用户的上下文信息进行获取，并且针对接口请求汇总时可以获取到返回数据的基本信息。但是我考虑到错误信息的完整度，本次设计就较为全面的分析了错误异常的范围及所需要信息的获取方式。

### 采集数据内容
| **用户行为数据** | **错误数据** |
| :---: | :---: |
| 用户在页面中的操作：点击、滑动等 | FEjs错误信息（分类见下方） |
| 页面路由跳转：Hash或history，SPA/MPA项目页面跳转，页面堆栈信息，访问轨迹等 | RD服务错误信息，如服务不可用或业务数据类型错误等 |
| 网络接口请求，接口请求顺序，接口返回结果类型等 | APP错误 |
| 控制台输出内容，log/info信息等 | 接口请求错误 |
| 自定义信息，自定义事件 |  |

### 
### 数据内容
![](https://cdn.nlark.com/yuque/0/2021/jpeg/620070/1615257074089-a6496a0a-4eef-4649-935d-956019034b9c.jpeg)

### 错误分类
![](https://cdn.nlark.com/yuque/0/2021/jpeg/620070/1615257073889-7bd5b706-0709-4b4a-a434-9cd50001581e.jpeg)## 异常级别分类优先级
正常情况，错误异常的分类会按照错误造成的后果进行分类，一般而言，我们会将收集信息的级别分为info，warn，error等，并在此基础上进行扩展。
异常错误引起的后果和基础的紧急情况可以分为四种情况严重-重要轻微-重要轻微-不重要严重-不重要。
有些异常，虽然发生了，并且也触发了sdk，但是并不影响用户的正常使用，甚至用户其实并没有感知到，理论上，这个错误应该被修复，但是实际上相对于其他异常而言，优先级可以被降低，待其他错误被处理后在进行处理。
## 异常日志前端存储
我们根据分类可以得知我们要拿到的异常日志信息，不仅仅包括本身错误相关的内容，最主要的是用户的上下文信息。因为单独的一条异常日志无法帮助我们快速定位问题的根源。我们需要获取用户的基础操作日志，这部分内容包括用户相关的点击操作和页面中的网络接口请求堆栈信息，还有包括访问页面路径信息等等内容。一方面我们不能每次用户在进行操作的时候就会发送一次请求，如果用户访问量大的话，就相当于是对服务器进行攻击。另一方面如果我们把所有的用户日志用一个变量保存起来，这样就会挤爆内存，并且会存在如果是在站外访问的话，用户如果一刷新，那这些数据就会丢失。

所以我们会先把这些日志存储在用户客户端本地里面，达到一定条件之后在同时打包上传一组。
技术方面选择中，如下方表格



| **存储方式** | cookie | localStorage | sessionStorage | IndexedDB |
| --- | --- | --- | --- | --- |
| **数据格式** | string | string | string | object |
| **容量** | 4k | 5M | 5M | 500M |
| **同步/异步** | 同步 | 同步 | 同步 | 异步 |
| **读写速度** | 快 | 读快写慢 | 读快写慢 | 读慢写快 |



根据上方对比，初步采用方案是indexDB。indexDB是分库的，每个库又分store，还能按照索引进行查询，比localStorage更适合做结构化数据管理。但是它有一个缺点，就是api非常复杂，不像localStorage那么简单直接。所以我们还需要对indexDB的基础api进行封装。
## 错误上报
### 上报方式 
### 上报时机
#### 实时上报
收集到错误日志后，立即触发上报函数。仅用于Error类异常。由于受到网络或机型等不确定因素影响，日志上报需要有一个确认的时机，只有确认服务端已经成功接收到该上报信息之后，才算完成。
#### 统一上报
一定时间段内sdk会把收集到的日志存储在本地缓存，当收集到够一定数量之后再打包一次性上报，或者按照一定的时间间隔打包上传。这相当于把多次合并为一次上报，以降低对服务器的压力。
#### 主动提交
比如页面卡死或者出现无法使用的情况，可以在界面上提供一个按钮，用户主动反馈bug。一方面来说可以加强与用户的互动，另一方面可以让用户来确认当前的异常是否完全影响了使用。
我们有一些异常是会涉及到用的一些隐私信息，所以我们可以在特殊情况下向用户发出一个请求，让用户自愿选择上传日志。 
### 上报方式
1、通过 Ajax 发送数据 因为 Ajax 请求本身也有可能会发生异常，缺点是有可能会引发跨域问题。
2、使用动态创建 img 标签的形式进行上报。 
需要解决的问题：
跨域/性能/信息量/附带上下文信息
### 采样率
如果某个网站访问量很大，假如某个页面PV达到千万量级，那么一个必然发生的错误信息就有千万条，我们可以通过具体实际的情况来设定一个采集率，比如用一个随机数，只上报百分之30的错误信息，也可以具体根据用户的某些特征来进行判定。
## 业务兼容
业务群、业务线区分，以便于后期公司内部使用
## 白名单机制
增加白名单机制，可以动态监控某个接口返回值的类型
# 待确认
1、现在后端提供的接口状态码是根据接口返回的statusCode来判断不同返回接口，比如：400，404，300，302等。
2、rrweb：打开 web 页面录制与回放的黑盒子，是否使用。
# 任务列表

- [ ] 第一期
- [ ] sdk基础设计
- [ ] 异常错误原因分类
- [ ] sdk基础模板搭建
- [ ] ajax网络请求异常捕获
- [ ] js脚本错误捕获，重写console方法
- [ ] 用户基本上下文操作
- [ ] POST上报
- [ ] 第二期
- [ ] promise异常，iframe异常
- [ ] 用户行为点击操作的获取
- [ ] SPA应用适配，针对单页面项目路由跳转兼容
- [ ] cdn静态资源异常和功能上报
- [ ] 增加上报方式，img图片形式等，增加手动弹窗上报
- [ ] react项目组件错误的捕获
- [ ] Vue组件异常的捕获
- [ ] 前端数据存储的数据结构设计
- [ ] 第三期
- [ ] 前端数据存储
- [ ] sourcemap 









































### Hi there 👋
 
## 吃饭必备

![HTML5](https://img.shields.io/badge/-HTML5-%23E44D27?style=for-the-badge&logo=html5&logoColor=ffffff)
![CSS3](https://img.shields.io/badge/-CSS3-%231572B6?style=for-the-badge&logo=css3)
![JavaScript](https://img.shields.io/badge/-JavaScript-%23F7DF1C?style=for-the-badge&logo=javascript&logoColor=000000&labelColor=%23F7DF1C&color=%23FFCE5A)
![Vue.js](https://img.shields.io/badge/-Vue.js-%232c3e50?style=for-the-badge&logo=Vue.js)
![React](https://img.shields.io/badge/-React-%23282C34?style=for-the-badge&logo=react)
![Node](https://img.shields.io/badge/-NodeJS-%23F05032?style=for-the-badge&logo=Node.js&logoColor=%23ffffff)
![Webpack](https://img.shields.io/badge/-Webpack-%232C3A42?style=for-the-badge&logo=webpack)
![ESlint](https://img.shields.io/badge/-ESLint-%234B32C3?style=for-the-badge&logo=eslint)
![Git](https://img.shields.io/badge/-Git-%23F05032?style=for-the-badge&logo=git&logoColor=%23ffffff)
![VS Code](https://img.shields.io/badge/-VSCode-%23007ACC?style=for-the-badge&logo=visual-studio-code)
![Docker](https://img.shields.io/badge/-Docker-%232081e8?style=for-the-badge&logo=docker&logoColor=fff)
![TypeScript](https://img.shields.io/badge/-TypeScript-%23031d30?style=for-the-badge&logo=typescript)
![Prettier](https://img.shields.io/badge/-Prettier-%23142027?style=for-the-badge&logo=prettier)
![TaroJs](https://img.shields.io/badge/-TaroJs-%230000b5?style=for-the-badge&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAFgAAAA8CAYAAADi8H14AAAAAXNSR0IArs4c6QAABO9JREFUeAHtW8lqFUEU9TlFBRU1Ik4IwRlRiPNCcCAuVHChS0Vw5Q/4Ee4FF/oD4l4MuNZEs3AgRg1RiKhx1jiLGs8Jr5O2X3dVdfere8u0F4rXr+/tuueeVJ+uqn6pTaqIjYyMLESp+9CWoy1Bm4Xm03pqtdr5qT4zhNA3iJ0GHEfRdqFNEcR0j7kmNMEgl4SeQtvAYoVtlODJwkml0x1DQg1yn0Me3rFY5xGM0dCK+ON1wOvxOQdNy86igHOm5MC7CP6dphiPvtHRy/6tBANoC+JO15smqcQb2Z3owPDZAV/N4Pfp6o06NxIMcum/iHY4uiCAT956XSYcwE3p22yK8ej7ib77o/5tGnwBgSGRS9ydkIdfUQEZnytw3vc0LCP1pH7g+xE5MwnGKGhH0IkoMKDPyw5Y+IzQsjH9JYBMguGj7oZmIwB0xQGUJsFj+kucqQRj9M6Gj5Pz0Owmbr9XJlDAPgP+NlOMR98w8D2N959KMAL2oBkfgPFOBI9d5GEN8GTV5RvqX/LAZFlA9vtGUrB/F4KDkQfWmDVKQyT4I/AeggQcsfxxuOdQZgbxCNe/tuTIct9POhoIRgGc4qxKBgbw/TswHLDg4N7DMkuMzf3XQ8oWHPMPQn+HY99HD9MkIsTRS7AfkuBTvs9MOZfn1GcEsxWxBv1lJ/8KwZyeNYyOFBbKEvwipU/XU3aCIQ8knJvSodknALKt3oiZU7QyVpRgrtwG0hInR/AWBM1LC1Q+5yIP04ExWU8e2L8RbJxjGzp7AP3lHkSDJQFVWX/fgB2Xu6SBRJxIlQcG/gsE8/b7SrAW05IHwrITDP3l8niHpQgNt4s8cN+X+9ZlrKj+voU8DGUljo/g3QjiC8LQzIVgjt4ym+vfcL3LLCWNm8zRy+A4wSHqLx88LoVrTs+MC5PQCeb0jCTbTEt/OT9vWB7HwY4SXF8er447Ajl2kQcuj8tIG0kqOj17DP39YuIqGsEdpiBFnwvBZeWB7/jGXvHkrNWov+wr2uwJUX+5ucOHj804d31vCzL4r8LXafCbXDdMTvpq9eUxb5H5tmBh/0vkGxTIeRK3+V1feSgRXB6HRi7rdZEHxpUx7jFbb/MyCUhwiPrLmQOL923dGL1Fl8dO2KjBLOaSU7SfoKXodm2ia2qvy/QscVnur125r8h5QZnVT85U6eF4BpyBZ2+61/vZgxjB1HpvRolQs/oDdqsSgAHf5LIuVYKRfx0aN5k07LpEUm2Ct0sUmZHjP8EZxDTjNBcxt5rRka0PtREM/eUSd6MNoCc//0Gl6PI4FyQ1goGyHS1aqucC3YRgEXkgTk2CJ7z+VpXgZ5CHwSbcBU5dqIxg6G8r0LU5IWx+kJg8ELoKwchbCXmoIsHc2Olh4VKmNYK3SRWYyHMb+lv0x32Jrty+ihMM/V0JaAvc4DU9SlR/iV6cYOSsjP5WjWD+9uwhi5Y00REMeeDrda7gNIxvL/iKXtRECUZlm9BaRCscT3Zt/FDuSJpgLf3lyO2Wo3U8U1UI7oM8lPntxDhjOY/ECIb+zgW25MvNnHALh3t/uZmFTIxgAOC7N62XrCr6S9IlCdbSX67celmshlWBYE7PUv9BRYJwEYKhv8tRzGKJglJyiC+P4xhECEZCLXlgrWoPOCaf6ATzB9JDLFTLpF469qHAJwpFev3lpEs9fwBpxe4J7SMAYgAAAABJRU5ErkJggg==) 
![UNIAPP](https://img.shields.io/badge/-UNIAPP-%23CCC?style=for-the-badge&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAABCElEQVRoge3YMa4BURSH8Y8o7UAp0WgkotBZwluAfhqlZSgUGr23ENUUCpppJnTswAIUSCaTiziZJ8d9/193zdzrfMltABF5plb+oLscDoAV0Pn8OC/lwDhL0k35QT3wstcIuM61Cj0IhXiNuAvOFwr5SgrxRiHeKMSbhnHfAVgU1i1gajhnBpwK6wnQtgxkDTlmSTq/L7rLYQ9byG+WpLvCOT8YQ6K5WgrxRiHeKMQbhXijEG8U4o1CvIkmxPrDquwMrI37KlFJSJake2BUxVlW0VytaEKsV6t5+8Ohak3rRmtIH9hav/QvRHO1FOKNQrwJheQfn+I9wflCIeNHLzuQc51PRP6rC1ZeIm1I8cC5AAAAAElFTkSuQmCC&logoColor=fff)


<img src="polarisxu-qrcode-small.jpg" alt="polarisxu" height="120" align="center"/>

<img src="https://github-profile-trophy.vercel.app/?username=polaris1119&theme=flat&column=7" alt="logo" height="160" align="center" style="margin: auto; margin-bottom: 20px;" />

<!--
**polaris1119/polaris1119** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

![](https://github-readme-stats.vercel.app/api?username=GaryBeacher&theme=dark)
