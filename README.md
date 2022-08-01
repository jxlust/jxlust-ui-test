## 前言

- 如何使用 pnpm 搭建出一个 Monorepo 环境
- 如何使用 vite 搭建一个基本的 Vue3 脚手架项目
- 如何开发调试一个自己的 UI 组件库
- 如何使用 vite 打包并发布自己的 UI 组件库

## Monorepo 环境

就是指在一个大的项目仓库中，管理多个模块/包（package），这种类型的项目大都在项目根目录下有一个 packages 文件夹，分多个项目管理。
简单来说就是单仓库 多项目

## 搭建

### pnpm

- .npmrc -> shamefully-hoist = true
  如果某些工具仅在根目录的 node_modules 时才有效，可以将其设置为 true 来提升那些不在根目录的 node_modules，就是将你安装的依赖包的依赖包的依赖包的...都放到同一级别（扁平化）。说白了就是不设置为 true 有些包就有可能会出问题。

- 新建 pnpm-workspace.yaml 这样就能将我们项目下的 packages 目录和 examples 目录关联起来了

- 安装依赖

```shell
pnpm i vue@next typescript sass -D -w
```

```shell
#tsconfig.json
npx tsc --init
```

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "jsx": "preserve",
    "strict": true,
    "target": "ES2015",
    "module": "ESNext",
    "skipLibCheck": true,
    "esModuleInterop": true,
    "moduleResolution": "Node",
    "lib": ["esnext", "dom"]
  }
}
```

### examples 搭建

- vue3+vite 初始化仓库

```shell
#进入examples文件夹，执行
pnpm init

#安装vite和@vitejs/plugin-vue
#@vitejs/plugin-vue用来支持.vue文件的转译
#依赖都安装到根目录下
pnpm install vite @vitejs/plugin-vue -D -w

```

- 配置 vite.config.ts

```js
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";

export default defineConfig({
  plugins: [vue()],
});
```

### packages 搭建

- 新建 packages
- utils 包，新建 utils
  进入 utils 文件夹执行：pnpm init 然后会生成一个 package.json 文件；这里需要改一下包名，我这里将 name 改成@jxlust-ui/utils 表示这个 utils 包是属于 jxlust-ui 这个组织下的。所以记住发布之前要登录 npm 新建一个组织；例如 jxlust-ui

```json
{
  "name": "@jxlust-ui/utils",
  "version": "1.0.0",
  "description": "",
  "main": "index.ts",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

- 组件库包(这里命名为 jxlust-ui)，新建 components
  同样初始化 pk

```json
{
  "name": "jxlust-ui",
  "version": "1.0.0",
  "description": "",
  "main": "index.ts",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

- 由于组件库是基于 ts 的，所以需要安装 esno 来执行 ts 文件便于测试组件之间的引入情况

```shell
npm i esno -g

```

### 包之间

- 进入 components 文件夹执行

```shell
pnpm install @jxlust-ui/utils

#"@jxlust-ui/utils": "workspace:^1.0.0"
```

pnpm 会自动创建个软链接直接指向我们的 utils 包

### 开发一个 Button 组件

- 目录

```
-- components
  -- src
    -- button
    -- icon
    -- index.ts
-- package.json
```

```js
//src/index.ts
import Button from "./button.vue";
export default Button;
//统一组件出口文件 index.ts
import Button from "./button";
export { Button };
```

### examples 使用组件

- 同样，直接在 examples 执行：pnpm i jxlust-ui，建个软链接直接指向我们的 xxx 包。

### components button 组件开发

- button 下新建 types.ts

```js
import { ExtractPropTypes } from "vue";

export const ButtonType = [
  "default",
  "primary",
  "success",
  "warning",
  "danger",
];

export const ButtonSize = ["large", "normal", "small", "mini"];

export const buttonProps = {
  type: {
    type: String,
    values: ButtonType,
  },
  size: {
    type: String,
    values: ButtonSize,
  },
};
export type ButtonProps = ExtractPropTypes<typeof buttonProps>;
```

- index.ts

```js
import button from './button.vue'
import type {App,Plugin} from "vue"
type SFCWithInstall<T> = T&Plugin
const withInstall = <T>(comp:T) => {
    (comp as SFCWithInstall<T>).install = (app:App)=>{
        //注册组件
        app.component((comp as any).name,comp)
    }
    return comp as SFCWithInstall<T>
}
const Button = withInstall(button)
export default Button
```

### 组件 vite 打包

- components 下新建 vite.config.ts

```js
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";
export default defineConfig({
  build: {
    target: "modules",
    //打包文件目录
    outDir: "es",
    //压缩
    minify: false,
    //css分离
    //cssCodeSplit: true,
    rollupOptions: {
      //忽略打包vue文件
      external: ["vue"],
      input: ["src/index.ts"],
      output: [
        {
          format: "es",
          //不用打包成.es.js,这里我们想把它打包成.js
          entryFileNames: "[name].js",
          //让打包目录和我们目录对应
          preserveModules: true,
          //配置打包根目录
          dir: "es",
          preserveModulesRoot: "src",
        },
        {
          format: "cjs",
          entryFileNames: "[name].js",
          //让打包目录和我们目录对应
          preserveModules: true,
          //配置打包根目录
          dir: "lib",
          preserveModulesRoot: "src",
        },
      ],
    },
    lib: {
      entry: "./index.ts",
      formats: ["es", "cjs"],
    },
  },
  plugins: [vue()],
});
```

- 那么如何向打包后的库里加入声明文件呢？其实很简单，只需要引入 vite-plugin-dts

```shell
#跟目录
pnpm i vite-plugin-dts -D -w
```

修改一下 vite.config.ts

```js
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue"
import dts from 'vite-plugin-dts'

export default defineConfig(
    {
        build: {...},
        plugins: [
            vue(),
            dts({
                //指定使用的tsconfig.json为我们整个项目根目录下掉,如果不配置,你也可以在components下新建tsconfig.json
                tsConfigFilePath: '../../tsconfig.json'
            }),
            //因为这个插件默认打包到es下，我们想让lib目录下也生成声明文件需要再配置一个
            dts({
                outputDir:'lib',
                tsConfigFilePath: '../../tsconfig.json'
            })

        ]
    }
)
```

- use 如果报模块错误

```js
//添加声明
declare module "jxlust-ui" {
  const content: { Button: any };
  export = content;
}

```

> 参考链接：
> https://mp.weixin.qq.com/s/RObdL09oDw-g50utLFTymA > https://gitee.com/geeksdidi/kittyui/tree/master/
