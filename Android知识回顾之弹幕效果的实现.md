
从2015年初接触android到现在想想也有不短的一段时间了，这期间从毕业设计到跟朋友一起接外包，到进入公司从事android开发，手中大大小小也经历了不少android的项目，然而直到读了研，信心满满的出去找工作的时候才发现，短短半年在学校学习其他课程的时光，android的东西竟已忘却了大半，面对面试官的你项目中曾经遇到过哪些难题，讲讲你当时是怎么解决的？What？Android有很难的东西吗？百度Github上不都有吗<黑人问号脸>？天知道我们后面的面试是怎么进行下去的，，，好吧，这个暂且不提，毕竟今天的主题不是吐槽，而是要掀开本人博客生涯的第一篇，毕竟，有些东西只有记下来，才感觉有可能会是自己的<笑哭>...

OK ,烟鬼正传，今天主要是回顾下之前项目中用到的自定义view弹幕效果的实现。
###首先功能分析
我们需要实现一个自定义的view，实现如下几点功能：
- 弹幕中的每一条弹幕从右进从左出，背景透明
- 弹幕中的每一条弹幕包括用户头像和文字，进行图文混排
- 弹幕中的每一条弹幕从左边完全消失后，过段时间按照其在弹幕列表中的顺序循环出现
看到右进左出，我们本能的可能会想到用到view的滑动，动态的更新弹幕的translationX，但是由于每条弹幕的速度不一致，因此需要创建不同线程来管理不同弹幕的滑动，但是这显然是不明智的，因为弹幕的数量是不封顶的，用线程来管理耗费代价太大，因此只能另寻他法，于是只能从搞清自定义view的流程来入手，由于view的绘制主要有measure、layout、draw三大流程，其中measure确定view的测量宽高，layout确定view的最终宽高和四个顶点的位置，而draw则将view绘制到屏幕上。可以看出measure、layout对我们帮助不大，由此考虑能否一次将所有弹幕信息绘制在界面上，然后更新弹幕的坐标信息，继续调用view的draw使之重绘，从而形成在用户看来的滑动效果，事实证明是可行的。

###实现思路

由于每一个DanmuView 中包括很多条滚动的弹幕，因此把每条弹幕封装成一个DanmuItem，而DanmuView则为每个DanmuItem的容器和管理者，同时为了弹幕展示的更加美观，为每个DanmuView 纵向设置N条弹道，每条弹道设置最多m条弹幕，把从服务器端获取的弹幕消息依次加入消息队列，每隔一秒钟从队列中取出一条消息封装成弹幕信息，然后随机的选择一条弹道进行绘制在界面，然后通过随机生成的速度参数更改其下一次应该绘制在界面的坐标，进行重绘，直到此条弹幕消息已从屏幕左边滑出，把其从正在移动的弹幕队列中删除，重新添加进入消息队列，等待相应的时间重新被取出，以此实现弹幕的循环展示效果。

###具体实现
此次的实现主要包括三个文件DanmuView,DanmuItem,IDanmuItem。其中IDanmuItem为接口，方便后面的修改和扩充，里面主要封装了DanmuItem的共有属性和方法。

	public interface IDanmuItem {
		//通过DanmuView传来的canvas参数绘制自身
	    void doDraw(Canvas canvas);
		//支持对弹幕字体大小的设置
	    void setTextSize(int sizeInDip);
		//支持对弹幕字体颜色的设置
	    void setTextColor(int colorResId);
		//支持对弹幕的起始位置进行设置
	    void setStartPosition(int x, int y);
		//设置弹幕的速度因子，速度大小为basespeed*factor
	    void setSpeedFactor(float factor);
		//获取速度因子
	    float getSpeedFactor();
		//判断自身坐标是否已经完全滑出屏幕
	    boolean isOut();
		//判断自身会不会与正在滑动的同一弹道的目标item发生碰撞
	    boolean willHit(IDanmakuItem runningItem);
		//释放掉context
	    void release();
		//得到自身的宽度
	    int getWidth();
		//得到自身的高度
	    int getHeight();
		//得到自身现在的X坐标
	    int getCurrX();
		//得到自身现在的Y坐标
	    int getCurrY();
	}
DanmuItem则为IDanmuItem接口的具体实现，这里就不粘出全部代码了，只贴出关键代码。在DanmuItem中其实主要实现的就是图文混排以及文字换行在界面的绘制和提供自身会不会与正在滑动的同一弹道的目标item发生碰撞的判断，还有是否已滑出窗口的判断。下面依次粘出关键代码。
####图文混排
图文混排的实现主要靠SpannableString ,SpannableString其实和String一样，都是一种字符串类型，同样TextView也可以直接设置SpannableString作为显示文本，不同的是SpannableString可以通过使用其方法setSpan方法实现字符串各种形式风格的显示,重要的是可以指定设置的区间，也就是为字符串指定下标区间内的子字符串设置格式。

setSpan(Object what, int start, int end, int flags)方法需要用户输入四个参数，what表示设置的格式是什么，可以是前景色、背景色也可以是可点击的文本等等，start表示需要设置格式的子字符串的起始下标，同理end表示终了下标，flags属性就有意思了，共有四种属性：
Spanned.SPAN_INCLUSIVE_EXCLUSIVE 从起始下标到终了下标，包括起始下标
Spanned.SPAN_INCLUSIVE_INCLUSIVE 从起始下标到终了下标，同时包括起始下标和终了下标
Spanned.SPAN_EXCLUSIVE_EXCLUSIVE 从起始下标到终了下标，但都不包括起始下标和终了下标
Spanned.SPAN_EXCLUSIVE_INCLUSIVE 从起始下标到终了下标，包括终了下标

	SpannableString spannableString = new SpannableString(content);
    if (bitmap != null) {
	    //压缩bitmap
        Bitmap scaledBitmap = Bitmap.createScaledBitmap(bitmap, Dip2PxUtils.dip2px(context, 30), Dip2PxUtils.dip2px(context, 30), true);
        VerticalImageSpan imageSpan = new VerticalImageSpan(context, getRoundedCornerBitmap(scaledBitmap));
        spannableString.setSpan(imageSpan, 0, 1, Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
        }

详情请参考[用SpannableString打造绚丽多彩的文本显示效果](http://www.jianshu.com/p/84067ad289d2)

####文字换行
使用Canvas的drawText绘制文本是不会自动换行的，即使一个很长很长的字符串，drawText也只显示一行，超出部分被隐藏在屏幕之外。可以逐个计算每个字符的宽度，通过一定的算法将字符串分割成多个部分，然后分别调用drawText一部分一部分的显示， 但是这种显示效率会很低。
StaticLayout是android中处理文字换行的一个工具类，StaticLayout已经实现了文本绘制换行处理
	`需要指出的是这个layout是默认画在Canvas的(0,0)点的，如果需要调整位置只能在draw之前移Canvas的起始坐标canvas.translate(x,y);`

		TextPaint tp = new TextPaint();
        tp.setAntiAlias(true);
        tp.setColor(mTextColor);
        tp.setTextSize(mTextSize);
        strokePaint.setTextSize(mTextSize);

        mContentHeight = getFontHeight(tp);
        staticLayout = new StaticLayout(mContent, tp,
                            (int) Layout.getDesiredWidth(mContent, 0, mContent.length(), tp) + 1,
                            Layout.Alignment.ALIGN_NORMAL,
                            1.0f,
                            0.0f,
                            false);
        mContentWidth = staticLayout.getWidth();
        borderStaticLayout = new StaticLayout(mContent, strokePaint,
                (int) Layout.getDesiredWidth(mContent, 0, mContent.length(), tp) + 1,
                Layout.Alignment.ALIGN_NORMAL,
                1.0f,
                0.0f,
                false);
 borderStaticLayout 主要为字体描边所用，因为所要实现的字体是黑边框内白的字体。
 详情请参考[android staticlayout使用讲解](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/0915/1682.html)

####DanmuItem的绘制
	public void doDraw(Canvas canvas) {
        int canvasWidth = canvas.getWidth();
        int canvasHeight = canvas.getHeight();
		//判断屏幕是否切换至横屏了
        if (canvasWidth != this.mContainerWidth || canvasHeight != this.mContainerHeight) {//phone rotated !
            this.mContainerWidth = canvasWidth;
            this.mContainerHeight = canvasHeight;
        }

        canvas.save();
		//先移动画布，然后进行文字背景的绘制，在进行文字的绘制，实现文字描边效果
        canvas.translate(mCurrX,mCurrY);
        borderStaticLayout.draw(canvas);
        staticLayout.draw(canvas);
        canvas.restore();

        mCurrX = (int) (mCurrX - sBaseSpeed * mFactor);//更改下次绘制时的X坐标，向左移动
    }
####DanmuItem的碰撞检测
	public boolean willHit(IDanmaItem runningItem) {
		//如果正在移动的Item还未完全滑出屏幕右侧，肯定会碰撞
        if (runningItem.getWidth() + runningItem.getCurrX() > mContainerWidth) {
            return true;
        }
		//如果滑动的Item移速比自身快，不会碰撞
        if (runningItem.getSpeedFactor()>= mFactor) {
            return false;
        }
		//判断移动的Item右侧完全滑出屏幕左侧的时间内自身划过的距离，对比得出是否会碰撞
        float len1 = runningItem.getCurrX() + runningItem.getWidth();
        float t1 = len1 / (runningItem.getSpeedFactor() * DanmaItem.sBaseSpeed);
        float len2 = t1 * mFactor * DanmaItem.sBaseSpeed;
        if (len2 > len1) {
            return true;
        } else {
            return false;
        }
    }
####DanmuView 的实现

这是实现弹幕效果的核心逻辑控制层，首先，定义了一个R.styleable.DanmuView,用来动态的设置DanmuView的一些属性和便于管理

		<declare-styleable name="DanmakuView">
	        <attr name="max_row" format="integer" />
	        <attr name="pick_interval" format="integer" />
	        <attr name="max_running_per_row" format="integer" />
	        <attr name="show_debug" format="boolean" />
	        <attr name="start_Y_offset" format="float" />
	        <attr name="end_Y_offset" format="float" />
	    </declare-styleable>
通过DanmuView的三个参数的构造方法来获取布局文件中的属性设置

	public DanmuView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mContext = context;
        TypedArray a = mContext.obtainStyledAttributes(attrs, R.styleable.DanmuView, 0, 0);
        mMaxRow = a.getInteger(R.styleable.DanmuView_max_row, 1);
        mPickItemInterval = a.getInteger(R.styleable.DanmuView_pick_interval, 1000);
        mMaxRunningPerRow = a.getInteger(R.styleable.DanmuView_max_running_per_row, 1);
        mShowDebug = a.getBoolean(R.styleable.DanmuView_show_debug, false);
        mStartYOffset = a.getFloat(R.styleable.DanmuView_start_Y_offset, 0.1f);
        mEndYOffset = a.getFloat(R.styleable.DanmuView_end_Y_offset, 0.9f);
        a.recycle();
        checkYOffset(mStartYOffset, mEndYOffset);
        init();
    }

其次，把DanmuView在Y轴分为mMaxRow个弹道，生成一个HashMap<Integer, ArrayList<IDanmakuItem>> mChannelMap来保存所有要显示在弹幕上的弹幕信息，每一个弹道通过一个ArrayList<IDanmuItem>来保存当前弹道中的弹幕信息，然后创建一个先进先出的waiting队列LinkedList<IDanmakuItem> mWaitingItems用来保存即将要展示在弹幕上的DanmuItem，弹幕显示的三种状态，STATUS_RUNNING、STATUS_PAUSE、STATUS_STOP，最后还有一些具体设置属性

	private int mMaxRow = 1; //最多几条弹道
    private int mPickItemInterval = 1000;//每隔多长时间取出一条弹幕来播放.
    private int mMaxRunningPerRow = 1; //每条弹道上最多同时有几个弹幕在屏幕上运行
    private float mStartYOffset = 0.1f; //第一个弹道在Y轴上的偏移占整个View的百分比
    private float mEndYOffset = 0.9f;//最后一个弹道在Y轴上的偏移占整个View的百分比
    private int[] mChannelY; //每条弹道的Y坐标
    private static final float mPartition = 0.95f; //仅View顶部的部分可以播放弹幕百分比

下面则为具体的实现代码

	//由构造函数调用的初始化，初始化背景透明和弹道Map以及每个弹道Y坐标mChannelY的初始化
	private void init() {
        setBackgroundColor(Color.TRANSPARENT);
        setDrawingCacheBackgroundColor(Color.TRANSPARENT);
        calculation();
    }

    private void calculation() {
        if (mShowDebug) {	//调试信息
            fpsPaint = new TextPaint(Paint.ANTI_ALIAS_FLAG);
            fpsPaint.setColor(Color.YELLOW);
            fpsPaint.setTextSize(20);
            times = new LinkedList<>();
            lines = new LinkedList<>();
        }
        initChannelMap();
        initChannelY();
    }
	//初始化Map和每个弹道对应的ArrayList
 	private void initChannelMap() {
        mChannelMap = new HashMap<>(mMaxRow);
        for (int i = 0; i < mMaxRow; i++) {
            ArrayList<IDanmakuItem> runningRow = new ArrayList<IDanmakuItem>(mMaxRunningPerRow);
            mChannelMap.put(i, runningRow);
        }
    }
	
    private void initChannelY() {
        if (mChannelY == null) {
            mChannelY = new int[mMaxRow];
        }
		//计算每条弹道中具体DanmuItem需要显示的Y坐标
        float rowHeight = getHeight() * (mEndYOffset - mStartYOffset) / mMaxRow;
        float baseOffset = getHeight() * mStartYOffset;
        for (int i = 0; i < mMaxRow; i++) {
            mChannelY[i] = (int) (baseOffset + rowHeight * (i + 1) - rowHeight * 3 / 4);//每一行空间顶部留1/4,剩下3/4显示文字

        }
		//主要是一些调试信息，don't care
        if (mShowDebug) {
            lines.add(baseOffset);
            for (int i = 0; i < mMaxRow; i++) {
                lines.add(baseOffset + rowHeight * (i + 1));
            }
        }
    }
	//设置是否无限循环播放弹幕
	public void setInfinitScroll(boolean infinit) {
        this.infinit = infinit;
    }
	 @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (status == STATUS_RUNNING) {//判断当前状态，运行状态则进行DanmuItem的绘制
            try {
                canvas.drawColor(getResources().getColor(android.R.color.transparent));

                //先绘制正在播放的弹幕
                for (int i = 0; i < mChannelMap.size(); i++) {
                    ArrayList<IDanmakuItem> list = mChannelMap.get(i);
                    for (Iterator<IDanmakuItem> it = list.iterator(); it.hasNext(); ) {
                        DanmakuItem item = (DanmakuItem) it.next();
						//判断如果Item已滑出View，从ArrayList中删除，
						//如果循环播放设置为true，则加入waiting队列等待再次滑出
						//如果没有滑出View则调用其自身的绘制方法，展示在view上
                        if (item.isOut()) {
                            it.remove();
                            if (infinit) {
                                mWaitingItems.add(item);
                            }
                        } else {
                            item.doDraw(canvas);
                        }
                    }
                }
                //检查是否需要加载播放下一个弹幕
                if (System.currentTimeMillis() - previousTime > mPickItemInterval) {
                    previousTime = System.currentTimeMillis();
                    IDanmakuItem di = mWaitingItems.pollFirst();
                    if (di != null) {
						//得到一个可播放的弹道index，即不会发生碰撞的弹道
                        int indexY = findVacant(di);
                        if (indexY >= 0) {//判断index是否有效，有效则滑出
                            di.setStartPosition(canvas.getWidth() - 2, mChannelY[indexY]);
                            di.doDraw(canvas);
                            mChannelMap.get(indexY).add(di);//不要忘记加入正运行的维护的列表中
                        } else {
                            addItemToHead(di);//找不到可以播放的弹道,则把它放回列表中
                        }
                    } else {
                        //no item 弹幕播放完毕,
                    }
                }
				//调试信息
                if (mShowDebug) {
                    int fps = (int) fps();
                    canvas.drawText("FPS:" + fps, 5f, 20f, fpsPaint);
                    for (float yp : lines) {
                        canvas.drawLine(0f, yp, getWidth(), yp, fpsPaint);
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
            invalidate();//通过此函数进行重绘，实现Item移动效果

        } else {//暂停或停止,隐藏弹幕内容
            canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);//清空画布
        }
    }
	//计算可以加入并展示的弹道index
	private int findVacant(IDanmakuItem item) {
        try {//遍历判断ArrayList如果为空则直接可以加入
            for (int i = 0; i < mMaxRow; i++) {
                ArrayList<IDanmakuItem> list = mChannelMap.get(i);
                if (list.size() == 0) {
                    return i;
                }
            }
            int ind = random.nextInt(mMaxRow);
            for (int i = 0; i < mMaxRow; i++) {
                ArrayList<IDanmakuItem> list = mChannelMap.get((i + ind) % mMaxRow);
                if (list.size() > mMaxRunningPerRow) {//每个弹道最多mMaxRunning个弹幕
                    continue;
                }
				//直接判断会不会和arraylist队尾的Item碰撞即可
                IDanmakuItem di = list.get(list.size() - 1);
                if (!item.willHit(di)) {
                    return (i + ind) % mMaxRow;
                }
            }
        } catch (Exception e) {
            Log.w(TAG, "findVacant,Exception:" + e.toString());
        }
        return -1;
    }
 	/**
     * 播放显示弹幕
     */
    public void show() {
        status = STATUS_RUNNING;
        invalidate();
    }

    /**
     * 隐藏弹幕,暂停播放
     */
    public void hide() {
        status = STATUS_PAUSE;
        invalidate();
    }

    /**
     * 清空正在播放和等待播放的弹幕
     */
    public void clear() {
        status = STATUS_STOP;
        clearItems();
        invalidate();
        init();
    }
	/**
     * 是否新建后台线程来执行添加任务
     */
    public void addItem(final List<IDanmakuItem> list, boolean backgroundLoad) {
        if (backgroundLoad) {
            new Thread() {
                @Override
                public void run() {
                    synchronized (mWaitingItems) {
                        mWaitingItems.addAll(list);
                    }
                    postInvalidate();
                }
            }.start();
        } else {
            this.mWaitingItems.addAll(list);
        }
    }
	//计算fps值 frame per second
	private double fps() {
        long lastTime = System.nanoTime();
        times.addLast(lastTime);
        double NANOS = 1000000000.0;
        double difference = (lastTime - times.getFirst()) / NANOS;
        int size = times.size();
        int MAX_SIZE = 100;
        if (size > MAX_SIZE) {
            times.removeFirst();
        }
		//计算每秒的帧数
        return difference > 0 ? times.size() / difference : 0.0;
    }

其他的属性设置和资源回收的代码就不一一粘贴了，下面看一下具体的代码调用，由于业务需要，本次直接通过代码进行调用

		if (mType.equals("0")) {
            mDanmuLayout = View.inflate(context, R.layout.danmu, null);
            mDanmuView = (DanmuView) mDanmuLayout.findViewById(R.id.danmuView);
            ViewGroup.LayoutParams layoutParams = mDanmuView.getLayoutParams();
            layoutParams.height = ScreenUtils.getScreenWidth(context) / 2;
            mDanmuView.setLayoutParams(layoutParams);
            mDanmuView.show();
            linearLayout.addView(mDanmuLayout);
        }
        return linearLayout;

关于R.layout.danmu的配置如下

	<DanmuView
        android:id="@+id/danmuView"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:end_Y_offset="0.8"
        app:max_row="5"
        app:max_running_per_row="1"
        app:pick_interval="1000"
        app:show_debug="false"
        app:start_Y_offset="0.2" />

剩下的只需在网络访问得到数据后把数据封装成DanmuItem,添加进DanmuView中即可
另由于此项目中是要展示圆形的用户图片，所以附加一个得到圆形图片的方法

	//获取圆角bitmap
    public Bitmap getRoundedCornerBitmap(Bitmap bitmap) {
        Bitmap output = Bitmap.createBitmap(bitmap.getWidth(),
                bitmap.getHeight(), Bitmap.Config.ARGB_8888);
        Canvas canvas = new Canvas(output);

        final int color = 0xff424242;
        final Paint paint = new Paint();
        final Rect rect = new Rect(0, 0, bitmap.getWidth(), bitmap.getHeight());
        final RectF rectF = new RectF(rect);
        final float roundPx = bitmap.getWidth() / 2;

        paint.setAntiAlias(true);
        canvas.drawARGB(0, 0, 0, 0);
        paint.setColor(color);
        canvas.drawRoundRect(rectF, roundPx, roundPx, paint);

        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));
        canvas.drawBitmap(bitmap, rect, rect, paint);

        return output;
    }
主要是通过PorterDuffXfermode来实现，

	canvas原有的图片 可以理解为背景 就是dst 
	新画上去的图片 可以理解为前景 就是src 

![](http://img.blog.csdn.net/20130828212947609?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdDEyeDM0NTY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

详情请查看 [Android 颜色渲染(九) PorterDuff及Xfermode详解](http://blog.csdn.net/t12x3456/article/details/10432935)

OK 到现在自定义View实现弹幕已经完成，但自定义view的威力还远远不止这些，还有很多可以发挥的空间，希望以后能做出更多好看好玩的东西出来给大家...

另这里还有一篇关于View的绘制流程的分析，有兴趣的可以参考一下
[Android中View绘制流程以及invalidate()等相关方法分析](http://blog.csdn.net/qinjuning/article/details/7110211/)