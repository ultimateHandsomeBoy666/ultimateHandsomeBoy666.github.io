项目里需要App端不断地从服务器获取数据，实时生成图表。图表控件使用的是[MPAndroidChart](https://github.com/PhilJay/MPAndroidChart)。自己写了些实时更新折线图的demo，数据是线程随机生成的，不是后台数据。
######1、Message配合Handler实现
效果如下
![Message配合Handler实现.gif](Android%E5%AE%9E%E7%8E%B0%E5%9B%BE%E8%A1%A8%E5%AE%9E%E6%97%B6%E6%9B%B4%E6%96%B0.assets/strip.gif)
在MainActivity中创建一个产生随机数据的线程，每产生一个数据发送一个Message，Handler收到Message之后更新折线图。

MainActivity代码如下：
```Java
public class MainActivity extends AppCompatActivity {
    private static final int TAG = 1;//Message的what标识
    private TextView mTextView;
    private Button mStartButton;

    private LineChart mLineChart;
    private Data[] mDatas;
    private List<Entry> mEntries = new ArrayList<>();

    private Thread mThread;
    private Handler mHandler;
    private Random mRandom;

    private StringBuilder mStringBuilder;
    private int mEndIndex;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mTextView = findViewById(R.id.test_txt);
        mStartButton = findViewById(R.id.start_button);
        mLineChart = findViewById(R.id.line_chart);

        mRandom = new Random();
        mStringBuilder = new StringBuilder("现在Y轴数字是0哦");
        mEndIndex = 1;

        //先创建5个Data数据
        mDatas = new Data[]{new Data(1,5),new Data(2,8),
               new Data(3,10),new Data(4,13),new Data(5,16)};
        for (Data data :mDatas){
            mEntries.add(new Entry(data.getValueX(),data.getValueY()));
        }
        LineDataSet dataSet = new LineDataSet(mEntries,"number");
        LineData lineData = new LineData(dataSet);
        mLineChart.setData(lineData);
        mLineChart.invalidate();


        mHandler = new Handler(){
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                if (msg.what==TAG){
                    updateTxt(msg);
                    updateChart(msg);
                }
            }
        };

        mThread = new Thread(new Runnable() {
            @Override
            public void run() {
                int corrX = 6;//已经有了五个数据，下一个数据的x坐标从6开始
                while (true){
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    int corrY = mRandom.nextInt(20) + 5;
                    Message message = Message.obtain();
                    message.arg1 = corrY;
                    message.arg2 = corrX;
                    message.what = TAG;
                    mHandler.sendMessage(message);
                    corrX += 1;
                }
            }
        });

        mStartButton.setOnClickListener((View v) -> mThread.start());

    }

    //更新SpannableString类型的文本需要用该函数判断更新数字的位数
    private int endIndex(int i){
        int index = 0;
        while (i!=0){
            i = i/10;
            index += 1;
        }
        return index;
    }

    //更新显示当前值的TextView
    private void updateTxt(Message msg){
        mStringBuilder.replace(7,7 + mEndIndex, msg.arg1 + "");//将原来的数字替换

        SpannableStringBuilder spannableStringBuilder = new SpannableStringBuilder(mStringBuilder);
        ForegroundColorSpan foregroundColorSpan = new ForegroundColorSpan(Color.BLUE);
        RelativeSizeSpan relativeSizeSpan = new RelativeSizeSpan(1.5f);

        mEndIndex = endIndex(msg.arg1);//新的y值的位数

        spannableStringBuilder.setSpan(foregroundColorSpan,7,7 + mEndIndex, Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
        spannableStringBuilder.setSpan(relativeSizeSpan,7,7 + mEndIndex, Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
        mTextView.setText(spannableStringBuilder);

    }

    //刷新折线图
    private void updateChart(Message msg){
        mEntries.add(new Entry(msg.arg2,msg.arg1));
        LineDataSet dataSet = new LineDataSet(mEntries,"number");
        LineData lineData = new LineData(dataSet);
        mLineChart.setData(lineData);
        mLineChart.invalidate();
    }

}
```
Data类如下
```Java
public class Data {
    private int valueX;
    private int valueY;

    public Data(int x,int y){
        this.valueX = x;
        this.valueY = y;
    }

    public int getValueX() {
        return valueX;
    }

    public void setValueX(int valueX) {
        this.valueX = valueX;
    }

    public int getValueY() {
        return valueY;
    }

    public void setValueY(int valueY) {
        this.valueY = valueY;
    }
}
```
######2、RxJava实现
Rxjava在处理复杂的多线程事件逻辑时比Handler/Async等要简单易用可靠。用来写这个demo算是大炮打蚊子，纯当练手了。

MainActivity
```
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";
    private LineChart mLineChart;
    private TextView mTextView;
    private Button mStartButton;
    private List<Entry> mEntryList = new ArrayList<>();

    private Random mRandom;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mLineChart = findViewById(R.id.line_chart);
        mStartButton = findViewById(R.id.start_button);
        mTextView = findViewById(R.id.value_txt);

        mStartButton.setOnClickListener((View v) -> intervalObservable());
    }

    private void intervalObservable() {
        mRandom = new Random();
        Observable.interval(1000, 1000, TimeUnit.MILLISECONDS)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(Long aLong) throws Exception {
                        Log.e("intervalObservable", Thread.currentThread().getName());
                        Long x = aLong;
                        int y = mRandom.nextInt(10);
                        Data data = new Data(x, y);
                        x++;

                        mEntryList.add(new Entry(data.getX(), data.getY()));
                        LineDataSet dataSet = new LineDataSet(mEntryList, "label");
                        LineData lineData = new LineData(dataSet);
                        mLineChart.setData(lineData);
                        mLineChart.invalidate();
                        mTextView.setText("当前y值为" + y);
                    }
                });
    }
}
```
写的时候发现如果直接用Observable.create()生成数据的话，速度太快，MPAndroidChart刷新不过来，一片空白。所以改用Observable.interval()，每个1秒生成一个，但是这个函数只能返回一个Observable<Long>的对象，每次发射的都是Long类型的数据，所以把Data类型中的x值改成了Long类型。
还要注意的是，Observable.interval()默认订阅Schedulers.computation这个线程，如果有UI更新的话，需要在主线程中进行观察，即调用observeOn(AndroidSchedulers.mainThread())。
######但是我发现一个很神奇的事，MPAndroidChart可以在非UI线程中进行刷新。

难道只能用Observal.interval()吗?其实不是的，我发现只要使被观察者线程休眠一小段时间，就能让折线图刷新出来，代码如下
```
Observable.create(new ObservableOnSubscribe<Data>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Data> e) throws Exception {
                Long x = 0l;
                Random random = new Random();
                while(x < 1000) {
                    int y = random.nextInt(10);
                    Data data = new Data(x, y);
                    e.onNext(data);
                    x++;
                    Thread.sleep(1000);//休眠1秒
                }
                Log.e(TAG, "Observable thread is : " + Thread.currentThread().getName());
            }
        }).subscribeOn(Schedulers.newThread())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(data -> {
                Log.i("onNext(Data data)", data.toString());
                Log.e(TAG, "Observer thread is :" + Thread.currentThread().getName());
                mEntryList.add(new Entry(data.getX(), data.getY()));
                LineDataSet dataSet = new LineDataSet(mEntryList, "label");
                LineData lineData = new LineData(dataSet);
                mLineChart.setData(lineData);
                mLineChart.invalidate();
                mTextView.setText("当前值为" + data.getY());
            });
```
其实在订阅者线程中休眠也可以正常接收到数据，但订阅者线程一般是UI线程，休眠的话，UI就不会更新了。
另外，被观察者的线程没有休眠的话，即使被观察者数据发送的很快，订阅者在onNext()即使进行了线程休眠，数据也能全部接收到，不会出现事件丢失的情况，这一点让我比较疑惑。
如果被观察者的线程调用了Thread.sleep(1)，而观察者在onNext()中调用了Thread.sleep(1000)，那么会出现上下游事件处理速率不匹配，事件丢失，OOM等情况。
这个时候就要用支持背压的Flowable了。
其实用Flowable同样可以实现折线图更新，代码如下：
```
Flowable.create(new FlowableOnSubscribe<Data>() {
            @Override
            public void subscribe(FlowableEmitter<Data> e) throws Exception{
                Long x = 0l;
                Random random = new Random();
                while(x < 1000) {
                    int y = random.nextInt(10);
                    Data data = new Data(x, y);
                    e.onNext(data);
                    x++;
                    Thread.sleep(1000);
                }
                Log.e(TAG, "Observable thread is : " + Thread.currentThread().getName());
            }
        }, BackpressureStrategy.BUFFER).subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<Data>() {
                    Subscription mSubscription;
                    @Override
                    public void onSubscribe(Subscription s) {
                        mSubscription = s;
                        s.request(1);
                    }

                    @Override
                    public void onNext(Data data) {
                        Log.i("onNext(Data data)", data.toString());
                        Log.e(TAG, "Observer thread is :" + Thread.currentThread().getName());
                        mEntryList.add(new Entry(data.getX(), data.getY()));
                        LineDataSet dataSet = new LineDataSet(mEntryList, "label");
                        LineData lineData = new LineData(dataSet);
                        mLineChart.setData(lineData);
                        mLineChart.invalidate();
                        mTextView.setText("当前值为" + data.getY());

                        mSubscription.request(1);
                    }

                    @Override
                    public void onError(Throwable t) {

                    }

                    @Override
                    public void onComplete() {

                    }
                });
```


