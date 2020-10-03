## Jest难点进阶

### snapshot 快照测试

#### 快照的使用

```javascript
const generateConfig= () => ({
  host: 'localhost',
  port: 3000
})

test('test config snapshot', () => {
  expect(generateConfig()).toMatchSnapshot()
})

test('test another config snapshot', () => {
  expect(generateAnotherConfig()).toMatchSnapshot()
})
```

初次执行会生成快照文件，第二次执行同样生成快照文件，并与第一次快照进行比较，快照内容相同则测试通过。

快照比对失败，可以用 `u` 键批量更新快照，也可以用 `i` 键来决定每一个快照的更新操作。

如果 `generateConfig` 返回的配置信息在每次函数执行都会发生变化，难道需要我们每次都去更新快照，那么这种情况该如何处理呢？

比如

```javascript
const generateConfig= () => ({
  host: 'localhost',
  port: 3000，
  time: new Date()
})
```

这里的 `time` 字段在每次函数执行的时候都会更新，从而导致快照比对失败，那么能否在进行快照对比的时候自动忽略掉 `time` 字段的比对呢，其实是可以的

```js
test('test config snapshot', () => {
  expect(generateConfig()).toMatchSnapshot({
    time: expect.any(Date)
  })
})
```

#### 内联快照

内联快照-不会生成新的快照文件，而是将快照内容写入当前执行测试的文件中

其中在生成内联快照的时候需要第三方的包 `prettier` 配合

```
test('test config snapshot', () => {
  expect(generateConfig()).toMatchInlineSnapshot({
    time: expect.any(Date),
    // 内联快照会放在这里
  })
})
```

总结：快照测试常用于配置文件的测试、React、Vue组件等的测试

### mock深入学习

#### mock 接口请求

在 `Jest基础入门` 中实现过接口请求的模拟，如下：

```javascript
import axios from 'axios'
import { getData } from './api'

jest.mock('axios')

test.only('test axios getData', async () => {
  // mock 改变函数的内部实现
  axios.get.mockResolvedValue({ data: 'hello' })
  await getData().then((response) => {
    expect(response).toBe('hello')
  })
})
```

这种方法是通过改变 `axios` 内部方法实现来进行的模拟，这样所有的 `get` 请求都会被拦截到

另一种方式是对 `getData` 方法的 mock

```javascript
jest.mock('./api')
import { getData } from './api'

test.only('test axios getData', async () => {
  await getData().then((response) => {
    expect(response).toBe('hello')
  })
})
```

通过`mock('./api')` 这个文件，在下文导入  `getData` 这个方法的时候，不再从 `'./api'` 文件中导入，而是去 `__mocks__` 文件夹下对应的 `api` 文件中查找并导入，通过这种方式实现对 `getData` 方法的拦截处理

#### 取消mock

```javascript
jest.unmock('./api')
```

在实际的开发中，无需手动进行 `unmock` 操作，只需在 `jest.config.js` 中打开 `clearMocks` 选项即可，该选项会自动清理 mock 调用

#### 局部mock

什么是局部mock，即只针对一个文件中的部分内容进行mock，另一部分保留原始内容，不进行模拟

比如：

```javascript
// util.js
import axios from 'axios'

export const getConfig = () => {
  return axios.get('/config')
}

export const getNumber => () => 12345
```

在编写测试用例的时候，只想对 `getConfig` 方法进行mock，`getNumber` 使用真实值，不进行mock

这时就用上了 `jest.requireActual` 方法

```javascript
jest.mock('./util')
import { getConfig } from './util'
const { getNumber } = jest.requireActual('./util')

test('test axios getConfig', async () => {
  // 从 __mocks__下 util 文件中查找
  await getConfig().then((response) => {
    expect(response).toMatchObject({
      success: true
    })
  })
})

test('test getNumber', () => {
  // 使用真实函数
  expect(getNumber()).toEqual(12345)
})
```

### mock timers

对定时器进行测试

```javascript
const timerFunc = (callback) => {
	setTimeout(() => {
    callback()
  }, 3000)
}

test('test timer', (done) => {
  timerFunc(() => {
    expect(1).toBe(1)
    done()
  })
})
```

如果定时器时间过长，则测试时间也会很长，那么如何mock timer呢

```javascript
// mock timer
jest.useFakeTimers()

test('test timer', () => {
  timerFunc(() => {
    const fn = jest.fn()
    timer(fn)
    // 立即执行全部定时器
    jest.runAllTimers()
    // jest.runOnlyPendingTimers() 只运行处于已加入队列中的定时器
		expect(fn).toHaveBeenCalledTimes(1)
  })
})
```

当不知道该使用 `runAllTimers` 还是 `runOnlyPendingTimers`时，可以使用一个API进行替代，就是 `advanceTimersByTime`，其可以快进指定的时间，来让定时器提前执行，比如 

```javascript
// 定时器快进 3s 执行
jest.advanceTimersByTime(3000)
```

为了保证每一个测试用例 `timer` 相互隔离，可以将 `useFakeTimers` 提至 `beforeEach` 钩子函数中

```javascript
beforeEach(() => {
	jest.useFakeTimers()
})
```

### ES6中类的测试

对 `class` 中负责逻辑的测试

```javascript
// util.js
class Util {
  init() {
    
  }
  complex() {
    
  }
}

export default Util
```

实例化 `Util`

```javascript
let util
beforeAll(() => {
	util = new Util()
})
```

实际开发中一般对 `Util` 进行 mock

```javascript
// jest mock 发现util是一个类，会自动把类的构造函数和方法变成 jest.fn()
jest.mock('./util')
import Util from './util'

let util
beforeAll(() => {
	util = new Util()
})

test('test Util', () => {
  expect(Util).toHaveBeenCalled()
  util.complex('complex')
  expect(Util.mock.instances[0].complex).toHaveBeenCalled()
})
```

也可以采用自定义mock，在 `__mocks__` 下创建 `util.js`

```javascript
const Util = jest.fn(() => {
  console.log('constructor')
})

Util.prototype.complex = ject.fn(() => {
  console.log('complex')
})

export default Util
```

自定义mock的另一种方式

```javascript
jest.mock('./util', () => {
  const Util = jest.fn(() => {
    console.log('constructor')
  })

  Util.prototype.complex = ject.fn(() => {
    console.log('complex')
  })
  
  return Util
})
```

这里推荐在 `__mocks__` 文件夹下进行自定义mock

### jest中对DOM节点操作的测试

创建DOM节点

```javascript
import $ from 'jquery'

const addDivToBody = () => {
  $('body').append('<div/>')
}

test('test dom', () => {
  addDivToBody()
  expect($('body').find('div').length).toBe(1)
})
```

Nodejs不具备dom，jest在node环境中自己模拟了一套dom环境 `jsDom`