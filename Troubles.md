# Troubles

**1.MBProgressHUD**

版本号：0.9

问题：hud实例不正常remove的问题

分析：

a.加载到UIApplication.shareApplication.keyWindow上存在隐患

原因：keywindow可能会被改变，例如弹出系统弹窗时，此时keyWindow为系统弹窗的window

解决方案：避免添加hud到keyWindow上。而应添加至APPDelegate的widow上。

b.连续添加多个hud，并连续移除多个hud时使用移除动画。

分析：连续添加多个hud，移除时使用动画，会触发源代码中NSObject类的cancelPreviousPerformRequestsWithTarget:selector:object:方法，导致即将需要移除的hud移除失败。

解决方案：连续创建hud并移除时，不使用动画，或者直接升级版本只1.0.0以及以上版本，1.0.0后的版本移除这一做法。


**2.YYLabel**

问题：YYLabel使用YYLayout计算labl的高度，设置label的属性时，在XR上造成绘图（因为YYLabel显示的内容是生成image并将其赋值给对应view的layer的contents属性）模糊，但在IphoneX以下机型不明显，个人猜测受分辨率影响。

分析：通过打印YYLabel的API计算出的label的宽高，以及使用autoLayout计算出来的宽高，发现两者之间存在像素偏差，[原因暂未知]()

解决方案：重新计算label文本的实际宽高，并设置frame，或者使用autoLayout布局，让label自适应宽高。不使用YYLayout计算出的宽高进行赋值

**2.Xcode Debug/Release**

问题：Debug和Release环境下，NSNumber强转BOOL值表现不一致。
例如：

	- (void)viewDidLoad {
    	[super viewDidLoad];
    
    	[self transform:@(NO)];
	}

	- (void)transform:(BOOL)value {
   		NSLog(@"%d", value);
	}
	
在Release情况下，value的值为YES，在Debug模式下，value的值为NO。

解决方案：传入对应数据类型，不进行强转，调用boolValue。
