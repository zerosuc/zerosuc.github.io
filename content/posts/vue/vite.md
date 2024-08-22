---
title: "vue常用套路"
date: 2022-05-20T10:20:10-08:00
draft: false
categories: ["vue"]
tags: ["vue"]
---

### vite vue 文件配置

```js
// https://vitejs.dev/config/
export default defineConfig(({ command, mode }) => {
  // 根据当前工作目录中的 `mode` 加载 .env 文件
  // 设置第三个参数为 '' 来加载所有环境变量，而不管是否有 `VITE_` 前缀。
  const env = loadEnv(mode, process.cwd(), "");

  return {
    // vite 配置
    plugins: [
      vue(),
      AutoImport({
        imports: ["vue", "vue-router", "pinia", "vue-i18n"],
        resolvers: [ElementPlusResolver()],
      }),
      Components({
        resolvers: [ElementPlusResolver()],
      }),
    ],
    server: {
      // 如果使用docker-compose开发模式，设置为false
      open: true,
      port: 8081,
      proxy: {
        // 把key的路径代理到target位置
        [env.VITE_BASE_API]: {
          // 需要代理的路径   例如 '/api'
          target: `${env.VITE_BASE_PATH}/`, // 代理到 目标路径
          changeOrigin: true,
          //rewrite: path => path.replace(new RegExp('^' + env.VITE_BASE_API), ''),
        },
      },
    },
    resolve: {
      alias: {
        "@": fileURLToPath(new URL("./src", import.meta.url)),
      },
    },
  };
});
```

### vue 通识

```vue
<template>
  <span>我是App.vue</span>
  <!-- 路由占位符，会导入匹配到的$route.path的视图组件 -->
  <router-view></router-view>
</template>

<style>
/*设置html和body 日常置位*/
html,
body {
  width: 100%;
  height: 100%;
  padding: 0;
  margin: 0;
}
#nprogress .bar {
  /*自定义进度条颜色*/
  background: #2186c0 !important;
}
</style>
```

### 分栏

```vue
<template>
  <div class="deploy">
    <el-row>
      <!-- 头部1 -->
      <el-col :span="24"></el-col>
      <!-- 头部2 -->
      <el-col :span="24"></el-col>
      <!-- 数据表格 -->
      <el-col :span="24"></el-col>
    </el-row>
  </div>
</template>
```

### 请求拦截

```js
import { useUserStore } from "@/stores/user.js";
// 6: 然后创建一个axios的实例
const request = axios.create({ ...config });

// request request请求拦截器
request.interceptors.request.use(
  function (config) {
    // 这个判断就是为了那些不需要token接口做开关处理，比如：登录，检测等
    if (!config.noToken) {
      // 如果 token为空，说明没有登录。你就要去登录了
      const userStore = useUserStore();
      const isLogin = userStore.isLogin;
      if (!isLogin) {
        router.push("/login");
        return;
      } else {
        // 这里给请求头增加参数.request--header，在服务端可以通过request的header可以获取到对应参数
        // 比如go: c.GetHeader("Authorization")
        // 比如java: request.getHeader("Authorization")
        config.headers.Authorization = userStore.getToken();
      }
    }
    return config;
  },
  function (error) {
    // 判断请求超时
    if (
      error.code === "ECONNABORTED" &&
      error.message.indexOf("timeout") !== -1
    ) {
      ElMessage({ message: "请求超时", type: "error", showClose: true });
      // 这里为啥不写return
    }
    return Promise.reject(error);
  }
);

// request response 响应拦截器
request.interceptors.response.use(
  (response) => {
    // 在这里应该可以获取服务端传递过来头部信息
    // 开始续期
    if (response.headers["new-authorization"]) {
      const userStore = useUserStore();
      userStore.setToken(response.headers["new-authorization"]);
    }

    if (response.data.code === 20000) {
      return response.data;
    } else {
      if (response.data.message) {
        TIPS.error(response.data.message);
      }
      // 返回接口返回的错误信息
      return Promise.reject(response.data);
    }
  },
  (err) => {
    if (err && err.response) {
      switch (err.response.status) {
        case 400:
          err.message = "请求错误";
          break;
        case 401:
          err.message = "未授权，请登录";
          break;
        case 403:
          err.message = "拒绝访问";
          break;
        case 404:
          err.message = `请求地址出错: ${err.response.config.url}`;
          break;
        case 408:
          err.message = "请求超时";
          break;
        case 500:
          err.message = "服务器内部错误";
          break;
        case 501:
          err.message = "服务未实现";
          break;
        case 502:
          err.message = "网关错误";
          break;
        case 503:
          err.message = "服务不可用";
          break;
        case 504:
          err.message = "网关超时";
          break;
        case 505:
          err.message = "HTTP版本不受支持";
          break;
        default:
      }
    }
    if (err.message) {
      TIPS.error(err.message);
    }
    // 判断请求超时
    if (err.code === "ECONNABORTED" && err.message.indexOf("timeout") !== -1) {
      TIPS.error("服务已经离开地球表面，刷新或者重试...");
    }
    // 返回接口返回的错误信息
    return Promise.reject(err);
  }
);

export default request;
```

### 静态路由

```js
const router = createRouter({
  history: createWebHashHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: "/",
      name: "Home",
      component: Layout,
    },
    {
      path: "/login",
      name: "Login",
      meta: { title: "login" },
      component: () => import("@/views/Login.vue"),
    },
  ],
});

const router404 = {
  path: "/:pathMatch(.*)*",
  name: "NotFound",
  component: () => import("@/views/error/NotFound.vue"),
};
```

### api 调用封装

```js
import service from "@/utils/request";
// @Summary 用户登录 获取动态路由
// @Produce  application/json
// @Param 可以什么都不填 调一下即可
// @Router /menu/getMenu [post]
export const asyncMenu = () => {
  return service({
    url: "/menu/getMenu",
    method: "post",
  });
};
```

### 自定义指令

```js
export default {
  install(app) {
    // app 为根组件实例
    app.directive("permission", {
      mounted(el, binding) {
        hasPermission(binding.value, el);
      },
    });
  },
};

function hasPermission(value, el = false) {
  //判断传入的数据是否符合自己设定的预期
  if (!Array.isArray(value)) {
    //如果传入的不是数组则抛出错误
    throw new Error(`需要传入正确的数据类型`);
  }
  //判断是否拥有权限
  const userStore = useUserStore();
  let data = userStore.permissionCode; //存储着是否拥有该请求权限的数组
  //如果在data中包含了传入的数据字段，则代表拥有权限，没有则返回 -1  此时如果为false 则代表没有权限
  let hasAuth = value.findIndex((v) => data.includes(v)) != -1;
  //当使用的自定义指令dom存在时，且没有权限
  if (el && !hasAuth) {
    //移除组件
    el.parentNode.removeChild(el);
  }
}
```

### 全局注册组件或者插件方式注册自定义组件

```js
for (const [key, component] of Object.entries(ElementPlusIconsVue)) {
  app.component(key, component);
}
或者;
export default {
  install(app) {
    // 全自动化过程注册全局组件，就不需要在引入在注册
    // 把src/components目录下的以.vue结尾的文件全部匹配出来。包括子孙目录下的.vue结尾的文件
    const modules = import.meta.glob("../components/**/*.vue");
    for (let key in modules) {
      var componentName = key.substring(
        key.lastIndexOf("/") + 1,
        key.lastIndexOf(".")
      );
      app.component(componentName, modules[key]);
    }
  },
};
```
