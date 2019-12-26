如何在Flutter中实现Redux (Implementing Redux architecture with Flutter)

最近有机会做一个Flutter商业项目。之前也试过各种不同的Flutter架构，但这次我决定用Redux，因为觉得有趣而且有挑战性。

为了方便分享我的经验，我就以注册流程为例(Welcome-Signin-Signup)。

在继续阅读本文之前，请先了解一下：
1. 关于Flux的理论知识，以及Redux框架。就是说，你应该明白什么是Store, 为什么说它是Single Soure of Truth，什么是Reducer，为什么它被称为"Vanilla Function"，等等。。。
2. 对redux和flutter_redux插件有基本的了解

我有四个页面：
1. 欢迎页面 (Welcome)
2. 登陆页面 (Sign-In)
3. 首页 (Home)
4. 注册页面 (Sign-Up)

首先，我要创建AppState, AppReducer和Store实体。这些实体是全局的，但同时State是不可改变的，意思就是说，任何变化发生，你都要重新创建它。
在我的实现中，AppState是担当页面模块States的载体：

```
@immutable
class AppState{

  final AuthState authState;
  final SignInState signInState;

  AppState({
    @required this.authState,
    @required this.signInState
  });

  factory AppState.initial(){
    return AppState(
      authState: AuthState.initial(),
      signInState: SignInState.initial()
    );
  }

  AppState copyWith({
    AuthState authState,
    SignInState signInState,
  }){
    return AppState(
      authState: authState ?? this.authState,
      signInState: signInState ?? this.signInState
    );
  }
}
```

AppReducer也是如此:
```
AppState appReducer(AppState state, dynamic action) =>
    new AppState(
        authState: authReducer(state.authState,action),
        signInState: signinReducer(state.signInState,action)
    );
```

AppState中的SignInState包含了一堆业务逻辑：
```
@immutable
class SignInState{


  final ScreenState type;
  final LoadingStatus loadingStatus;
  final String password;
  final String passwordError;
  final String email;
  final String emailError;
  final String code;
  final String codeError;

  SignInState({this.type, this.loadingStatus, this.password, this.passwordError,
      this.email, this.emailError, this.code, this.codeError});


  SignInState copyWith({
    ScreenState type,
    LoadingStatus loadingStatus,
    String password,
    String passwordError,
    String retypePassword,
    String retypePasswordError,
    String email,
    String emailError,
    String token,
    String code,
    String codeError,
  }){
    return new SignInState(
        type: type ?? this.type,
        loadingStatus: loadingStatus ?? this.loadingStatus,
        password: password ?? this.password,
        passwordError: passwordError ?? this.passwordError,
        email: email ?? this.email,
        emailError: emailError ?? this.emailError,
        code: code ?? this.code,
        codeError: codeError ?? this.codeError
    );
  }

  factory SignInState.initial(){
    return new SignInState(
        type: ScreenState.WELCOME,
        loadingStatus: LoadingStatus.success,
        password: "",
        passwordError: "",
        email: "",
        emailError: "",
        code: "",
        codeError: "");

  }
}
```

请注意，SignInState的copyWith(….) 会替换掉老的状态所包含的属性，来创建一个新的状态。

我使用combineReducers<SignInState>()作为SignInState的reducer。简单来说，combineReducer接受一个TypedReducer的数组，它自身是一个包含许多reducer的reducer，这样可以很爽的构建工程结构。

SignInReducer长这个样子：
```
final signinReducer = combineReducers<SignInState>([
  TypedReducer<SignInState,ValidateEmailAction>(_validateEmail),
  TypedReducer<SignInState,ValidatePasswordAction>(_validatePassword),
  TypedReducer<SignInState,ValidateLoginFields>(_validateLoginFieldsAction),
  TypedReducer<SignInState,ChangeLoadingStatusAction>(_changeLoadingStatusAction),
  TypedReducer<SignInState,EmailErrorAction>(_emailErrorAction),
  TypedReducer<SignInState,PasswordErrorAction>(_passwordErrorAction),
  TypedReducer<SignInState,SaveTokenAction>(_saveToken),
  TypedReducer<SignInState,ConfirmForgotPasswordAction>(_confirmCodeAction),
  TypedReducer<SignInState,CheckTokenAction>(_checkTokenAction),
  TypedReducer<SignInState,ClearErrorsAction>(_clearErrorsAction),
  TypedReducer<SignInState,ChangeScreenStateAction>(_changeScreenStateAction),
]);

...

SignInState _validateEmail(SignInState state, ValidateEmailAction action){
  return state.copyWith(email: action.email);
}

SignInState _validatePassword(SignInState state, ValidatePasswordAction action) =>
    state.copyWith(password: action.password);
....
```
我用同样的方式创建了SignUpState和SignUpReducer，你可以去看看我的代码（本文末尾有链接）。

然后，创建Store就不费吹灰之力了：
```
import 'dart:async';
import 'package:redux/redux.dart';
import 'package:reduxsample/redux/app/app_state.dart';
import 'package:reduxsample/redux/middleware/local_storage_middleware.dart';
import 'package:reduxsample/redux/middleware/navigation_middleware.dart';
import 'package:reduxsample/redux/middleware/validation_middleware.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:redux_logging/redux_logging.dart';
import 'package:reduxsample/redux/app/app_reducer.dart';

Future<Store<AppState>> createStore() async {
  var prefs = await SharedPreferences.getInstance();
  return Store(
    appReducer,
    initialState: AppState.initial(),
    middleware: [
      ValidationMiddleware(),
      LoggingMiddleware.printer(),
      LocalStorageMiddleware(prefs),
      NavigationMiddleware()
    ],
  );
}
```
在上面的代码中，你会发现，Store接受主要的AppReducer，一个AppState实例，以及一个Middleware的数组。

中间件(Middleware)负责产生副作用(side-effect)以及业务逻辑相关的工作(一般来说是异步的)。经过中间件处理的动作（Action），可能会被拒绝或允许继续被Reducer处理。

中间件也可以在必要的时候，分发新的Action，例如，下面的例子演示了中间件如何处理字段验证：
```
import 'dart:async';
import 'package:redux/redux.dart';
import 'package:reduxsample/models/auth_request.dart';
import 'package:reduxsample/models/loading_status.dart';
import 'package:reduxsample/redux/app/app_state.dart';
import 'package:reduxsample/redux/auth/auth_actions.dart';
import 'package:reduxsample/redux/auth/screen.dart';
import 'package:reduxsample/utils/strings.dart';

class ValidationMiddleware extends MiddlewareClass<AppState>{

  final String emailPattern = r'^(([^<>()[\]\\.,;:\s@\"]+(\.[^<>()[\]\\.,;:\s@\"]+)*)|(\".+\"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$';

  @override
  void call(Store<AppState> store, dynamic action, NextDispatcher next) {
      if(action is ValidateEmailAction){
        validateEmail(action.screen,action.email, next);
      }

      if(action is ValidatePasswordAction){
        validatePassword(action.screen,action.password, next);
      }

      if(action is ValidatePasswordMatchAction){
        validatePassMatch(action.screen,action.password, action.confirmPassword, next);
      }
      ...
      next(action);
  }

  void validatePassMatch(Screen screen, String password, String confirmPassword, NextDispatcher next) {
    if(password != confirmPassword){
      next(new RetypePasswordErrorAction(password_match_error,screen));
    }else{
      next(new RetypePasswordErrorAction("",screen));
    }
  }

  void validatePassword(Screen screen, String password, NextDispatcher next) {
    if(password.length<6){
      next(new PasswordErrorAction(password_error,screen));
    }else{
      next(new PasswordErrorAction("",screen));
    }
  }

  void validateEmail(Screen screen, String email, NextDispatcher next) {
    RegExp exp = new RegExp(emailPattern);
    if(!exp.hasMatch(email)){
      next(new EmailErrorAction(email_error,screen));
    }else{
      next(new EmailErrorAction("",screen));
    }
  }
}
```

到此为止，本项目的Dart部分已经完成。现在来看看，如何在Flutter中实现这一流程。

首先，我们在main函数里面创建Store实体对象。
然后，用StoreProvider来包装App Widget。
下面是代码：
```
import 'package:flutter/material.dart';
import 'package:redux/redux.dart';
...

void main() async{
  SystemChrome.setPreferredOrientations([
    DeviceOrientation.portraitDown,
    DeviceOrientation.portraitUp,
  ]);
  var store = await createStore();
  runApp(new App(store));
}

class App extends StatefulWidget {
  final Store<AppState> store;

  App(this.store);

  @override
  _AppState createState() => _AppState();
}

class _AppState extends State<App> {
  @override
  Widget build(BuildContext context) {
    return new  StoreProvider<AppState>(
      store: widget.store,
      child: new MaterialApp(
        theme: new ThemeData(
            brightness: Brightness.dark,
            // ignore: strong_mode_invalid_cast_new_expr
            primaryColor: const Color(0xFF000000),
            accentColor: const Color(primaryPink),
        ),
        home: SignIn(),
        navigatorKey: Keys.navKey,
        routes:  <String, WidgetBuilder>{
          "/signin": (BuildContext context) => new SignIn(),
          "/signup": (BuildContext context) => new SignUp(),
        }
      ),
    );
  }
}
```
同时，我也添加了一个NavigatorKey， 作为GlobalKey的一个静态实例，并将其移到了一个独立的文件 - 现在我们可以用它在中间件里面进行导航。

现在我们来看看StoreConnector，它是Flutter Redux不可缺少的成员。

StoreConnector能将ViewModel连接到AppState和Store。通常，它位于页面组件的Root层，这样StoreConnector可以罩住整个页面，代码如下：
```
@override
  Widget build(BuildContext context) => new Scaffold(
    body: LayoutBuilder(
      builder: (context, constraints)=>SafeArea(
        child: new Container(
          color: Colors.black,
          child: new StoreConnector<AppState,LoginViewModel>(
              onInit: (store){
                ...
              },
              converter: (store) => LoginViewModel.fromStore(store),
              builder: (_, viewModel) => content(viewModel,constraints),
          ),
        ),
      ),
    ),
  );
```

StoreConnector需要两个属性，converter和builder:
1. converter是一个函数，它接受Store作为参数，创建并返回一个ViewModel实例。
2. builder使用新创建的viewModel来渲染UI。

除此之外，StoreConnector还有一些有用的回调属性，例如onInit, onWillChange, onDidChange 等等。。。你会看到在代码里面，处理过viewModel之后我实现了onInit callback。

回调是很叼的，但是在StoreConnector中，ViewModel才是最重要的角色，在每个页面中viewModel也是我们使用最多的实体对象。
然而大多数Redux教程都没有介绍这个重要的概念，而是使用一些简单的列表来糊弄读者，这就有点二逼了。。。

下面是Sign-In页面的ViewModel:
```
import 'package:redux/redux.dart';
import 'package:reduxsample/models/loading_status.dart';
import 'package:reduxsample/redux/app/app_state.dart';
import 'package:reduxsample/redux/auth/auth_actions.dart';
import 'package:reduxsample/redux/auth/screen_state.dart';
import 'package:reduxsample/redux/auth/screen.dart';

class LoginViewModel{

  final LoadingStatus status;
  final ScreenState type;
  final String password;
  final String passwordError;
  final String email;
  final String emailError;
  
  final Function(String) validateEmail;
  final Function(String) validatePassword;
  final Function(String email, String password) login;
  final Function clearError;
  final Function navigateToRegistration;


  LoginViewModel({this.status,
    this.type,
    this.password,
    this.passwordError,
    this.email,
    this.emailError,
    this.validateEmail,
    this.validatePassword,
    this.login,
    this.clearError,
    this.navigateToRegistration});


  static LoginViewModel fromStore(Store<AppState> store){
    return LoginViewModel(
      status: store.state.signInState.loadingStatus,
      type: store.state.signInState.type,
      email: store.state.signInState.email,
      emailError: store.state.signInState.emailError,
      password:store.state.signInState.password,
      passwordError: store.state.signInState.passwordError,
      validateEmail: (email) => store.dispatch(new ValidateEmailAction(email,Screen.SIGNIN)),
      validatePassword: (password) =>store.dispatch(new ValidatePasswordAction(password,Screen.SIGNIN)),
      login: (email, password) {
        store.dispatch(new ValidateLoginFields(email, password));
      },
      clearError: () => store.dispatch(new ClearErrorsAction()),
      navigateToRegistration: () => store.dispatch(new NavigateToRegistrationAction())
    );
  }
}
```

你应该看得出来，这个类的前半部分初始化了widget状态，后半部分的函数会触发事件和动作。

在ViewModel中的函数只能分发新的动作 - 否则，就会破坏唯一真理原则(Single Source of Truth principle)。
接下来，使用ViewModel来初始化UI就很简单了：
```
Widget content(LoginViewModel viewModel, BoxConstraints constraints) =>
      (viewModel.status != LoadingStatus.loading || viewModel?.type == ScreenState.WELCOME)? new Container(
      child: new Stack(
          fit: StackFit.expand,
          alignment: Alignment.topCenter,
          children: <Widget>[
            new Align(
              alignment: Alignment.topCenter,
              child: new Transform(
                  transform: new Matrix4.translationValues(0.0, _animation.value, 0.0),
                  child: SingleChildScrollView(
                    child: new Column(
                      children: <Widget>[
                       ...
                        viewModel.type == ScreenState.SINGIN?new Column(
                          mainAxisSize: MainAxisSize.max,
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: <Widget>[
                            emailInput(viewModel),
                            (viewModel.status == LoadingStatus.error && viewModel.emailError.isNotEmpty)?emailError(viewModel):const SizedBox(),
                            ...
                            Padding(
                              padding: const EdgeInsets.only(left: 65.0,right: 65.0,top: 16.0),
                              child: Row(
                                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                                mainAxisSize: MainAxisSize.max,
                                children: <Widget>[
                                  ...
                                  loginBtn((){
                                    viewModel.login(viewModel.email,viewModel.password);
                                  })
                                ],
                              ),
                            ),
                            ...
                            termsAndConditions(),
                            new Padding(padding: const EdgeInsets.all(16.0))
                          ],
                        ):Container()
                      ],
                    ),
                  ),
              ),
            ),
          ],
      ),
  ):new Center(
    child: new CircularProgressIndicator(),
  );
```

基本上，我们在这里所做的跟之前对Stateful Widget State所做的一样，初始化UI并根据events来分发action，然后重建AppState，然后改变viewModel，等等。。。让我们以Email输入为例，看看具体怎么做：
```
Widget emailInput(LoginViewModel viewModel) => Padding(
    padding: const EdgeInsets.only(left: 65.0,right: 65.0,top: 26.0),
    child: Container(
      alignment: new Alignment(0.5, 0.5),
      height: 56.0,
      decoration: BoxDecoration(
        color: Color(semiTransparentGray),
        border: Border(bottom: BorderSide(
            color: new Color(_getColor(viewModel.status, viewModel.emailError, _emailNode)),
            width:1.0
        )),
      ),
      child:  Padding(
        padding: const EdgeInsets.only(left: 16.0,bottom: 8.0),
        child: new TextField(
          focusNode: _emailNode,
          style: TextStyle(
            fontSize: 17.0
          ),
          keyboardType: TextInputType.emailAddress,
          controller: _emailController,
          decoration: new InputDecoration(
              labelText: email,
              labelStyle: new  TextStyle(
                color: new Color(_getColor(viewModel.status, viewModel.emailError, _emailNode))
              ),
              border: InputBorder.none
          ),
          onChanged: (email){
            viewModel.validateEmail(email);
          },
        ),
      ),
    ),
  );
```

在上面的代码中，每一次输入都会触发一个ValidateEmailAction，并转发输入的字符串。然后，用户所输入的Email会在ValidationMiddleware中使用正则表达式来验证。如果Email验证成功，ValidationMiddleware将返回一个包含空字符串的EmailErrorAction。否则，ValidationMiddleware返回一个带有错误消息的EmailErrorAction，并且会改变AppState。ViewModel会得到这个错误消息，但不会把它显示出来。

如果用户按下了登录按钮，ViewModel会发送一个ValidateLoginFields动作给ValidationMiddleware。如果有任何错误消息，ValidationMiddleware会改变AppState以及ViewModel的状态属性为LoadingState.Error，然后显示所有的非空错误消息给用户。如果EmailErrorAction不包含错误，ValidationMiddleware则会发送一个SignInAction，给RestMiddleware。RestMiddleware会调用登录api，如果请求返回200，RestMiddleware会发送一个NavigationToHomeAction，使用NavigationMiddleware跳到另一个页面：
```
class ValidationMiddleware extends MiddlewareClass<AppState>{

  final String emailPattern = r'^(([^<>()[\]\\.,;:\s@\"]+(\.[^<>()[\]\\.,;:\s@\"]+)*)|(\".+\"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$';

  @override
  void call(Store<AppState> store, dynamic action, NextDispatcher next) {
      if(action is ValidateEmailAction){
        validateEmail(action.screen,action.email, next);
      }

      ...

      if(action is ValidateLoginFields){
        validateEmail(Screen.SIGNIN,action.email, next);
        validatePassword(Screen.SIGNIN,action.password, next);
        RegExp exp = new RegExp(emailPattern);
        if(!exp.hasMatch(action.email) || action.password.length<6){
          next(ChangeLoadingStatusAction(LoadingStatus.error));
        }else{
          next(new SignInAction(new AuthRequest(action.email,action.password)));
        }
      }
      ...
      next(action);
  }
```

这听起来虽然有点复杂，但是其实代码很简单。并且，如果你使用额外的工具，比如redux_loggin和dev_tools，代码会变得更加自解释(self-explanatory)。

现在让我们回到StoreConnector的回调属性，没有他们，本教程会变得非常不完整。。。让我们考虑一个简单的情况，每次进入欢迎页面，我们都要检查authorization token，如果有token，我们进入app主界面，如果没有，我们进入登录页面。

StoreConnector的onInit回调就是一个绝佳的选择。OnInit为我们提供了一个Store实例，也就意味着我们可以发送动作。当我们发送一个CheckTokenAction，这个动作会被发送到LocalStorageMiddleware，在那里使用SharedPreferences检查token。让我们来看看两种主要的路径：
1. 简单情况：在CheckTokenAction中发送两个回调函数：noToken，会启动登录页面的过场动画，hasToken，会跳到主界面。我是在LocalStorageMiddleware中触发这些回调的 -因为在中间件中，仍然会发生side-effects，所以我认为这样做不会破坏整体架构：
```
//onInit ^ StoreConnector
onInit: (store){
                store.dispatch(new ClearErrorsAction());
                store.dispatch(new CheckTokenAction(
                    hasTokenCallback: (){

                    },
                    noTokenCallback: (){
                      _animation.addStatusListener((status){
                        if(status == AnimationStatus.dismissed || status == AnimationStatus.completed){
                          store.dispatch(new ChangeScreenStateAction(ScreenState.SINGIN));
                        }
                      });
                      _animation.addListener((){
                        setState((){});
                      });
                      _controllerAnim.forward();
                    }
                ));
  
  //middleware
  class LocalStorageMiddleware extends MiddlewareClass<AppState>{

  final SharedPreferences preferences;

  LocalStorageMiddleware(this.preferences);

  @override
  void call(Store<AppState> store, dynamic action, NextDispatcher next) {
    next(action);
    if(action is CheckTokenAction){
      var token = preferences.getString(TOKEN);
      if(token!=null && token.isNotEmpty){
        action.hasTokenCallback();
      }else{
        action.noTokenCallback();
      }
    }
  }
}
```

2. 更简洁的代码：在改变模块viewModel的时候发送一个独立的事件。
比如说，我们可以注册StoreConnector的onDidChange回调，这个回调会在每一次ViewModel发生变化时被触发，然后我们可以根据ViewModel的变化值，来决定是进入登录还是主界面：
```
child: new StoreConnector<AppState,LoginViewModel>(
              onInit: (store){
                store.dispatch(new ClearErrorsAction());
                store.dispatch(new CheckTokenAction());
              },
              onDidChange: (viewModel){
                if(viewModel.tokenReqeust == true){
                  if(viewModel.token.isEmpty()){
                    // TODO run animation
                  }else{
                    //TODO Navigate to home
                   }
                }
              },
              converter: (store) => LoginViewModel.fromStore(store),
              builder: (_, viewModel) => content(viewModel,constraints),
          ),
```
这一种写法的弊端是，你可能需要写更多的代码。。。
以上两种方式都可以达到同样的目的，怎么写还是在于你的决定。

最后，代码在这里，欢迎参考：
https://github.com/swat-cat/reduxsample

其实关于Redux的正确姿势，我也才刚刚找到点感觉，也许在你阅读本文的时候，我的理解又发生了很大的变化。如果对于如何改进上述代码有任何的想法或者点子，请分享给我，我会非常感激。文章到此结束，希望你们的Flutter之旅一路顺风，下篇文章见：）

