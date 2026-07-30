ViewModel是MVVM项目架构中的“VM”。

项目架构的意义，是为了提高项目代码的可读性，易于后期功能拓展和维护。

在Android项目中，数据的请求和处理不直接写在View中，View一般只进行数据的呈现和UI交互。

在MVC、MVP、MVVM中，M都承载了数据请求与处理的功能，V即视图，一般指Activity、Fragment等。这三种架构模式的区别：

- 在MVC中，View创建Controller，Controller调用Model，Model又和View互相调用。环环引用，如果没有做好回收管理，存在内存泄漏风险，耦合度较高。

- 在MVP中，去除了View和Model的直接调用，View和Model的交互都通过Persenter来进行，这样降低了耦合度。Presenter需要写很多的接口来实现Model和View的交互，如果业务复杂，Presenter会非常臃肿。

- MVVM是MVP的升级版，Model和View同样是通过ViewModel来进行交互。ViewModel是一个可观察的对象，当其属性发生变化时，会自动更新视图。

## ViewModel的简单使用

```java
public abstract class BaseActivity<M extends BaseViewModel> extends AppCompatActivity {
    protected String TAG;
    protected M mViewModel;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        TAG=this.getClass().getSimpleName();
        initViewModel();
        mViewModel.onCreate(this);
    }

    protected void initViewModel() {
        Class<M> viewModel = createViewModelClass();
        if (viewModel == null) {
            return;
        }
        mViewModel = new ViewModelManager<M>().createModel(mViewModel, this);
    }
}
```

## 多个Activity共享ViewModel

使用场景：当多个Activity有较多的业务交叉，或者是Activity与Fragment之间的数据交互，可以直接使用一个ViewModel来进行数据交互。

`ViewModelStoreOwner`是一个ViewModel存储器，可以用它来实现单例ViewModel。

```java
public class AfterSalesViewModelOwner {
    private static ViewModelStoreOwner viewModelStoreOwner;

    public static ViewModelStoreOwner getViewModelStoreOwner() {
        return viewModelStoreOwner;
    }

    public static void setViewModelStoreOwner(ViewModelStoreOwner viewModelStoreOwner) {
        AfterSalesViewModelOwner.viewModelStoreOwner = viewModelStoreOwner;
    }
}
```

在首个使用到该ViewModel的View中初始化ViewModelStoreOwner

```java
protected void initViewModel() {
//初始化ViewModelStoreOwner
        AfterSalesViewModelOwner.setViewModelStoreOwner(this);
//在后续使用中，通过ViewModelStoreOwner创建ViewModelProvider，通过key获取特定ViewModel
        mViewModel = new ViewModelProvider(AfterSalesViewModelOwner.getViewModelStoreOwner())
                .get("AfterSalesViewModel", AfterSalesViewModel.class);
    }
```
