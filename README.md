# 原理

懒加载的原理其实挺简单的, 最主要的就是利用fragment中的`setUserVisibleHint(boolean isVisibleToUser)`方法中传进来的那个`isVisibleToUser`这个参数, 这个参数的字面意思是表示当前fragment是否对用户可见.注意fragment还有一个`getUserVisibleHint()`的方法, 这个方法在我看来其实没什么用, 因为我试过打印这个方法的返回值, 返回为`true`并不能保证用户切换到了当前fragment.

# 重写setUserVisibleHint()

首先定义一个基类, BaseLazyFragment
重写`setUserVisibleHint()`这个方法

```
    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        Log.d(TAG, "setUserVisibleHint, isVisibleToUser = " + isVisibleToUser);
        super.setUserVisibleHint(isVisibleToUser);
        // 如果还没有加载过数据 && 用户切换到了这个fragment
        // 那就开始加载数据
        if (!mHaveLoadData && isVisibleToUser) {
            loadDataStart();
            mHaveLoadData = true;
        }
    }

```

用一个布尔变量`mHaveLoadData`来表示该fragment是否加载过数据. 如果没有加载过数据, 并且`isVisibleToUser`为`true`(表示用户切换到了这个fragment), 那就开始加载数据, 然后标记该fragment已经加载过数据

# loadDataStart()

加载数据这个方法是基类里的一个抽象方法, 需要子类来重写, 因为具体的加载过程是需要子类自己来实现的.
写到这里我突然想到了那个抽象类和接口有什么区别的面试题.
在这里的情景的话, 抽象类就是帮子类统一处理了一些逻辑, 比如判断什么时候需要进行加载数据, 这是由父类帮我们做好的, 子类就不需要再写重复的代码了, 而具体的请求过程是由子类自己去实现的.抽象类在这里的作用就是将重复的逻辑统一处理.
而接口的作用, 就我自己来说用的最多的就是使用一个接口类型的变量来引用一个对象, 这样就不用去关心这个对象具体是什么, 而我们只要知道这个对象中一定有接口中的方法, 到时候我们就能调用这个对象的方法, 虽然我们并不知道方法中的具体逻辑.
扯远了.
现在我们写一个子类继承这个基类fragment

```
    @Override
    public void loadDataStart() {
        Log.d(TAG, "loadDataStart");
        // 模拟请求数据
        mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                mData = "这是加载下来的数据";
                // 一旦获取到数据, 就应该立刻标记数据加载完成
                mLoadDataFinished = true;
                if (mViewInflateFinished) {
                    mTextView.setVisibility(View.VISIBLE);
                    mTextView.setText(mData);
                    mTextView.setText("这是改变后的数据");
                    mPb.setVisibility(View.GONE);
                }
            }
        }, 3000);
    }

```

在具体的fragment中模拟请求数据, 在请求完成后将`mLoadDataFinished`这个变量置为true, 这个字段是继承自基类fragment的. 然后再判断是否布局加载和找控件已经完成, 防止将数据设置到控件上的时候出现控件空指针的错误.
那么我们是在那里将mViewInflateFinished置为true的呢?在去看基类

# onCreateView()

```
    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
            @Nullable Bundle savedInstanceState) {
        Log.d(TAG, "onCreateView");
        if (mRootView != null) {
            return mRootView;
        }
        mRootView = initRootView(inflater, container, savedInstanceState);
        findViewById(mRootView);
        mViewInflateFinished = true;
        return mRootView;
    }

```

我们回到基类看`onCeateView()`方法, 在这里将布局用一个`mRootView`的全局变量储存起来是因为当viewpager中的fragment比较多的时候, 切换到别的fragment会导致回调`onDestroyView()`方法, 再切回来的时候导致`onCreateView()`和`onViewCreate()`方法又被调用, 为了防止fragment重新从layout文件中加载布局导致之前设置到控件上的变量和状态丢失, 在布局初次加载完成之后用`mRootView`这个变量储存起来, 当这个变量不为null时就直接复用这个布局就好了.
`initRootView()`是初次从layout文件加载布局的方法, 是一个抽象方法, 由子类具体去实现, 返回的View表示fragment的布局.
在初次加载布局完成之后就是找控件的findViewById()方法了, 这也是个抽象方法, 需要子类自己去实现.
在`findViewById()`完成之后就将`mViewInflateFinished`置为`true`, 表示加载布局和找控件完成.

# findViewById()

我们再去看子类具体实现的findViewById()方法.

```
    @Override
    protected void findViewById(View view) {
        mTextView = view.findViewById(R.id.section_label);
        mPb = view.findViewById(R.id.pb);
        if (mLoadDataFinished) { // 一般情况下这时候数据请求都还没完成, 所以不会进这个if
            mTextView.setVisibility(View.VISIBLE);
            mTextView.setText(mData);
            mPb.setVisibility(View.GONE);
        }
    }

```

在这个方法里, 如果找控件完成之后, 我们立刻将数据设置到控件上.但是在这之前我们还是需要判断一下是否数据已经加载完成, 如果数据已经加载完成, 那就将数据设置到控件上.这个`mLoadDataFinished`标志位在上面已经提过.

# initRootView

这个方法其实没什么好说的

```
    @Override
    protected View initRootView(LayoutInflater inflater, ViewGroup container,
            Bundle savedInstanceState) {
        Log.d(TAG, "initRootView");
        return inflater.inflate(R.layout.fragment_tab, container, false);
    }

```

# 完整代码

再来贴一下基类和子类的完整代码
BaseLazyFragment

```
public abstract class BaseLazyFragment extends Fragment {

    public final String TAG = getClass().getSimpleName();

    public boolean mHaveLoadData; // 表示是否已经请求过数据

    public boolean mLoadDataFinished; // 表示数据是否已经请求完毕
    private View mRootView;

    // 表示开始加载数据, 但不表示数据加载已经完成
    public abstract void loadDataStart();

    // 表示找控件完成, 给控件们设置数据不会报空指针了
    public boolean mViewInflateFinished;

    @Override
    public void onAttach(Context context) {
        Log.d(TAG, "onAttach");
        super.onAttach(context);
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Log.d(TAG, "onCreate");
    }

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        Log.d(TAG, "onActivityCreated");
        super.onActivityCreated(savedInstanceState);
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
            @Nullable Bundle savedInstanceState) {
        Log.d(TAG, "onCreateView");
        if (mRootView != null) {
            return mRootView;
        }
        mRootView = initRootView(inflater, container, savedInstanceState);
        findViewById(mRootView);
        mViewInflateFinished = true;
        return mRootView;
    }

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        Log.d(TAG, "onViewCreated");
    }

    protected abstract void findViewById(View view);

    protected abstract View initRootView(LayoutInflater inflater, ViewGroup container,
            Bundle savedInstanceState);

    @Override
    public void onDestroyView() {
        Log.d(TAG, "onDestroyView");
        super.onDestroyView();
    }

    @Override
    public void onDestroy() {
        Log.d(TAG, "onDestroy");
        super.onDestroy();
    }

    @Override
    public void onDetach() {
        Log.d(TAG, "onDetach");
        super.onDetach();
    }

    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        Log.d(TAG, "setUserVisibleHint, isVisibleToUser = " + isVisibleToUser);
        super.setUserVisibleHint(isVisibleToUser);
        // 如果还没有加载过数据 && 用户切换到了这个fragment
        // 那就开始加载数据
        if (!mHaveLoadData && isVisibleToUser) {
            loadDataStart();
            mHaveLoadData = true;
        }
    }

}

```

子类PlaceholderFragment0

```
public class PlaceholderFragment0 extends BaseLazyFragment {

    private TextView mTextView;
    private ProgressBar mPb;
    private Handler mHandler = new Handler();
    private String mData;

    public PlaceholderFragment0() {
    }

    /**
     * Returns a new instance of this fragment for the given section
     * number.
     */
    public static PlaceholderFragment0 newInstance() {
        PlaceholderFragment0 fragment = new PlaceholderFragment0();
        return fragment;
    }

    @Override
    public void loadDataStart() {
        Log.d(TAG, "loadDataStart");
        // 模拟请求数据
        mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                mData = "这是加载下来的数据";
                // 一旦获取到数据, 就应该立刻标记数据加载完成
                mLoadDataFinished = true;
                if (mViewInflateFinished) { // mViewInflateFinished一般都是true
                    mTextView.setVisibility(View.VISIBLE);
                    mTextView.setText(mData);
                    mTextView.setText("这是改变后的数据");
                    mPb.setVisibility(View.GONE);
                }
            }
        }, 3000);
    }

    @Override
    protected View initRootView(LayoutInflater inflater, ViewGroup container,
            Bundle savedInstanceState) {
        Log.d(TAG, "initRootView");
        return inflater.inflate(R.layout.fragment_tab, container, false);
    }

    @Override
    protected void findViewById(View view) {
        mTextView = view.findViewById(R.id.section_label);
        mPb = view.findViewById(R.id.pb);
        if (mLoadDataFinished) { // 一般情况下这时候数据请求都还没完成, 所以不会进这个if
            mTextView.setVisibility(View.VISIBLE);
            mTextView.setText(mData);
            mPb.setVisibility(View.GONE);
        }
    }

}

```

# 完结
