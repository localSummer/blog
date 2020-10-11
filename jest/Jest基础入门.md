### Jest 前端自动化测试框架基础入门

1. 常用匹配器
   - toBe() 引用比较
   - toEqual() 值比较
   - toBeTruthy 布尔值判断
   - toBeGreaterThan 数字比较
   - toBeLessThan 数字比较
   - toMatch 字符串正则匹配
   - toMatchObject 对象检测包含元素
   - toContain 数组检测包含元素
   - not 取反
2. 异步处理

使用done完成函数

```javascript
test('异步处理 - done', (done) => {
  expect.assertions(1)
	fetchData().then((response) => {
    expect(response.data).toMatchObject({
      success: true
    })
    done()
  })
})
```

test函数中直接`return Promise`

```js
test('异步处理 - return Promise', () => {
  expect.assertions(1)
  return fetchData().then((response) => {
    expect(response).toMatchObject({
      success: true
    })
  })
})
```

使用 `resolves` 或者 async...await

```javascript
test('异步处理 - resolves', () => {
  expect(fetchData()).resolves.toMatchObject({
    success: true
  })
})

test('异步处理2 - async-await', async () => {
  expect.assertions(1)
	try {
    let response = await fetchData()
    expect(response).toMatchObject({
      success: true
    })
  } catch(err) {
    
  }
})
```

3. jest中的钩子函数
   1. beforeAll 在所有测试用例执行之前调用
   2. beforeEach 每一个测试用例执行前调用
   3. afterEach 每一个测试用例执行后调用
   4. afterAll 在所有测试用例执行之后调用

4. 测试分组 `describe`
5. 钩子函数执行顺序
   1. beforeAll
   2. describe - beforeAll 只对当前 describe 块生效
   3. beforeEach
   4. describe - beforeEach 只对当前 describe 块生效
   5. describe - afterEach 只对当前 describe 块生效
   6. afterEach
   7. describe - afterAll 只对当前 describe 块生效
   8. afterAll
   9. `注意：` 不在上述钩子函数内部或测试用例内部的 `log` 率先执行
6. jest中mock-模拟函数（jest.fn()）

```javascript
test('test jest.fn', () => {
  // 可传入一个函数，函数的返回值作为模拟函数func的返回值 等价于 jest.mockImplementation(() => 'dell')
  const func = jest.fn()
  func.mockReturnValueOnce('Dell')
  // 调用外部导入的函数
  runCallback(func)
  runCallback(func)
  expect(func.mock.calls.length).toBe(2)
  
// { 
//   calls: [], 每次函数被调用信息，数组中为每次调用函数传入的参数
//   instances: [], 每次函数被实例化的信息，如 new Func()
//   invocationCallOrder: [], 函数调用顺序
//   result: [] 每次函数执行，函数返回结果信息
// }
  console.log(func.mock)
})
```

`mock` 功能包括

- 捕获函数的调用和返回结果，以及this和调用顺序
- 可以自由配置函数的返回结果
- 改变函数的内部实现

7. jest中mock-模拟模块Axios接口请求

```javascript
import axios from 'axios'

jest.mock('axios')

test.only('test axios getData', async () => {
  // mock 改变函数的内部实现
  axios.get.mockResolvedValue({ data: 'hello' })
  await getData().then((response) => {
    expect(response).toBe('hello')
  })
})
```



