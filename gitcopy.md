<h2>umi notes</h2>

<h3>目录结构：</h4>

```js
.
├── package.json // 插件和插件集
├── .umirc.ts //配置文件，包含umi内置功能和插件的配置
├── .env  //环境变量集
├── dist  //build
├── mock  
├── public //这个目录下的文件会被copy到输出路径
└── src
    ├── .umi //临时文件目录，webpack依赖图的入口文件，路由等临时生成的文件，不用提交此目录到git里面，类似于 npm run dev 时webpack的虚拟build的dist文件
    ├── layouts/index.tsx //约定式路由时的全局布局文件
    ├── pages //路由组件存放地
        ├── index.less
        └── index.tsx
    └── app.ts //运行时配置文件
```

<h3>配置式路由</h3>

umi还是基于spa单页面应用的存在，页面跳转还是在浏览器端完成，不会重新请求服务端获取html，html只在应用初始化时加载一次。页面的切换还是组件的切换。只需要做到不同路由路径和对应组件关联即可。

<h4>配置路由</h4>

可以在配置文件中通过**routes**进行配置，格式为路由信息的数组。

```js
export default {
  routes: [
    { exact: true, path: '/', component: 'index' },
    { exact: true, path: '/user', component: 'user' },
  ],
}
```

##### path: `string`

##### component: `string`,配置 location 和 path 匹配后用于渲染的 React 组件路径。可以是绝对路径，也可以是相对路径，如果是相对路径，会从 `src/pages` 开始找起.

##### exact: `boolean`,Default: `true`  表示是否严格匹配，即 location 是否和 path 完全对应上。

##### routes配置子路由，通常在需要为多个路径增加 layout 组件时使用。

Eg:

```js
export default {
  routes: [
    { path: '/login', component: 'login' },
    {
      path: '/',
      component: '@/layouts/index',
      routes: [
        { path: '/list', component: 'list' },
        { path: '/admin', component: 'admin' },
      ],
    }, 
  ],
}
```

然后在 `src/layouts/index` 中通过 `props.children` 渲染子路由，

```jsx
export default (props) => {
  return <div style={{ padding: 20 }}>{ props.children }</div>;
}
```

这样，访问 `/list` 和 `/admin` 就会带上 `src/layouts/index` 这个 layout 组件。

##### redirectType: `string`配置路由跳转。

##### wrappersType: `string[]`配置路由的高阶组件封装。

Eg: 权限控制：

```js
export default {
  routes: [
    { path: '/user', component: 'user',
      wrappers: [
        '@/wrappers/auth',
      ],
    },
    { path: '/login', component: 'login' },
  ]
}
```

然后在 `src/wrappers/auth` 中，

```jsx
import { Redirect } from 'umi'
export default (props) => {
  const { isLogin } = useAuth();
  if (isLogin) {
    return <div>{ props.children }</div>;
  } else {
    return <Redirect to="/login" />;
  }
}
```

这样，访问 `/user`，就通过 `useAuth` 做权限校验，如果通过，渲染 `src/pages/user`，否则跳转到 `/login`，由 `src/pages/login` 进行渲染。

<h3>文件路由（约定式路由）</h3>

通过目录和文件及其命名分析出路由配置。

比如以下文件结构：

```bash
.
  └── pages
    ├── index.tsx
    └── users.tsx
```

会得到以下路由配置，

```js
[
  { exact: true, path: '/', component: '@/pages/index' },
  { exact: true, path: '/users', component: '@/pages/users' },
]
```

需要注意的是，满足以下任意规则的文件不会被注册为路由，

- 以 `.` 或 `_` 开头的文件或目录
- 以 `d.ts` 结尾的类型定义文件
- 以 `test.ts`、`spec.ts`、`e2e.ts` 结尾的测试文件（适用于 `.js`、`.jsx` 和 `.tsx` 文件）
- `components` 和 `component` 目录
- `utils` 和 `util` 目录
- 不是 `.js`、`.jsx`、`.ts` 或 `.tsx` 文件
- 文件内容不包含 JSX 元素

### 动态路由

约定 `[]` 包裹的文件或文件夹为动态路由。

比如：

- `src/pages/users/[id].tsx` 会成为 `/users/:id`
- `src/pages/users/[id]/settings.tsx` 会成为 `/users/:id/settings`

### 动态可选路由

约定 `[ $]` 包裹的文件或文件夹为动态可选路由。

比如：

- `src/pages/users/[id$].tsx` 会成为 `/users/:id?`
- `src/pages/users/[id$]/settings.tsx` 会成为 `/users/:id?/settings`

### 嵌套路由

Umi 里约定目录下有 `_layout.tsx` 时会生成嵌套路由，以 `_layout.tsx` 为该目录的 layout。layout 文件需要返回一个 React 组件，并通过 `props.children` 渲染子组件。

比如以下目录结构，

```bash
.
└── pages
    └── users
        ├── _layout.tsx
        ├── index.tsx
        └── list.tsx
```

会生成路由，

```js
[
  { exact: false, path: '/users', component: '@/pages/users/_layout',
    routes: [
      { exact: true, path: '/users', component: '@/pages/users/index' },
      { exact: true, path: '/users/list', component: '@/pages/users/list' },
    ]
  }
]
```

<h3>插件</h3>

每个插件都会对应一个 id 和一个 key，**id 是路径的简写**，**key 是进一步简化后用于配置的唯一值**。

比如插件 `/node_modules/@umijs/plugin-foo/index.js`，通常来说，其 id 为 `@umijs/plugin-foo`，key 为 `foo`

请不要配置 npm 包的插件，否则会报重复注册的错误.因为Umi 会自动检测 `dependencies` 和 `devDependencies` 里的 umi 插件。

<h3>页面跳转</h3>

声明式通过link，通常作为react组件使用

命令式通过history模式使用，通常在事件处理中被调用，也可以直接从组件的属性中取得history。

<h3>Mock</h3>

Umi 约定 `/mock` 文件夹下所有文件为 mock 文件。

```js
export default {
  // 支持值为 Object 和 Array
  'GET /api/users': { users: [1, 2] },

  // GET 可忽略
  '/api/users/1': { id: 1 },

  // 支持自定义函数，API 参考 express@4
  'POST /api/users/create': (req, res) => {
    // 添加跨域请求头
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.end('ok');
  },
}
```

[Mock.js](http://mockjs.com/) 是常用的辅助生成模拟数据的三方库，借助他可以提升我们的 mock 数据能力。

比如：

```js
import mockjs from 'mockjs';


export default {
  // 使用 mockjs 等三方库
  'GET /api/tags': mockjs.mock({
    'list|100': [{ name: '@city', 'value|1-100': 50, 'type|0-2': 1 }],
  }),
};
```