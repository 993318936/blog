---
layout: post
title: 学习React Native
excerpt: "因为要移植部分复杂系统功能到手机端，毙掉微信公众号的方案，决定采用React Native"
categories: [leonids]
comments: true
---
## 阅读学习
首先参考[官方文档](https://reactnative.dev/docs/getting-started)，把RN的概念、组件用法大致看了一遍。
### 搭建开发环境的选择
>If you are new to mobile development, the easiest way to get started is with Expo CLI. Expo is a set of tools built around React Native and, while it has many features, the most relevant feature for us right now is that it can get you writing a React Native app within minutes. You will only need a recent version of Node.js and a phone or emulator. If you'd like to try out React Native directly in your web browser before installing any tools, you can try out Snack.

>If you are already familiar with mobile development, you may want to use React Native CLI. It requires Xcode or Android Studio to get started. If you already have one of these tools installed, you should be able to get up and running within a few minutes. If they are not installed, you should expect to spend about an hour installing and configuring them.

根据文档描述采用了Expo集成工具，一键傻瓜式安装。看了一下Expo的文档，功能还挺全的。例如同时集成了iOS、Android、Web三种开发环境（对于我这种客户端开发小白来说非常适合，之前有一阵也想研究RN，被复杂的安卓开发环境劝退），支持Webpack配置(但是只能用于Web开发)等等。
[Expo文档](https://docs.expo.io/)

## 上手
`npm install --global expo-cli`

`brew install watchman`
检测文件变化的工具

`expo init my-project`

因为终端提示node版本不支持所以后来要特地切换成12.13.0

和其他脚手架差不多，按需选择，一个初始化的项目就建成了。运行了一下，目录结构都一目了然，感觉还挺有意思的。接下来需要选择一个适合的UI库。
>Below (in no particular order) are some of the most popular UI Libraries. Test them all out and see what you like!
 * React Native Paper
 * React Native Elements
 * Native Base
 * React Native Material UI
 * React Native UI Kitten
 * React Native iOS Kit

以上是Expo推荐的一些UI组件，评判的标准主要是看主题颜色风格（或者是否提供定制），常用组件的覆盖数及其可定制化程度，兼容性，文档的可读性，版本维护更新迭代的频率，issues的回复速度等等。综上考虑最终选择了Native Base。

[NativeBase文档](https://docs.nativebase.io/docs/GetStarted.html)

`yarn add native-base --save`

`expo install expo-font`

重点代码，必须要等待字体文件加载完毕
{% highlight js %}
async componentDidMount() {
    await Font.loadAsync({
      Roboto: require('native-base/Fonts/Roboto.ttf'),
      Roboto_medium: require('native-base/Fonts/Roboto_medium.ttf'),
      ...Ionicons.font,
    });
    this.setState({ isReady: true });
  }
{% endhighlight %}

接下来定制路由，没啥好纠结的，官方推荐[React Navigation](https://reactnavigation.org/docs/getting-started/)

`yarn add @react-navigation/native`
`expo install react-native-gesture-handler react-native-reanimated react-native-screens react-native-safe-area-context @react-native-community/masked-view`

App.js的render方法
{% highlight js %}
import { AppLoading } from 'expo';
import {Provider} from "react-redux";
import Navigate from './config/route.js';
import { createStore } from 'redux';
import { Container, Root } from 'native-base';
import reducers from './src/reducers/index';

let store = createStore(reducers);
render() {
    const { isReady } = this.state;
    if (!isReady) {
      return <Root><AppLoading /></Root>;
    }
    return (
      <Root>
        <Container>
          <Provider store={store}>
            <Navigate />
          </Provider>
        </Container>
      </Root>
    )
  }
{% endhighlight %}

route.js
{% highlight js %}
import * as React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';
import Home from "../src/pages/home/index";
import Login from "../src/pages/user/login";

const Stack = createStackNavigator();
function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Login" headerMode="none">
        <Stack.Screen name="Home" component={Home} />
        <Stack.Screen name="Login" component={Login} />
      </Stack.Navigator>
    </NavigationContainer>

  )
}
export default App;
{% endhighlight %}

headerMode='node'是因为Header用的是NativeBase的Header，并非Navigation的，所以Header的样式暂时是写在各自组件里的，这里以后应该是可以抽成一个Header组件的。

考虑到移动端的功能只是PC端部分功能的复制版，其中的逻辑需求都还比较清楚，有大量的引用其他组件内状态的需求，所以还引入了Redux全局状态管理。

reducers/index.js
{% highlight js %}
import { combineReducers } from 'redux';
import { reducer as formReducer } from 'redux-form';
import homeReducer from './home';
import userReducer from './user';
const reducers = combineReducers(
  {
    home: homeReducer,
    user: userReducer,
    form: formReducer
  }
);
export default reducers;
{% endhighlight %}

以下列举两个简单的reducer，分别是请求验证码和登录提交

reducers/user.js
{% highlight js %}
import request from '../utils/request';
import constant from '../utils/constant'
export default function(state, {type, payload}) {
  switch(type) {
    case 'USER_LOGIN':
    async function login() {
      const { data, callback } = payload;
      try {
        const response = await request({url: `${constant.ROOT_PATH}${constant.API_PATH}/users/login`, method: 'POST', data});
        if (callback && typeof callback === 'function') callback(response);
        return {
          ...state,
          ...response,
        }
      } catch(e) {
        return null;
      }
    }
    return login();
    case 'USER_GET_CAPTCHA':
    async function captcha() {
      const { data, callback, fail } = payload;
      try {
        const response = await request({url: `${constant.ROOT_PATH}${constant.API_PATH}/base/kaptcha/image`, data: payload});
        if (callback && typeof callback === 'function') callback(response);
      } catch(e) {
        if (fail && typeof fail === 'function') fail('error');
      }
      return null;
    }
    return captcha();
    default:
      return {
        ...state
      };
  }
}
{% endhighlight %}

request.js
{% highlight js %}
const request = ({url, method = 'GET', data}) => {
  return new Promise((resolve, reject) => {
    axios(url, {
      method,
      headers: {
        accept: 'application/json',
        'Content-Type': 'application/json'
      },
      data: {
        ...data,
      }
    })
      .then(response => resolve(response.data))
      .catch(({response, request, message}) => {
                if (response) {
                  // The request was made and the server responded with a status code
                  // that falls out of the range of 2xx
                  const text = (response.data && response.data.errorMessage) || codeMessage[response.status]
                  Toast.show({
                    text,
                    type: 'danger',
                    position: 'top',
                  });
                } else if (request) {
                  // The request was made but no response was received
                  // `error.request` is an instance of XMLHttpRequest in the browser and an instance of
                  // http.ClientRequest in node.js
                  const text =
                    message.indexOf('timeout') >= 0 ? '连接超时，请稍候重试' : error.message;
                  Toast.show({
                    text,
                    type: 'danger',
                    position: 'top',
                  });
                } else {
                  // Something happened in setting up the request that triggered an Error
                  Toast.show({
                    text: error.message,
                    type: 'danger',
                    position: 'top',
                  });
                }
        reject(response)
      })
      .finally(() => {})
  })
};
{% endhighlight %}

actions/user.js
{% highlight js %}
export function login(payload){
  return{
    type: "USER_LOGIN",
    payload,
  };
}
export function getCaptcha() {
  return {
    type: 'USER_GET_CAPTCHA',
  }
}
{% endhighlight %}

在login组件内，以下是一些关键代码，包括表单的提交、校验：
{% highlight js %}
import { connect } from 'react-redux';
import axios from 'axios';
const validate = values => {
  const error= {
    userId: '',
    password: values.password? '': '请输入密码',
    verifyCode: values.verifyCode? '': '请输入验证码',
  };
  if (!values.userId) {
    error.userId = '请输入身份证';
  }else if(!/(^\d{15}$)|(^\d{18}$)|(^\d{17}(\d|X|x)$)/.test(values.userId)){
    error.userId= '身份证号错误';
  }
  return error;
};
@reduxForm({
  form: 'login',
  validate,
})
class Login extends React.Component {
  ...
  fetchCaptcha() {
      const { getCaptcha } = this.props;
      getCaptcha({
        callback: ({imgBytes, id}) => {
          this.setState({captcha: imgBytes, verifyCodeId: id});
        },
        fail: () => {
        // 失败后重新请求验证码
          this.fetchCaptcha();
        }
      })
    }
    renderInput({ input, label, type, meta: { touched, error, warning } }){
        let hasError= false;
        if(error !== undefined){
          hasError= true;
        }
        return(
          <Item error={touched&& hasError}>
            <Input {...input} placeholder={label} type={type}/>
            {touched&& hasError ? <Text>{error}</Text> : <Text />}
          </Item>
        )
      }
      onSubmit(values) {
          const { login } = this.props;
          const { verifyCodeId } = this.state;
          login({
            data: values,
            callback: (response) => {
              if (response && response.token) {
                axios.defaults.headers.common.Authorization = response.token;
                localStorage.setItem('user-info', JSON.stringify({ ...response }));
                navigation.navigate('Home');
              }
            }
          });
        }
        render() {
            const { navigation, handleSubmit } = this.props;
            const { captcha } = this.state;
            return (
              <Container>
                <Header>
                  <Left>
                    <Button transparent>
                      <Icon name='arrow-back' onPress={() => navigation.goBack()} />
                    </Button>
                  </Left>
                  <Body>
                    <Title>登录</Title>
                  </Body>
                  <Right />
                </Header>
                <Content padder>
                  <Form>
                    <Field name="userId" type='text' component={this.renderInput} label='身份证' />
                    <Field name="password" type='password' component={this.renderInput} label='密码' />
                    <Field name="verifyCode" type='text' component={this.renderInput} label='验证码' />
                    <Item style={styles.captchaImg}>
                        {captcha? <Image source={{uri: captcha}} style={{width: 100, height: 100, resizeMode: 'contain'}} />: null}
                        <Button transparent light onPress={this.fetchCaptcha}>
                          <Icon name='refresh' />
                        </Button>
                    </Item>
                    <Button block primary style={styles.loginButton} onPress={handleSubmit(this.onSubmit.bind(this))}>
                      <Text>登录</Text>
                    </Button>
                  </Form>
                </Content>
              </Container>
            )
          }
}
const matchDispatchToProps = (dispatch) => {
  return {
    login: payload => {
      dispatch({
        type: 'USER_LOGIN',
        payload,
      })
    },
    getCaptcha: payload => {
      dispatch({
        type: 'USER_GET_CAPTCHA',
        payload,
      })
    }
  }
};
const WrapperLogin = connect(null, matchDispatchToProps)(Login);
export default WrapperLogin;
{% endhighlight %}

因为暂时不太熟悉OSX是如何操作代理的，所以只能和以前一样先用webpack做代理配置，只支持web环境！！

webpack.config.js
{% highlight js %}
module.exports = async function (env, argv) {
  const config = await createExpoWebpackConfigAsync(env, argv);
  // Customize the config before returning it.
  if (config.mode === 'development') {
    config.devServer = {
      proxy: {
        [your_path]: {
          target: Address.line,
          changeOrigin: true,
        },
        '/assets': {
          target: Address.line,
          changeOrigin: true,
        },
      }
    }
  }
  return config;
};
{% endhighlight %}

以上一套基本的RN框架就搭建好了。
