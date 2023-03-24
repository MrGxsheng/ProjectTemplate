### 代码

```js
export async function sleep(ms){
  return new Promise(resolve => setTimeout(resolve,ms));
}

```

### 引用

别傻了吧唧的直接复制 改改url

```js
import {sleep} from "../Common.js";
```

