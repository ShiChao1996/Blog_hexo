---
 title: redux your app
 tags: 
    - redux
    - react
 cover: https://image.littlechao.top/20180204020725000005.jpg
 author:
    nick: Lovae
 date: 2017-10-8

 subtitle:
    使用 redux 管理你的 React Web APP
 categories: FE

---

### redux简介

简单来说，redux 就是帮我们统一管理了 react 组件的 state 状态。

为什么要使用 redux 统一管理 state 呢？没有 redux 我们依旧可以开发 APP，但是当 APP 的复杂度到达一定程度的时候，摆在我们面前的就是 难以维护 的代码（其中包含组件大量的异步回调，数据处理等等），但是使用 redux 也会增加我们整个项目的复杂度，这就需要我们在两者之间进行权衡了，对于这一部分，redux 开发者给我们下面几个参考点：

>以下几种情况不需要使用 redux：

* 整体 UI 很简单，没有太多交互。

* 不需要与服务器进行大量交互，也没有使用 WebSocket。

* 视图层只从单一来源获取数据。

>以下几种情况可考虑使用 redux：

* 用户的交互复杂。

* 根据层级用户划分功能。

* 多个用户之间协作。

* 与服务器大量交互，或使用了 WebSocket。

* 视图层需要从多个来源获取数据。


总结以上内容：redux 适用于 多交互，多数据源，复杂程度高的工程中。

也就是说，当我们的组件出现 某个状态需要共享，需要改变另一个组件状态 等传值比较不容易的情况。就可以考虑 redux ，当然还有其他 redux 的替代产品供我们使用。。

### 重要内容
![](http://upload-images.jianshu.io/upload_images/2041009-06f5967c685f961b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>为了避免混乱，我们不应该直接修改 redux 的状态，需要有特定的办法来修改状态。
首先我们需要 **action** 来触发一个行为，告知 redux 我们需要修改状态，
然后应由专门生产新状态的 **reducer** 来产生新的状态

##### action
> action 是一个对象，包含这个行为的 **类型** 和必要的参数(可选)。action也可写成一个返回对象的函数(ActionCreator)

```js
function testAction(key1, key2, ...keyN) {
          return {
              type: "TEST_ACTION",
              key1: key1,
              //...
              keyN: keyN
          }
      }
```
##### reducer
> reducer 是一个纯函数，满足以下条件：
* 相同输入必须有相同输出
* 不能修改传入的参数
* 不能包含 random、Date 等非纯函数

***另外，reducer 每次返回的必须是一个全新的状态***
```js
const initialState = {
  article: {}
};

export function article(state = initialState, action) {
  switch (action.type) {
    case Actions.ADD_ARTICLE_TAG: {
      let article = Object.assign(state.article);
      if(article.tags){
        article.tags.push(action.tag);
      }else{
        article.tags = [action.tag];
      }
      return { ...state, article: article };
    }
  }
}
```

### 在 react 项目中使用 redux
##### 安装
```
 npm install --save redux
 npm install --save react-redux
 npm install --save-dev redux-devtools
```
##### 配置
###### 在 index.js 文件：
```
import {createStore, applyMiddleware} from 'redux';
import {Provider} from 'react-redux';
import thunk from 'redux-thunk';
import reducers from './src/reducers/index';

const createStoreWithMiddleware = applyMiddleware(thunk)(createStore);
const store = createStoreWithMiddleware(reducers);

const router = (
  <Provider store={store}>   // 在根组件注入store
    <Router history={hashHistory}>
      <Route path="/">
        <IndexRoute component={Login} />
        <Route path="/articles" component={Articles}/>
        <Route path="/articledetail" component={ArticleDetail}/>
      </Route>
    </Router>
  </Provider>

)

ReactDOM.render(router, document.getElementById('root'));

```
以上： 我们先引入redux和一些函数，再引入写好的 reducer ，`createStoreWithMiddleware` 函数创建了
一个顶级的管理所有状态的储存器，统一管理状态。然后将生成的 store 通过 `Provider` 组件注入，
这样为 `Provider` 的子组件提供了状态的获取途径。

###### reducers
index文件:
```js
import {combineReducers} from "redux";
import { admin } from './admin';  //我自己的 reduce 文件
import { article } from "./article";  //同上

module.exports = combineReducers( {
  admin: admin,
  article: article
});
```

reduce 文件(示例):
```js
import Actions from "../actions/config";

const initialState = {
  logged: false,
  token: ''
};

export function admin(state = initialState, action) {
  switch (action.type) {
    case Actions.USER_LOGIN: {
      let token = action.token;
      return { ...state, logged: true, token: token };
    }

    case Actions.USER_LOGOUT: {
      return { ...state, logged: false };
    }
  }

  return state;
}

```

###### actions
index: 
```js
import { admin } from "./admin";
import { article } from "./article";

module.exports = {
  ...admin,
  ...article
};
```

admin action: 
```js
import Actions from "./config";   //储存常量字符串

function login(token) {
  return {
    type: Actions.USER_LOGIN,
    token: token
  };
}

function logOut() {
  return {
    type: Actions.USER_LOGOUT,
  };
}

export const admin = {
  login,
  logOut,
};
```

config：
```js
module.exports = {
  //user
  USER_LOGIN: "USER_LOGIN",
  USER_LOGOUT:"USER_LOGOUT",


  //article
  ADD_ARTICLE_TAG: "ADD_ARTICLE_TAG",
  REMOVE_ARTICLE_TAG: "REMOVE_ARTICLE_TAG",
  SAVE_CONTENT: "SAVE_CONTENT",
  EDIT_ARTICLE: "EDIT_ARTICLE",
  CLEAR: "CLEAR",

  //http
  GET_PERSONAL_PAGE_INFO: "GET_PERSONAL_PAGE_INFO",
  GET_USER_INFO: "GET_USER_INFO"
};

```

##### 使用redux
>以上步骤已经配置好了。接下来在组件中使用redux

>首先要用到的是高阶函数 `connect`

引入 `connect`：
```js
import {connect} from "react-redux";
```
连接：
```js
function select(store) {
  return {
    logged: store.admin.logged,  // 这里写你当前组件要用到的redux里的状态
    token: store.admin.token
  }
}

export default connect(select)(componentName);  //componentName 是你的组件名称
//如果不需要redux的数据，可以这样写：
//export default connect()(componentName); 
```

这一步把当前组件和 redux 连在一起，把 `logged`、`token`传到当前组件的 props 里。
需要注意的是：还会隐藏的将 `dispatch` 方法一起注入到 props 

使用：
使用属性的时候，只需要：
```js
this.props.propName
```
如果要修改状态，不能直接赋值，应该使用在 actions 里的行为来触发 reducer 的纯函数来修改状态：
```js
import {
  login
} from '../actions/index';

...
this.props.dispatch(login());
```
这里的 dispatch 意为 分发、派遣、调度，发起一个行为（action），由reducer接收并处理

### 更多
本文只是对redux的一个入门了解，要想学的更深入还是拜读官方文档，或者去看看源码。
另外，redux 有一个最佳实践写法，是由阿里前辈推出的 [dva](https://github.com/dvajs/dva) 框架，包含了更多的理解与应用，
让redux 强化了处理异步操作的能力。推荐去看看。

##### 最后
本文是对自己写博客以及管理后台的时候遇到的问题的总结，有很多问题说的不太对，也没有仔细深究，
如果有不对的地方，还请指出。示例代码不详细的话，可以戳[这里](https://github.com/ShiChao1996/BlogAdmin)获取项目源码.

###### 欢迎来访，手动笔芯~