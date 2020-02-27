---
title: Jetpack中的ViewModel
date: 2019-01-05 11:36:18
categories: Android
tags: Jetpack
---

### 前言

首先祝大家在新的一年中，身体健康，心想事成。

算了下，去年技术类博客写了 15 篇，看下数量还算比较满意，可是基本都是前半年写的，那时候刚刚下定决心要好好写博客，后半年因为工作上还有自己偷懒心理，基本都没怎么写了，真是惭愧。

不过经过去年写了这些文章，慢慢也学习到了一些写技术文章上的技巧，希望今年能方得始终，提升文章的质量。

### 关于 Jetpack

相信已经有不少人对 Google 推出的 Jetpack 系列组件都有所耳闻，现在网上已经有不少的分析文章了，涵盖了用法和源码解析，所以我就不重复造轮子，在这个系列文章中，不会涉及使用教程等，只写一些个人的使用体会，也当作给自己做笔记。

### 关于本文

这篇文章主要讲 Jetpack 中的 ViewModel

### ViewModel

> The [`ViewModel`](https://developer.android.com/reference/android/arch/lifecycle/ViewModel.html) class is designed to store and manage UI-related data in a lifecycle conscious way. 

简而言之，就是在生命周期中管理数据。说到生命周期，我们都知道 Android 中 `Activity` 和 `Fragment` 都有各自对应的生命周期，比如 `Activity`，它的生命周期如下图：

![activity_lifecycle](https://leo-doc-img.oss-cn-hangzhou.aliyuncs.com/doc-img/activity_lifecycle.png?x-oss-process=style/doc-img)

通常我们会在 `onCreate()` 初始化数据，在 `onResume()` 中展示数据，`onDestroy()` 中释放数据，类似下面这些伪代码：

``` java
onCreate() {
    initData();
}

onResume() {
    displayData()
}

onDestory() {
    recycleData();
}
```

如果没有在 `onDestory()` 中及时释放某些资源，可能还会导致内存泄漏，这是第一个问题。

第二个问题，Android 系统可能会在内存不足的情况下，回收了 `Activity`，导致 `Activity` 重建时数据会丢失，对于这种情况，Android 提供了 `onSaveInstanceState` 中保存数据，在 `onRestoreInstanceState` 和 `onCreate` 中获取。类似下面这些伪代码：

``` java
onSaveInstanceState(Bundle outState) {
 	outState.putData();   
}

onRestoreInstanceState(Bundle outState) {
    outState.getData();
}

onCreate(Bundle savedInstanceState) {
    if (savedInstanceState != null) {
        savedInstanceState.getData();
    }
}
```

这种方式除了重复的胶水代码以外，还存在 `Bundle` 存储只适用于支持序列化（Serializable 和 Parcelable）的少量数据，当然除了使用 Android SDK 提供的这种方案以外，我们也可以自己实现类似的方案，类似以下伪代码：

``` java
// Data Repository
Map<String,Data> map;

getData(String id) {
    if (map.get(id) == null){
        Data data = createData();
        map.put(id,data);
    }
    return map.get(id);
}

removeData(String id) {
    map.removeByKey(id);
}

// Activity
onCreate() {
    getData(this);
}

onDestory() {
    if (!isChangingConfigurations){
            removeData(this);
    }
}
```

看了上面解决思路后，我们再来看看 Google 提供的 `ViewModel`，它是如何解决上面提到的两个问题的。首先看下，`ViewModel` 的典型用法，来源于官方文档：

首先定义一个 `MyViewModel` 继承于 `ViewModel`，这里使用 `LiveData` 作为数据源，这里我们只需要知道 `LiveData` 和 `RxJava` 是差不多的东西就可以了

``` java
public class MyViewModel extends ViewModel {
    private MutableLiveData<List<User>> users;
    public LiveData<List<User>> getUsers() {
        if (users == null) {
            users = new MutableLiveData<List<User>>();
            loadUsers();
        }
        return users;
    }

    private void loadUsers() {
        // Do an asynchronous operation to fetch users.
    }
}
```

定义好 `ViewModel` 之后，我们在 `Activity` 中使用它：

``` java
public class MyActivity extends AppCompatActivity {
    public void onCreate(Bundle savedInstanceState) {
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.

        MyViewModel model = ViewModelProviders.of(this).get(MyViewModel.class);
        model.getUsers().observe(this, users -> {
            // update UI
        });
    }
}
```

我们先看下 `ViewModelProviders.of` 的函数签名：

``` java
public static ViewModelProvider of(@NonNull Fragment fragment);
public static ViewModelProvider of(@NonNull FragmentActivity activity);
public static ViewModelProvider of(@NonNull Fragment fragment, @Nullable Factory factory);
public static ViewModelProvider of(@NonNull FragmentActivity activity,
            @Nullable Factory factory);
```

需要将当前 `Activity` 或 `Fragment` 作为参数，这也是 `ViewModel` 将数据与生命周期结合起来的地方。那具体也是如何实现的呢？

``` java
@NonNull                                                                             
@MainThread                                                                          
public static ViewModelProvider of(@NonNull FragmentActivity activity,               
        @Nullable Factory factory) {                                                 
    Application application = checkApplication(activity);                            
    if (factory == null) {                                                           
        factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
    }                                                                                
    return new ViewModelProvider(activity.getViewModelStore(), factory);             
}                                                                                    
```

`factory` 顾名思义就是 `ViewModel` 的工厂类，这里默认使用 `AndroidViewModelFactory` 最后返回 `ViewModelProvider` 实例

``` java
// ViewModelProvider
@NonNull                                                                                 
@MainThread                                                                              
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {  
    ViewModel viewModel = mViewModelStore.get(key);                                      
                                                                                         
    if (modelClass.isInstance(viewModel)) {                                              
        //noinspection unchecked                                                         
        return (T) viewModel;                                                            
    } else {                                                                             
        //noinspection StatementWithEmptyBody                                            
        if (viewModel != null) {                                                         
            // TODO: log a warning.                                                      
        }                                                                                
    }                                                                                    
                                                                                         
    viewModel = mFactory.create(modelClass);                                             
    mViewModelStore.put(key, viewModel);                                                 
    //noinspection unchecked                                                             
    return (T) viewModel;                                                                
}

// AndroidViewModelFactory
@NonNull                                                                                   
@Override                                                                                  
public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {                      
    if (AndroidViewModel.class.isAssignableFrom(modelClass)) {                             
        //noinspection TryWithIdenticalCatches                                             
        try {                                                                              
            return modelClass.getConstructor(Application.class).newInstance(mApplication); 
        } catch (NoSuchMethodException e) {                                                
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);   
        } catch (IllegalAccessException e) {                                               
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);   
        } catch (InstantiationException e) {                                               
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);   
        } catch (InvocationTargetException e) {                                            
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);   
        }                                                                                  
    }                                                                                      
    return super.create(modelClass);                                                       
}                                                                                          
```

我们将以上两段代码合起来看，其实逻辑是比较清晰的：

![ViewModelProvider#create](https://leo-doc-img.oss-cn-hangzhou.aliyuncs.com/doc-img/viewmodel/ViewModelProvider%23create.png?x-oss-process=style/doc-img)

`Factor` 的实现可以通过反射来实现，比如默认的 `AndroidViewModelFactory` 会优先调用使用 `Application` 作为参数的构造方法，来创建实例。所以，如果自定义的 `ViewModel` 构造方法有其他参数，就需要自定义 `Factor`

而 `ViewModelStore` 则是 `Activity` 重建时还能拥有之前数据的保障。

``` java
private final HashMap<String, ViewModel> mMap = new HashMap<>();                  
                                                                                  
final void put(String key, ViewModel viewModel) {                                 
    ViewModel oldViewModel = mMap.put(key, viewModel);                            
    if (oldViewModel != null) {                                                   
        oldViewModel.onCleared();                                                 
    }                                                                             
}                                                                                 
                                                                                  
final ViewModel get(String key) {                                                 
    return mMap.get(key);                                                         
}                                                                                 
                                                                                  
/**                                                                               
 *  Clears internal storage and notifies ViewModels that they are no longer used. 
 */                                                                               
public final void clear() {                                                       
    for (ViewModel vm : mMap.values()) {                                          
        vm.onCleared();                                                           
    }                                                                             
    mMap.clear();                                                                 
}                                                                                 
```

`ViewModelStore` 的源码很短，可以看到其实就是使用 `HashMap` 作为数据载体。既然有使用，那就需要清理操作，可以看到有个 `clear` 它会清除 `HashMap` 中的缓存数据。我们首先看下 `ViewModelProvider` 中的 `mViewModelStore` 是在哪里赋值的：

``` java
public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) { 
    this(owner.getViewModelStore(), factory);                                            
}                                                                                        
                                                                                                                                                                             
public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {      
    mFactory = factory;                                                                  
    this.mViewModelStore = store;                                                        
}                                                                                        
```

通过 `ViewModelStoreOwner.getViewModelStore` 获取 `ViewModelStore` 实例对象，而 `ViewModelStoreOwner` 实际就是我们调用 `ViewModelProviders.of` 中传递的 `FragmentActivity` 和 `Fragment`

> `FragmentActivity` 中会通过 `onRetainNonConfigurationInstance` 和 `getLastNonConfigurationInstance` 去保持 `Activity` 因为屏幕旋转等配置发生改变而导致重建时，数据的唯一性

而 `Fragment` 和 `FragmentActivity` 会在 `onDestory` 时判断是否需要调用 `ViewModelStore.clear`

``` java
@Override                                                            
protected void onDestroy() {                                         
    super.onDestroy();                                               
                                                                     
    if (mViewModelStore != null && !isChangingConfigurations()) {    
        mViewModelStore.clear();                                     
    }                                                                
                                                                     
    mFragments.dispatchDestroy();                                    
}                                                                    
```

讲完 `ViewModelStore` 的缓存功能之后，我们再来看下，`ViewModelProvider.of` 不同签名的方法：

``` java
@NonNull                                                         
@MainThread                                                      
public static ViewModelProvider of(@NonNull Fragment fragment) { 
    return of(fragment, null);                                   
}

@NonNull                                                                    
@MainThread                                                                 
public static ViewModelProvider of(@NonNull FragmentActivity activity) {    
    return of(activity, null);                                              
}                                                                           
```

这里需要注意的是，`ViewModel` 在不同作用域下的实例，首先，`FragmentActivity` 和 `Fragment` 都实现了 `ViewModelStoreOwner` 即它们都有各自的 `ViewModelStore` 简单来说，如果想在 `Fragment` 中获取依附的 `Activity` 的 `ViewModel` 实例，那需要使用 `of(FragmentActivity)` 的方法，这也是一种 `Activity` 和 `Fragment` 之间通信的好方式。

### 总结

现在我们来比较下自定义实现的方案和 `ViewModel` ，可以发现其实核心思想是共通，首先我们需要一个保存于 `Activity` 和 `Fragment` 生命周期之外的存储空间，在 `ViewModel` 中是 `ViewModelStore`，其次我们需要在 `Activity` 和 `Fragment` 对应的生命周期中，去初始化和清理这个 `ViewModelStore`