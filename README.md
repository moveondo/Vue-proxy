# Vue-proxy


 
 
 
 1、首先axios不支持vue.use()方式声明使用，看了所有近乎相同的axios文档都没有提到这一点 
建议方式

在main.js中如下声明使用
```
import axios from 'axios';
Vue.prototype.$axios=axios;
```
那么在其他vue组件中就可以this.$axios调用使用


2.小小的提一下vue cli脚手架前端调后端数据接口时候的本地代理跨域问题，如我在本地localhost访问接口http://40.00.100.100:3002/是要跨域的，相当于浏览器设置了一到门槛，会报错XMLHTTPRequest can not load http://40.00.100.100:3002/. Response to preflight request doesn’t pass access control…. 
为什么跨域同源非同源自己去查吧，在webpack配置一下proxyTable就OK了，如下 
config/index.js
```
dev: {
    加入以下
    proxyTable: {
      '/api': {
        target: 'http://40.00.100.100:3002/',//设置你调用的接口域名和端口号 别忘了加http
        changeOrigin: true,
        pathRewrite: {
          '^/api': '/'//这里理解成用‘/api’代替target里面的地址，后面组件中我们掉接口时直接用api代替   
           比如我要调用'http://40.00.100.100:3002/user/add'，直接写‘/api/user/add’即可
        }
      }
    },
   
 ```
 

试一下，跨域成功了，但是注意了，这只是开发环境（dev）中解决了跨域问题，生产环境中真正部署到服务器上如果是非同源还是存在跨域问题，如我们部署的服务器端口是3001，需要前后端联调，第一步前端我们可以分生产production和开发development两种环境分别测试，在config/dev.env.js和prod.env.js里也就是开发/生产环境下分别配置一下请求的地址API_HOST，开发环境中我们用上面配置的代理地址api，生产环境下用正常的接口地址，所以这样配置

```
module.exports = merge(prodEnv, {
  NODE_ENV: '"development"',//开发环境
  API_HOST:"/api/"
})

module.exports = {
  NODE_ENV: '"production"',//生产环境
  API_HOST:'"http://40.00.100.100:3002/"'
}
```

当然不管是开发还是生产环境都可以直接请求http://40.00.100.100:3002/。配置好之后测试时程序会自动判断当前是开发还是生产环境，然后自动匹配API_HOST，我们在任何组件里都能用process.env.API_HOST来使用地址如

```
instance.post(process.env.API_HOST+'user/login', this.form)
```

然后第二步后端服务器配置一下cros跨域即可，就是access-control-allow-origin：*允许所有访问的意思。综上：开发的环境下我们前端可以自己配置个proxy代理就能跨域了，真正的生产环境下还需要后端的配合的。某大神说：此方法ie9及以下不好使，如果需要兼容，最好的办法是后端在服务器端口加个代理，效果类似开发时webpack的代理。

3、axios发送get post请求问题 
发送post请求时一般都要设置Content-Type，发送内容的类型，application/json指发送json对象但是要提前stringify一下。application/xxxx-form指发送？a=b&c=d格式，可以用qs的方法格式化一下，qs在安装axios后会自动安装，只需要组件里import一下即可。

```
const postData=JSON.stringify(this.formCustomer);
'Content-Type':'application/json'}

const postData=Qs.stringify(this.formCustomer);//过滤成？&=格式
'Content-Type':'application/xxxx-form'}
```

4.axios拦截器的使用 
当我们访问某个地址页面时，有时会要求我们重新登录后再访问该页面，也就是身份认证失效了，如token丢失了，或者是token依然存在本地，但是却失效了，所以单单判断本地有没有token值不能解决问题。此时请求时服务器返回的是401错误，授权出错，也就是没有权利访问该页面。 
我们可以在发送所有请求之前和操作服务器响应数据之前对这种情况过滤。

// http request 请求拦截器，有token值则配置上token值
```
axios.interceptors.request.use(
    config => {
        if (token) {  // 每次发送请求之前判断是否存在token，如果存在，则统一在http请求的header都加上token，不用每次请求都手动添加了
            config.headers.Authorization = token;
        }
        return config;
    },
    err => {
        return Promise.reject(err);
    });
  
```

http response 服务器响应拦截器，这里拦截401错误，并重新跳入登页重新获取token

```
axios.interceptors.response.use(
    response => {
        return response;
    },
    error => {
        if (error.response) {
            switch (error.response.status) {
                case 401:
                    // 这里写清除token的代码
                    router.replace({
                        path: 'login',
                        query: {redirect: router.currentRoute.fullPath}//登录成功后跳入浏览的当前页面
                    })
            }
        }
        return Promise.reject(error.response.data) 
    });
   
   ```

参考链接：
 http://blog.csdn.net/u012369271/article/details/72848102
