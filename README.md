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

**3.Xcode Debug/Release**

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

**4.CLGeocoder（地址反编码）**
问题：开启定位权限，进行定位，在代理方法`locationManager:didUpdateLocations:`中使用CLGeocoder直接进行地址反编码，编码报错`Error Domain=kCLErrorDomain Code=2 "(null)"`。

	CLGeocoder *clGeoCoder = [[CLGeocoder alloc] init];
    CLGeocodeCompletionHandler handle = ^(NSArray *placemarks,NSError *error) {
        if (error) {
            if (self.delegate && [self.delegate respondsToSelector:@selector(locationAddressDictionary:error:)]) {
                  [self.delegate locationAddressDictionary:nil error:error];
                }
            }
            
        for (CLPlacemark *placeMark in placemarks) {
                NSDictionary *addressDic = placeMark.addressDictionary;
                
            if (self.locationAddressDictionary) {
                self.locationAddressDictionary(addressDic, error);
            }
        }
    };
        
    [clGeoCoder reverseGeocodeLocation:location completionHandler:handle];

问题分析:查看方法`reverseGeocodeLocation: completionHandler:`方法描述时发现，该请求不能在短时间内多次请求，否则会报错。

After initiating a reverse-geocoding request, do not attempt to initiate another reverse- or forward-geocoding request. Geocoding requests are rate-limited for each app, so making too many requests in a short period of time may cause some of the requests to fail. When the maximum rate is exceeded, the geocoder passes an error object with the value kCLErrorNetwork to your completion handler.

解决方案：使用标记位，判断回调回来时，是否正在进行反编码，如果在反编码则不调用`reverseGeocodeLocation: completionHandler:`方法.

	if (!self.isReverseGeocoding) {
       self.isReverseGeocoding = YES;
       CLGeocoder *clGeoCoder = [[CLGeocoder alloc] init];
       CLGeocodeCompletionHandler handle = ^(NSArray *placemarks,NSError *error) {
       self.isReverseGeocoding = NO;
       if (error) {
          if (self.delegate && [self.delegate respondsToSelector:@selector(locationAddressDictionary:error:)]) {
                [self.delegate locationAddressDictionary:nil error:error];
          }
       }
            
       for (CLPlacemark *placeMark in placemarks) {
           NSDictionary *addressDic = placeMark.addressDictionary;
                
           if (self.locationAddressDictionary) {
                self.locationAddressDictionary(addressDic, error);
           }
       };
        
    // 短时间内不能重复请求，否则会报错
    [clGeoCoder reverseGeocodeLocation:location completionHandler:handle];
    
**5.阿里百川SDK（AlibcTradeSDK）**

版本号：4.0.0.2

问题：启动APP触发[AppMonitorTaskPool start]: attempt to start the thread again。导致启动即闪退。

分析与解决方案：

1.更换新的已经修改改bug的SDK。（但是，我们咨询阿里相关开发，该问题他们已经知悉，但是需要下版本进行修复，但修复时间与自己项目发布时间不一致，因此只能项目中自己避免该问题)

2.咨询过阿里的同学知道这是`AppMonitorTaskPool `是继承自`NSThread`。因此可以判断出，造成闪退的原因是因为同一线程多次调用`start`方法引起。因此，可以通过hook掉`NSThread`的start方法，并添加计数器，判断调用`start`方法的次数是否大于等于1，如果大于等于1次，那么进行拦截，否则，正常调用`start`方法。在`cancel`方法中，如果计数器的值大于等于1，那么调用`cancel`方法时，将计数器减1.

	- (void)mm_start{
    	Class clsseee = NSClassFromString(@"AppMonitorTaskPool");
    	if ([self isKindOfClass:clsseee]) {
        	if (self.hookCount>=1) {
            return;
        	}
    	}
    	if ([self isKindOfClass:clsseee]) {
        	self.hookCount++;
    	}
    	[self mm_start];
	}