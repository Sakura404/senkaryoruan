# 博客项目vue-cli 迁移到nuxt.js实现ssr

### 前言

​	自己的个人博客早期由使用vue.js和Spring Boot实现的，想要被搜索引擎搜索到，但是到百度bing添加收录发现，搜索引擎爬虫爬取不到任何内容。众所周知，使用vue实现的单页面应用是不利于seo的，无奈下只能通过ssr（服务端渲染）来优化seo。但是使用vue脚手架开发实现ssr较为复杂 ，自己实现发生太多超出自己处理能力的错误，也曾想将项目迁移到vite下，目的也仅仅为ssr，迁移过程倒是顺利，本地渲染也挺正常的。但是实现ssr同样发生许多未知错误。最后发现nuxt.js能比较简单实现ssr。

### 构建新的nuxt.js项目

由于nuxt.js项目目录的结构与vue-cli大相径庭所以就不在原项目基础上迁移。

首先创建通过命令创建新项目

```bash
npx create-nuxt-app myblog-nuxt
```

随后出现创建目录的选项，根据自己需求选择，渲染模式选择ssr，ui组件库自己博客使用的是vuetiy。

```bash
create-nuxt-app v4.0.0
✨  Generating Nuxt.js project in myblog-nuxt
? Project name: myblog-nuxt
? Programming language: JavaScript
? Package manager: Npm
? UI framework: Vuetify.js
? Nuxt.js modules: (Press <space> to select, <a> to toggle all, <i> to invert selection)
? Linting tools: (Press <space> to select, <a> to toggle all, <i> to invert selection)
? Testing framework: None
? Rendering mode: Universal (SSR / SSG)
? Deployment target: Server (Node.js hosting)
? Development tools: (Press <space> to select, <a> to toggle all, <i> to invert selection)
? What is your GitHub username? sakura404
? Version control system: None
```

创建完成后进入目录。

```
├── components # 组件目录，支持自动导入
├── layouts # 布局目录 
├── assets # 静态资源目录 
├── nuxt.config.ts  # Nuxt 配置文件，
├── package.json
├── pages # 基于文件的路由
└── plugins #插件 
```

### 原项目文件迁移

将原来项目src文件夹整个复制过来到nuxtjs项目根目录下。由于原项目体量较为庞大和混乱（博客系统是刚接触vue的时候边学习边写的，项目周期长，不同时期的代码风格不同）。为了不破坏组件之间的引用，将src文件夹设置为nuxtjs入口。

```javascript
// nuxt.config.js

export default {
  srcDir: 'src/',
}
```

### 依赖安装

将原项目安装的依赖复制到packge.json中执行npm install

```
  "@packy-tang/vue-tinymce": "^1.1.2",
  "@tinymce/tinymce-vue": "^2.1.0",
  "prismjs": "^1.28.0",
  "tinymce": "^5.3.2",
  "moment": "^2.29.1",
```



### 路由迁移

nuxtjs项目的路由是由存放在pages文件夹下的vue文件自动生成。由于原系统对路由守卫，子路由等相关已设置好。安装官方插件用于实现原来的路由方式

```bash
npm install --save @nuxtjs/router
```

安装完成后在配置文件中引入

```javascript
// nuxt.config.js
export default {
    srcDir: 'src/',
	modules: [
   '@nuxtjs/router',
	]  
}
```

该插件识别项目根目录router.js为路由同时需要导出为createRouter函数。

将原route文件下index.js修改下

```javascript
export function createRouter() {
   let router = new VueRouter({
      routes,
      mode: 'history',
      base: process.env.BASE_URL,
      scrollBehavior(to, from, savedPosition) {
         if (savedPosition) {
            return savedPosition
         } else {
            return { top: 0 }
         }
      },
   })
   router.beforeEach((to, from, next) => {
   		//路由守卫代码
     })
   return router
}
```

需要移除路由的异步加载。否则运行会出现

**render function or template not defined in component: anonymous**

让人摸不着头脑的错误，经查询是路由异步加载的原因。

才想起原来为了提供首屏加速使用了异步加载和分片。

```javascript
//before
const home = () => import(/* webpackChunkName: "home" */ '../views/home');
const article = () => import(/* webpackChunkName: "home" */ '../components/post.vue')
const postlist = () => import(/* webpackChunkName: "home" */ '../views/postlist.vue')
const timelines = () => import(/* webpackChunkName: "home" */  '../views/timelines.vue')
const archive = () => import(/* webpackChunkName: "home" */  '../views/archive.vue')
const about = () => import(/* webpackChunkName: "home" */ '../views/about.vue')
const game = () => import(/* webpackChunkName: "home" */ '../views/game.vue')
//after
import home from '../views/home'
import article from '../components/post.vue'
import postlist from '../views/postlist.vue'
import timelines from '../views/timelines.vue'
import archive from '../views/archive.vue'
import about from '../views/about.vue'
import game from '../views/game.vue'
```

### mian.js

nuxtjs的入口文件不再是main.js，所以原来安装vue插件等操作需要将其存放在plugins文件夹中，作为nuxtjs的插件使用

```javascript
import Vue from 'vue'
import Prism from 'prismjs';
import randomImg from './randomImg.js'
import Moment from 'moment'
import 'moment/locale/zh-cn'
import VueTinymce from "@packy-tang/vue-tinymce";

Vue.use(Prism)
if (process.client) {
   Vue.prototype.$tinymce = tinymce; // 将全局tinymce对象指向给Vue作用域下
   Vue.use(VueTinymce); // 安装vue的tinymce组件
   Prism.highlightAll();
}
Vue.prototype.$randomImg = randomImg
Vue.prototype.$Moment = Moment
```

**注意**

由于服务端js没有window和document，当使用到的库或者插件中有使用到window或者document会报错。

使用process.client来判断当前环境，让这些操作仅在客户端执行。

配置使用插件，这样mainjs文件中的操作便会被执行

```
// nuxt.config.js
export default {
    srcDir: 'src/',
	modules: ['@nuxtjs/router',], 
	plugins: ['@/plugins/main'],
  }
```

### App.vue

原项目App.vue文件中包括了为路由视图展示的容器，也包含了自己封装snackbar插件的引入，而且vuefity的抽屉和工具栏等许多组件只有在<v-app>中才能正常工作。

因此将该文件改名default.vue并复制到layouts文件夹中

并且由于vue-axios会在该项目中报未知的错误，所以移除。导致原项目this.$http失效。重新安装nuxt-axios后仅能使用this.$axios。为避免重复的修改在app.vue文件中添加

```
  created() {
    Vue.prototype.$http = this.$axios;
  },
```

### 跨域代理

复制原项目proxy配置同时在axios配置代理

```javascript
 // nuxt.config.js
export default {
    srcDir: 'src/',
	modules: ['@nuxtjs/router','@nuxtjs/axios'], 
	plugins: ['@/plugins/main'],
    axios: {
      // baseURL: 'http://senkaryouran.top',
      proxy: true, // 表示开启代理
      prefix: '/', // 表示给请求url加个前缀 /api
      credentials: true // 表示跨域请求时是否需要使用凭证
   },
	proxy: {
      '/api': {
         target: 'http://senkaryouran.top:1145', // 目标接口域名
         // target: 'http://192.168.0.168:9001', // 本地
         changeOrigin: true, // 表示是否跨域
         pathRewrite: {
            '^/api': '/api', // 把 /api 替换成 /
         }
      }
   },
}
```

### 原项目代码适应修改

运行dev

![image-20220813031505886](C:\Users\Sakura\AppData\Roaming\Typora\typora-user-images\image-20220813031505886.png)

项目到这里运行已经基本能够渲染，但是有部分document和window的方法需要修改

由于服务端渲染只涉及到beforeCreated和created，因此涉及上面这两样的可以放在mouted阶段留给客户端渲染。实在需要在created执行的需要用process.client判断。比如一下绑定在组件上的事件

### 引用css，script的外部链接

原来vuecli项目是有一个index.html的模板文件用来处理这种需求的。nuxt中取而代之的是在配置文件中head设置

```javascript
 // nuxt.config.js
export default {
    .....
head: {
      titleTemplate: '%s - myblog-nuxt',
      title: 'myblog-nuxt',
      htmlAttrs: {
         lang: 'en'
      },
      meta: [
         { charset: 'utf-8' },
         { name: 'viewport', content: 'width=device-width, initial-scale=1' },
         { hid: 'description', name: 'description', content: '' },
         { name: 'format-detection', content: 'telephone=no' }
      ],
      link: [
         { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' },
         { rel: 'stylesheet', href: "https://fonts.googleapis.com/css?family=Roboto:100,300,400,500,700,900" },
         { rel: 'stylesheet', href: "https://fonts.googleapis.com/css?family=Noto+SerifMerriweather|Merriweather+Sans|Source+Code+Pro|Ubuntu:400,700|Noto+Serif+SC" }
      ],
      script: [{
         src: '/tinymce/tinymce.min.js'
      }]
   },
        .....
}
```

将原项目中需要引入的字体和tinymce编辑器相关的文件在此引入。

### asyncData

> asyncData方法会在组件（限于页面组件）每次加载之前被调用。它可以在服务端或路由更新之前被调用。在这个方法被调用的时候，第一个参数被设定为当前页面的上下文对象，你可以利用 asyncData方法来获取数据并返回给当前组件。

由于在created中异步获取的数据也不会在渲染返回的html中，渲染的只有赋有初值的data。不利于seo。所以使用asyncData方法才能实现我们的目的

博客类异步获取的数据要被爬取的有文章列表的和文章的详细内容。因此修改这两个方法为asyncData

```javascript
//  postlist.vue
  async asyncData({ app }) {
    let res = await app.$axios.get("/api/posts/");
    if ((res.data.code = 10000)) return { postList: res.data.data };
  }
  
//  post.vue
  async asyncData({ app, route, params }) {
    return app.$axios
      .get(`/api/posts/${params.id}`)
      .then((res) => {
        if (res.data.code == 10000) return { post: res.data.data };
        else return Promise.reject("文章不存在");
      })
      .catch((err) => {
        return { emptyPost: true };
        //console.log(app.router);
      });
  },
```

由于文章页面有查找到空文章就重定向到主页的需求，但是在asyncData方法中使用app.router.psuh()也无法重新定向，所以添加一个新的数据emptyPost用于检测文章是否为空，若为空在asyncData方法中返回的数据会覆盖原数据。再在mouted方法中判断是否重定向

```javascript
 //   post.vue
 mounted() {
    if (this.emptyPost) this.$router.push("./");
    .....
    }
```

### prismjs代码高亮

原项目prismjs代码高亮功能样式依赖于babel引入，

在配置文件中配置babel使用原来babel.config的配置

```javascript
export default {
 .....
 build: {
      babel: {
         "plugins": [
            ["prismjs", {
               "languages": ["java", "javascript", "css", "markup"],
               "plugins": ["line-numbers"],
               "theme": "tomorrow",
               "css": true
            }]
         ]
      }
   }, 
   .....
}
```

