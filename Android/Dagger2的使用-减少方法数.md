
Dagger2的使用-减少方法数
================================
　　Dagger2--一个完全静态的，编译时依赖注入框架，是Azimo Android app的代码支柱。我们已经意识到，随着开发团队的增长，干净的代码架构（Clean Code structure）已经是所有项目中最重要的事情。低耦合、易测试和更好的扩展性--这些只是使用像Dagger这样的依赖注入框架的一小部分好处.  
　　现在教你如何在android项目中开始使用dagger和依赖注入的说明和代码到处都是，如果你需要，下 面这些可能帮助你：  

​	[Introduction to Dependency Injection](http://frogermcs.github.io/dependency-injection-with-dagger-2-introdution-to-di/)

 　  	[Dagger 2 API](http://frogermcs.github.io/dependency-injection-with-dagger-2-the-api/)
 　  	[Dagger 2 — custom scopes](http://frogermcs.github.io/dependency-injection-with-dagger-2-custom-scopes/)  
 　	[Dagger 2 — graph creation performance](http://frogermcs.github.io/dagger-graph-creation-performance/)

 　	[Dependency injection with Dagger 2 — Producers](https://medium.com/@froger_mcs/dependency-injection-with-dagger-2-producers-c424ddc60ba3#.o9ks10hzp)

 　 	[Async Injection in Dagger 2 with RxJava](https://medium.com/@froger_mcs/async-injection-in-dagger-2-with-rxjava-e7df503343c0#.5vuheb5ui)
 　 	[Inject everything — ViewHolder and Dagger 2 (with Multibinding and AutoFactory example)](https://medium.com/@froger_mcs/inject-everything-viewholder-and-dagger-2-e1551a76a908#.ojfwmz35r)
　　但是这些文章都仅仅告诉你怎么开始去用DI,这就是为什么我们今天想分享我们两年来使用Dagger2的经验。  
　　＠Component vs Subcomponent
　　一般来说简介中Custom Scopes一般都讲得不清楚，但是这确实是Dagger2中一个强劲的功能。在复杂app中仅仅使用@Singleton 并不会让你更容易。  
　　让我们来看个简单的例子--我们有一些和当前黄金直接关联的依赖，比如用户简历页面的presenter。相比每次想要数据的时候去找数据仓库提取，我们还可以创建一个@User Scope来以一个单实例的形式在userComponent中保持依赖。
　　我们接下来讨论一下怎么做，无论你是否会在生产环境中使用自定义scope。
　　在Dagger2中我们有两种方法来创建自定义Scopes和Component用以内建和扩展对象图。
　　我们可以创建另一个@component明确表明是扩展component  

``` java
@UserScope
@Component(
    modules = UserModule.class,
    dependencies = AppComponent.class
)
public interface UserComponent {
    UserProfileActivity inject(UserProfileActivity activity);
}
```

 或者我们可以定义一个@subcomponent，通过基础component的的虚拟工厂方法创建  

``` java
 
@UserScope
@Subcomponent(
    modules = UserModule.class
)
public interface UserComponent {
    UserProfileActivity inject(UserProfileActivity activity);
}

//===== AppComponent.java =====

@Singleton
@Component(modules = {...})
public interface AppComponent {
    // Factory method to create subcomponent
    UserComponent plus(UserModule module);
}
```
###两种方法的区别##

 	文档告诉我我们区别在依赖可见度上。@subcompoent可以访问父component的所有依赖，但是@component只能访问基component的暴露在外的接口。

​	知道这些之后，我们开始挑一个方法来依赖。我们项目中所有的component都依赖于AppComponent 而且被@Singleton注解。

### 关Methods数量屁事…###

​	但是Component和SubCompo还有个重要的区别。我们来看下生成代码，SubcbCompoent显示只不过是基Component的一个内部类。

​	所以依赖于AppComponent的**UserProfileActivityComponent**的生成diama代码是这样子的：

```java
public final class DaggerAppComponent implements AppComponent {

    //...AppComponent code...

    private final class UserProfileActivityComponentImpl implements UserProfileActivityComponent {
        private final UserProfileActivityComponent.UserProfileActivityModule userProfileActivityModule;
        private Provider<UserProfileActivity> provideActivityProvider;
        private Provider<UserProfileActivityPresenter> userProfileActivityPresenterProvider;
        private MembersInjector<UserProfileActivity> userProfileActivityMembersInjector;

        private UserProfileActivityComponentImpl(
            UserProfileActivityComponent.UserProfileActivityModule userProfileActivityModule) {
            this.userProfileActivityModule = Preconditions.checkNotNull(userProfileActivityModule);
            initialize();
        }

        private void initialize() {
            this.provideActivityProvider = DoubleCheck.provider(BaseActivityModule_ProvideActivityFactory.create(userProfileActivityModule));

            this.userProfileActivityPresenterProvider = DoubleCheck.provider(
                UserProfileActivityPresenter_Factory.create(
                    MembersInjectors.<UserProfileActivityPresenter>noOp(),
                    provideActivityProvider,
                    DaggerAppComponent.this.logoutManagerProvider,
                    DaggerAppComponent.this.userManagerProvider)
            );

            this.userProfileActivityMembersInjector = UserProfileActivity_MembersInjector.create(
                DaggerAppComponent.this.logoutManagerProvider,
                DaggerAppComponent.this.userManagerProvider
                userProfileActivityPresenterProvider)
            );
        }

        @Override
        public UserProfileActivity inject(UserProfileActivity activity) {
            userProfileActivityMembersInjector.injectMembers(activity);
            return activity;
        }
    }
}
```

14-34行可以看出每个来自AppComponent的依赖请求都需要有能创建Provider的方法。为什么？因为依赖component只能访问基component通过public 接口暴露出来的对象。

​	这对我们来说以为着什么呢--各位Android开发者们？每个用这种方法产生的依赖都会让我们离65K methods越来越近。所以我们决定讲我们所有的@Compoennt依赖迁移至@SubComponent。结果这个操作让我们减少了5000个方法。

	### 快去分析你的Apk###

​	在文章的结尾有必要提一句，从As2.2开始，他们提供了一个分析apk文件的方式，***Build->Analyze Apk***….	![看看Dagger生成的内部类](https://cdn-images-2.medium.com/max/800/1*LX37WOAZPGsx9W73P879zg.png)

这个工具可以让我们看某个组件的大小，和对我们最有用的获取.dex文件的基本信息（类和方法数量），这是我们依赖注入优化的出发点。

​	这就是今天的全部啦~我们将分享更多使用DI的经验，谢谢阅读！！！

​	关注作者[Twitter](https://twitter.com/azimolabs) 和 [Github](https://github.com/azimolabs)



