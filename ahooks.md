# useRequest

## 快速开始

`useRequest`是一个强大的管理异步数据的 `hook`，可以满足 `React`项目中的各种网络请求的场景。

`useRequest`通过插入式的方式管理代码，核心代码极其的简单，可以很简单的被继承并开发更多的先进的功能。当前的功能包括：

* 自动/手动 请求
* 轮询
* 防抖
* 节流
* 窗口聚焦刷新
* 错误重试
* 延迟加载
* 过期重新生效
* 缓存

接下来，我们通过两个简单的例子来了解 `useRequest`

### 默认用法

`useRequest`的第一个参数是一个异步的函数，当组件第一次加载的时候会自动执行。同时，它会自动管理这个异步函数的 `loading`、`data`
`error`状态。

```typescript
const { data, error, loading} = useRequest(getUsername)
```

```typescript

import { useRequest } from 'ahooks';
import Mock from 'mockjs';
import React from 'react';

function getUsername(): Promise<string> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(Mock.mock('@name'));
    }, 1000);
  });
}

export default () => {
  const { data, error, loading } = useRequest(getUsername);

  if (error) {
    return <div>failed to load</div>;
  }
  if (loading) {
    return <div>loading...</div>;
  }
  return <div>Username: {data}</div>;
};

```

### 手动触发

如果设置了 `options.manual = true`，`useRequest`不会默认执行，需要通过 `run`手动触发执行。

```typescript
import { message } from 'antd';
import React, { useState } from 'react';
import { useRequest } from 'ahooks';

// eslint-disable-next-line @typescript-eslint/no-unused-vars
function changeUsername(username: string): Promise<{ success: boolean }> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve({ success: true });
    }, 1000);
  });
}

export default () => {
  const [state, setState] = useState('');

  const { loading, run } = useRequest(changeUsername, {
    manual: true,
    onSuccess: (result, params) => {
      if (result.success) {
        setState('');
        message.success(`The username was changed to "${params[0]}" !`);
      }
    },
  });

  return (
    <div>
      <input
        onChange={(e) => setState(e.target.value)}
        value={state}
        placeholder="Please enter username"
        style={{ width: 240, marginRight: 16 }}
      />
      <button disabled={loading} type="button" onClick={() => run(state)}>
        {loading ? 'Loading' : 'Edit'}
      </button>
    </div>
  );
};

```

在上面两个例子里面，我们展示了 `useRequest`最基本的用法，接下来，我们将会一个接一个的介绍 `useRequest`的特性。

## 基础用法

在这节里面，我们将会介绍 `useRequest`的核心和基础的功能。也就是说，`useRequest`的核心功能

### 默认请求

默认情况下，`useRequest`的第一个参数是一个一步的函数，当组件初始化之后将会自动执行。同时，它将会自动管理这个异步函数的 `loading`, `error`, `data` 的状态。

```typescript
import { useRequest } from 'ahooks';
import Mock from 'mockjs';
import React from 'react';

function getUsername(): Promise<string> {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (Math.random() > 0.5) {
        resolve(Mock.mock('@name'));
      } else {
        reject(new Error('Failed to get username'));
      }
    }, 1000);
  });
}

export default () => {
  const { data, error, loading } = useRequest(getUsername);

  if (error) {
    return <div>{error.message}</div>;
  }
  if (loading) {
    return <div>loading...</div>;
  }
  return <div>Username: {data}</div>;
};
```

### 手动触发

如果设置了 `options.manual === true`, `useRequest`将不会默认的执行，它的执行需要手动通过 `run`或者 `runAsync`触发

```typescript
const { loading, run, runAsync } = useRequest(service, {
  manual: true
});

<button onClick={run} disabled={loading}>
  {loading ? 'Loading' : 'Edit'}
</button>
```

`run` 和 `runAsync`的不同点是：

* `run` 是一个正常的异步函数，我们将会捕获执行，你可以使用 `options.onError`来处理执行的异常。
* `runAsync` 是一个返回一个 `Promise`的异步函数。如果你使用 `runAsync`去调用，也就意味着你需要自己捕获异常。

```typescript
runAsync().then((data) => {
  console.log(data);
}).catch((error) => {
  console.log(error);
})
```

接下来，我们将会通过一个简单的场景演示 `run` 和 `runAsync`的不同点。

```typescript
import { message } from 'antd';
import React, { useState } from 'react';
import { useRequest } from 'ahooks';

function editUsername(username: string): Promise<void> {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (Math.random() > 0.5) {
        resolve();
      } else {
        reject(new Error('Failed to modify username'));
      }
    }, 1000);
  });
}

export default () => {
  const [state, setState] = useState('');

  const { loading, run } = useRequest(editUsername, {
    manual: true,
    onSuccess: (result, params) => {
      setState('');
      message.success(`The username was changed to "${params[0]}" !`);
    },
    onError: (error) => {
      message.error(error.message);
    },
  });

  return (
    <div>
      <input
        onChange={(e) => setState(e.target.value)}
        value={state}
        placeholder="Please enter username"
        style={{ width: 240, marginRight: 16 }}
      />
      <button disabled={loading} type="button" onClick={() => run(state)}>
        {loading ? 'Loading' : 'Edit'}
      </button>
    </div>
  );
};
```

### 生命周期

`useRequest` 在异步函数执行的不同阶段，提供了下面几个生命周期给你。

* `onBefore`: 请求之前触发。
* `onSuccess`: 当请求完成的时候触发。
* `onError`: 当请求失败之后触发。
* `onFinally`: 当请求完成之后触发。

```typescript
import { message } from 'antd';
import React, { useState } from 'react';
import { useRequest } from 'ahooks';

function editUsername(username: string): Promise<void> {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (Math.random() > 0.5) {
        resolve();
      } else {
        reject(new Error('Failed to modify username'));
      }
    }, 1000);
  });
}

export default () => {
  const [state, setState] = useState('');

  const { loading, run } = useRequest(editUsername, {
    manual: true,
    onBefore: (params) => {
      message.info(`Start Request: ${params[0]}`);
    },
    onSuccess: (result, params) => {
      setState('');
      message.success(`The username was changed to "${params[0]}" !`);
    },
    onError: (error) => {
      message.error(error.message);
    },
    onFinally: (params, result, error) => {
      message.info(`Request finish`);
    },
  });

  return (
    <div>
      <input
        onChange={(e) => setState(e.target.value)}
        value={state}
        placeholder="Please enter username"
        style={{ width: 240, marginRight: 16 }}
      />
      <button disabled={loading} type="button" onClick={() => run(state)}>
        {loading ? 'Loading' : 'Edit'}
      </button>
    </div>
  );
};
```

### 刷新（重复最后一次的请求）

`useRequest` 提供 `refresh` 和 `refreshAsync` 方法，所以我们可以使用最后的参数去重新执行请求。
如果是在读取用户信息的场景

1. 我们通过 `run(1)`，读取ID为1的用户的信息。
2. 我们通过一些方式更新了用户信息
3. 我们希望重新初始化最后一个的请求，我们可以使用 `refresh`来替代 `run(1)`，这对复杂参数的场景特别的有用。

```typescript
import { useRequest } from 'ahooks';
import Mock from 'mockjs';
import React, { useEffect } from 'react';

function getUsername(id: number): Promise<string> {
  console.log('use-request-refresh-id', id);
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(Mock.mock('@name'));
    }, 1000);
  });
}

export default () => {
  const { data, loading, run, refresh } = useRequest((id: number) => getUsername(id), {
    manual: true,
  });

  useEffect(() => {
    run(1);
  }, []);

  if (loading) {
    return <div>loading...</div>;
  }
  return (
    <div>
      <p>Username: {data}</p>
      <button onClick={refresh} type="button">
        Refresh
      </button>
    </div>
  );
};
```

当然，`refresh`和 `refreshAsync`的不同点与 `run`和 `runAsync`一样。

### 立即改变数据

`useRequest`提供了 `mutate`, 可以立即修改数据。
`mutate`是和 `React.setState`一致的。支持：`mutate(newState)` 和 `mutate((old) => newDate)`。

在接下来的例子中，我们演示了一个 `mutate`的用法。

我门已经修改了用户的姓名，但是我们不想等到接口调用成功后再给用户反馈。而是直接修改数据，然后在后台里面调用请求，在请求返回以后提供额外的反馈

```typescript
import { message } from 'antd';
import React, { useState, useRef } from 'react';
import { useRequest } from 'ahooks';
import Mock from 'mockjs';

function getUsername(): Promise<string> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(Mock.mock('@name'));
    }, 1000);
  });
}

function editUsername(username: string): Promise<void> {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (Math.random() > 0.5) {
        resolve();
      } else {
        reject(new Error('Failed to modify username'));
      }
    }, 1000);
  });
}

export default () => {
  // store last username
  const lastRef = useRef<string>();

  const [state, setState] = useState('');

  // get username
  const { data: username, mutate } = useRequest(getUsername);

  // edit username
  const { run: edit } = useRequest(editUsername, {
    manual: true,
    onSuccess: (result, params) => {
      setState('');
      message.success(`The username was changed to "${params[0]}" !`);
    },
    onError: (error) => {
      message.error(error.message);
      mutate(lastRef.current);
    },
  });

  const onChange = () => {
    lastRef.current = username;
    mutate(state);
    edit(state);
  };

  return (
    <div>
      <p>Username: {username}</p>
      <input
        onChange={(e) => setState(e.target.value)}
        value={state}
        placeholder="Please enter username"
        style={{ width: 240, marginRight: 16 }}
      />
      <button type="button" onClick={onChange}>
        Edit
      </button>
    </div>
  );
};

```

### 取消响应

`useRequest` 提供了 `cancel`函数，它将会忽略当前的 `promise`返回的数据和错误

提示：调用 `cancel` 不会取消 `promise`的执行

同时，`useRequest`将会自动忽略下面一些情况的响应：

* 组件正在卸载，正在执行 `promise`
* 静态取消，当先前的 `promise` 还没有返回，如果下一次的请求已经准备好，那么先前的 `promise` 将会被忽略

```typescript
import { message } from 'antd';
import React, { useState } from 'react';
import { useRequest } from 'ahooks';

function editUsername(username: string): Promise<void> {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (Math.random() > 0.5) {
        resolve();
      } else {
        reject(new Error('Failed to modify username'));
      }
    }, 1000);
  });
}

export default () => {
  const [state, setState] = useState('');

  const { loading, run, cancel } = useRequest(editUsername, {
    manual: true,
    onSuccess: (result, params) => {
      setState('');
      message.success(`The username was changed to "${params[0]}" !`);
    },
    onError: (error) => {
      message.error(error.message);
    },
  });

  return (
    <div>
      <input
        onChange={(e) => setState(e.target.value)}
        value={state}
        placeholder="Please enter username"
        style={{ width: 240, marginRight: 16 }}
      />
      <button disabled={loading} type="button" onClick={() => run(state)}>
        {loading ? 'Loading' : 'Edit'}
      </button>
      <button type="button" onClick={cancel} style={{ marginLeft: 16 }}>
        Cancel
      </button>
    </div>
  );
};
```

### 参数管理

`useRequest`返回的参数将会记录 `service`的参数。例如，如果你触发 `run(1,2,3)`, 那么参数就是等于 `[1,2,3]`。
如果我们设置 `options.manual = false`, 首次调用 `service`的参数可以通过 `options.defaultParams`设置。

```typescript
import { useRequest } from 'ahooks';
import Mock from 'mockjs';
import React, { useState } from 'react';

function getUsername(id: string): Promise<string> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(Mock.mock('@name'));
    }, 1000);
  });
}

export default () => {
  const [state, setState] = useState('');

  // get username
  const {
    data: username,
    run,
    params,
  } = useRequest(getUsername, {
    defaultParams: ['1'],
  });

  const onChange = () => {
    run(state);
  };

  return (
    <div>
      <input
        onChange={(e) => setState(e.target.value)}
        value={state}
        placeholder="Please enter userId"
        style={{ width: 240, marginRight: 16 }}
      />
      <button type="button" onClick={onChange}>
        GetUserName
      </button>
      <p style={{ marginTop: 8 }}>UserId: {params[0]}</p>
      <p>Username: {username}</p>
    </div>
  );
};
```

### API

```typescript

const {
  loading: boolean,
  data?: TData,
  error?: Error,
  params: TParams || [],
  run: (...params: TParams) => void,
  runAsync: (...params: TParams) => Promise<TData>,
  refresh: () => void,
  refreshAsync: () => Promise<TData>,
  mutate: (data?: TData | ((oldData?: TData) => (TData | undefined))) => void,
  cancel: () => void,
} = useRequest<TData, TParams>(
  service: (...args: TParams) => Promise<TData>,
  {
    manual?: boolean,
    defaultParams?: TParams,
    onBefore?: (params: TParams) => void,
    onSuccess?: (data: TData, params: TParams) => void,
    onError?: (e: Error, params: TParams) => void,
    onFinally?: (params: TParams, data?: TData, e?: Error) => void,
  }
);

```

### Result

| 属性 | 描述                 | 类型 |
| ---- | -------------------- | ------ |
| data | service返回的数据  | TData|underfined |
| error| service抛出的异常 | Error｜underfined |
| loading| service是否已经在执行| boolean |
|params | service执行参数数组,例如，你执行`run(1,2,3)`, 那么参数就等于[1,2,3]  |TParams | [] |
|run | 1. 手动触发service的执行，参数将会传递给service  2. 异常自动处理，通过 onErrror 反馈| (...params: TParams) => void|
|runAsync |用法类似与run， 但是它返回一个Promise， 所以你可以自己处理异常 | (...params: TParams) => Promise< TData > |
|refresh | 使用最后一次的参数，执行run | () => void |
|refreshAsync| 使用最后一次的参数，执行runAsync | () => Promise< TData > |
|mutate |直接修改data | (data?: TData/((oldData?:TData) =>(TData/underfined))) => void |
|cancel | 忽略当前Promise响应|() => void |

### Options

|属性 |描述 |类型 | 默认值|
|-|-|-|-|
|manual | 默认值为false， 也就是说，service在初始化的过程中自动执行。如果设置了true，你需要手动的执行run 或者 runAsync手动触发执行 | boolean| false|
|defaultParams |第一次默认执行的时候传递给service的参数 |TParams | |
|onBefore |service执行前触发 |(params: TParams) => void | |
|onSucess |service resolve后触发 | | |
|onError |service reject的时候触发 | | |
|onFinally |service执行完成之后触发 | | |

上面我们介绍了useRequest最基础的功能，接下来我们将会介绍更多先进的功能

## 加载延迟

通过设置`options.loadingDelay`, 你可以延迟`loading`变成`true`的时间，有效的防止`UI`闪烁。

```typescript
const { loading, data } = useRequest(getUsername, {
   loadingDelay: 300
});

return <div>{ loading ? 'Loading...' : data }</div>
```

例如，在上面的场景里，如果`getUsername`在300ms内返回，`loading`不会变成`true`，避免了页面展示`loading`...

你可以快速的点击例子中的按钮去体验效果

```typescript
import { useRequest } from 'ahooks';
import React from 'react';
import Mock from 'mockjs';

function getUsername(): Promise<string> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(Mock.mock('@name'));
    }, 200);
  });
}

export default () => {
  const action = useRequest(getUsername);

  const withLoadingDelayAction = useRequest(getUsername, {
    loadingDelay: 300,
  });

  const trigger = () => {
    action.run();
    withLoadingDelayAction.run();
  };

  return (
    <div>
      <button type="button" onClick={trigger}>
        run
      </button>

      <div style={{ margin: '24px 0', width: 300 }}>
        Username: {action.loading ? 'Loading...' : action.data}
      </div>
      <div>
        Username: {withLoadingDelayAction.loading ? 'Loading...' : withLoadingDelayAction.data}
      </div>
    </div>
  );
};
```

### API

|属性|描述|类型|默认|
|-|-|-|-|
|loadingDelay |设置loading变成true的延迟时间 |number |0 |

### 备注

`options.loadingDelay`支持动态的变化

## 轮询
