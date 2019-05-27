# MVP概述
常见的架构模式分为`MVC`、`MVP`和`MVVM`三种，目前MVP在Android中的应用较多。
* `View`：对应Activity和Fragment，负责View的绘制以及与用户交互
* `Model`：业务逻辑和实体模型
* `Presenter`：负责完成View和Model之间的交互
# 简单的登录实践
参考googlesamples/android-architecture中的todo-mvp项目的组织结构和鸿洋的浅谈MVP的登录业务。 
![MVP代码结构](https://upload-images.jianshu.io/upload_images/3468445-aac16c85fbea8098.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
效果如下图
![mvp.gif](https://upload-images.jianshu.io/upload_images/3468445-7c1a4623c3ab3c36.gif?imageMogr2/auto-orient/strip)


# 1.Model
首先定义登录中使用的`User`实体类
```
package com.hfy.mvp.bean;

public class User {
    private String userName;
    private String password;

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```
然后定义登录的`回调接口`
```
public interface OnLoginListener {
    void onLoginSuccess(User user);

    void onLoginFailed();
}
```
登录业务
```
public class LoginBiz {
    public static void login(final String userName, final String password, final OnLoginListener loginListener) {
        //模拟子线程登录过程
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                if (userName.equals("messiahfy") && password.equals("999")) {
                    User user = new User();
                    user.setUserName(userName);
                    user.setPassword(password);
                    loginListener.onLoginSuccess(user);
                } else {
                    loginListener.onLoginFailed();
                }
            }
        }).start();
    }
}
```
Model层即为子线程暂停两秒模拟登录过程，登录结果用回调接口来通知。

# 2.View和Presenter

首先定义`View`和`Presenter`的基础接口
* Presenter基础接口：
```
public interface BasePresenter {
    //此接口方法在Google的示例中用于每个功能开始要执行的代码，例如在activity中的onResume()中执行
    //此实践未用
    void start();
}
```
* View基础接口：
```
public interface BaseView<T> {
    void setPresenter(T presenter);
}
```
然后在login包中，定义登录功能的契约接口LoginContract，此接口相当于登录功能中View和Presenter总体需求描述。
```
public interface LoginContract {
    interface View extends BaseView<Presenter> {
        String getUserName();

        String getPassword();

        void cleanUserName();

        void cleanPassword();

        void showLoading();

        void hideLoading();

        void toMainActivity(User user);

        void showFailedError();
    }

    interface Presenter extends BasePresenter {
        void login();
    }
}
```
在login包中LoginActivity为LoginContract.View的实现类。
```
public class LoginActivity extends AppCompatActivity implements LoginContract.View {

    private LoginContract.Presenter mPresenter = new LoginPresenter(this);
    private EditText userNameEt;
    private EditText passwordEt;
    private Button loginBtn;
    private Button clearBtn;
    private ProgressBar progressBar;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initView();
    }

    private void initView() {
        userNameEt = findViewById(R.id.username_et);
        passwordEt = findViewById(R.id.passwrod_et);
        loginBtn = findViewById(R.id.login_btn);
        clearBtn = findViewById(R.id.clear_btn);
        progressBar = findViewById(R.id.progress_bar);

        loginBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mPresenter.login();
            }
        });
        clearBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                cleanUserName();
                cleanPassword();
            }
        });
    }

    @Override
    public String getUserName() {
        return userNameEt.getText().toString();
    }

    @Override
    public String getPassword() {
        return passwordEt.getText().toString();
    }

    @Override
    public void cleanUserName() {
        userNameEt.setText("");
    }

    @Override
    public void cleanPassword() {
        passwordEt.setText("");
    }

    @Override
    public void showLoading() {
        progressBar.setVisibility(View.VISIBLE);
    }

    @Override
    public void hideLoading() {
        progressBar.setVisibility(View.INVISIBLE);
    }

    @Override
    public void toMainActivity(User user) {
        Toast.makeText(this, user.getUserName(), Toast.LENGTH_SHORT).show();
    }

    @Override
    public void showFailedError() {
        Toast.makeText(this, "登陆失败", Toast.LENGTH_SHORT).show();
    }

    @Override
    public void setPresenter(LoginContract.Presenter presenter) {
        mPresenter = presenter;
    }
}
```
LoginPresenter如下
```
public class LoginPresenter implements LoginContract.Presenter {
    private LoginContract.View mLoginView;
    private Handler handler = new Handler();

    public LoginPresenter(LoginContract.View loginView) {
        mLoginView = loginView;
        mLoginView.setPresenter(this);
    }

    @Override
    public void login() {
        mLoginView.showLoading();
        LoginBiz.login(mLoginView.getUserName(), mLoginView.getPassword(), new OnLoginListener() {
            @Override
            public void onLoginSuccess(final User user) {
                //切换回UI线程
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        mLoginView.hideLoading();
                        mLoginView.toMainActivity(user);
                    }
                });
            }

            @Override
            public void onLoginFailed() {
                //切换回UI线程
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        mLoginView.hideLoading();
                        mLoginView.showFailedError();
                    }
                });
            }
        });
    }

    @Override
    public void start() {

    }
}
```
Activity中将Presenter作为实例域，并将自身传入Presenter的构造方法，达到View和Presenter相互持有的效果。


