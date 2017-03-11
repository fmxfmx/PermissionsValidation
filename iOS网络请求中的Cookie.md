#iOS网络请求中的Cookie

在最近的项目中用到了网络请求相关的知识,之前有对权限认证的有关知识做了一个简单的[记录](http://www.jianshu.com/p/88336eab2b8d)有兴趣的可以去捧个场.下面进入正题:

##Cookie
我们先来简单的了解下Cookie是什么？

在我们使用的HTTP协议传输时一旦数据交互完毕这时候服务端与客户端的连接就会关闭,那么如果客户端这时再次与服务端交换数据的时候一切又要重新来过,这时候我们就需使用到Cookie了.

当客户端与服务端首次进行连接的时候,服务端会给客户端发一个Cookie(通行证),当再次与服务端进行交互的时候都需要带上这个Cookie(通行证),这样服务端就可以通过这个Cookie(通行证)确认访问者的身份了.举个栗子:当用户在某个网站进行了登录操作后，服务端会将Cookie信息返回给终端，终端会将这些信息进行保存，在下一次再次访问这个网站时，终端会将保存的Cookie信息一并发送到服务端，服务端根据Cookie信息是否有效来判断此用户是否可以自动登录.

那么在iOS中我们用到的`NSURLRequest`其实已经为我们自动保存了每次会话的Cookie,当然你不需要保存的时需要将这个属性`HTTPShouldHandleCookies`置为NO即可,那么我们请求的Cookie都会被存在`NSHTTPCookieStorage`中.

##ç
这个类采用了单例的设计模式,也就是说它管理着所有HTTP请求的Cookie,那么我们来看一下它常用的方法:

```objective-c
// 获取单例对象
@property(class, readonly, strong) NSHTTPCookieStorage *sharedHTTPCookieStorage;

// 含有所有Cookie的数组,在iOS中Cookie都被封装成了NSHTTPCookie对象,后面会有介绍
@property (nullable , readonly, copy) NSArray<NSHTTPCookie *> *cookies;

// 手动设置一条Cookie
- (void)setCookie:(NSHTTPCookie *)cookie;

// 删除某条Cookie
- (void)deleteCookie:(NSHTTPCookie *)cookie;

// 删除某个时间段后所有的Cookie(iOS 8以后有效)
- (void)removeCookiesSinceDate:(NSDate *)date;

// 根据一个Url获取到这个Url下的所有Cookie
- (nullable NSArray<NSHTTPCookie *> *)cookiesForURL:(NSURL *)URL;

// Cookie数据的接受方式,是个枚举
@property NSHTTPCookieAcceptPolicy cookieAcceptPolicy;
typedef NS_ENUM(NSUInteger, NSHTTPCookieAcceptPolicy) {
	// 接收所有的Cookie
    NSHTTPCookieAcceptPolicyAlways,
    // 不接收Cookie
    NSHTTPCookieAcceptPolicyNever,
    // 只接收特定的Cookie
    NSHTTPCookieAcceptPolicyOnlyFromMainDocumentDomain
};

// 此方法只有在cookieAcceptPolicy为NSHTTPCookieAcceptPolicyOnlyFromMainDocumentDomain的时候才有效
- (void)setCookies:(NSArray<NSHTTPCookie *> *)cookies forURL:(nullable NSURL *)URL mainDocumentURL:(nullable NSURL *)mainDocumentURL;
```

##NSHTTPCookie
Cookie在iOS系统中都被封装成了`NSHTTPCookie`对象,接下来简单介绍下这个类的使用方法:

```objective-c
//下面两个方法用于对象的创建和初始化 都是通过字典进行键值设置
- (nullable instancetype)initWithProperties:(NSDictionary<NSString *, id> *)properties;
+ (nullable NSHTTPCookie *)cookieWithProperties:(NSDictionary<NSString *, id> *)properties;

//返回Cookie数据中可用于添加HTTP头字段的字典
+ (NSDictionary<NSString *, NSString *> *)requestHeaderFieldsWithCookies:(NSArray<NSHTTPCookie *> *)cookies;

//从指定的响应头和URL地址中解析出Cookie数据
+ (NSArray<NSHTTPCookie *> *)cookiesWithResponseHeaderFields:(NSDictionary<NSString *, NSString *> *)headerFields forURL:(NSURL *)URL;

//Cookie数据中的属性字典
@property (nullable, readonly, copy) NSDictionary<NSString *, id> *properties;

//请求响应的版本
@property (readonly) NSUInteger version;

//请求相应的名称
@property (readonly, copy) NSString *name;

//请求相应的值
@property (readonly, copy) NSString *value;

// 如果YES在session结束后会被废弃,NO的话就不需要被废弃
@property (readonly, getter=isSessionOnly) BOOL sessionOnly;

//过期时间
@property (nullable, readonly, copy) NSDate *expiresDate;

//请求的域名
@property (readonly, copy) NSString *domain;

//请求的路径
@property (readonly, copy) NSString *path;

//是否是安全传输
@property (readonly, getter=isSecure) BOOL secure;

//是否只发送HTTP的服务
@property (readonly, getter=isHTTPOnly) BOOL HTTPOnly;

//响应的文档
@property (nullable, readonly, copy) NSString *comment;

//向用户描述服务器为此Cookie提供的Url注释
@property (nullable, readonly, copy) NSURL *commentURL;

//服务端口列表
@property (nullable, readonly, copy) NSArray<NSNumber *> *portList;
```

我们可以根据这两个类来管理我们在请求中的Cookie.

我们可以以请求"http://www.baidu.com"为例来看看这些东西:

第一步:先创建一个请求

```objective-c
    NSURLSession *session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration] delegate:self delegateQueue:[[NSOperationQueue alloc] init]];
    
    NSURLSessionDataTask *task = [session dataTaskWithURL:[NSURL URLWithString:@"http://www.baidu.com"]];
    [task resume];
```

第二步:在请求完成后点击屏幕获取到cookie

```objective-c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    NSHTTPCookieStorage *storeage = [NSHTTPCookieStorage sharedHTTPCookieStorage];
    NSLog(@"%@",storeage.cookies);
}

打印结果如下：
<__NSArrayM 0x6080002414a0>(
<NSHTTPCookie version:0 name:"BAIDUID" value:"FB727DBDDADC6677950CBE40B642FCFA:FG=1" expiresDate:2085-03-16 18:51:51 +0000 created:2017-02-26 15:37:44 +0000 sessionOnly:FALSE domain:".baidu.com" partition:"none" path:"/" isSecure:FALSE>,
<NSHTTPCookie version:0 name:"BIDUPSID" value:"FB727DBDDADC6677950CBE40B642FCFA" expiresDate:2085-03-16 18:51:51 +0000 created:2017-02-26 15:37:44 +0000 sessionOnly:FALSE domain:".baidu.com" partition:"none" path:"/" isSecure:FALSE>,
<NSHTTPCookie version:0 name:"H_PS_PSSID" value:"1461_21100_22035_20930" expiresDate:(null) created:2017-02-26 15:37:44 +0000 sessionOnly:TRUE domain:".baidu.com" partition:"none" path:"/" isSecure:FALSE>,
<NSHTTPCookie version:0 name:"PSTM" value:"1488123464" expiresDate:2085-03-16 18:51:51 +0000 created:2017-02-26 15:37:44 +0000 sessionOnly:FALSE domain:".baidu.com" partition:"none" path:"/" isSecure:FALSE>,
<NSHTTPCookie version:0 name:"BDSVRTM" value:"0" expiresDate:(null) created:2017-02-26 15:37:44 +0000 sessionOnly:TRUE domain:"www.baidu.com" partition:"none" path:"/" isSecure:FALSE>,
<NSHTTPCookie version:0 name:"BD_HOME" value:"0" expiresDate:(null) created:2017-02-26 15:37:44 +0000 sessionOnly:TRUE domain:"www.baidu.com" partition:"none" path:"/" isSecure:FALSE>
)
```
