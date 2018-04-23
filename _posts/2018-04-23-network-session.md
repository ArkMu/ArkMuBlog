![](https://user-gold-cdn.xitu.io/2018/4/21/162e75f041cb80ac?w=1000&h=750&f=jpeg&s=129416)
### Session
Session是服务器端使用一种类似散列表的结构来保存信息的机制。当应用程序请求简历Session连接时，服务器首先会检测请求头是否包含SessionID，如果已经包含，服务器回去散列表里查询响应的Session并进行行为关联。如果没有，则会先创建一个Session，生成一个SessionID，并在本次响应中返回个客户端。

### System Session

#### NSURLSession

理解Session的原理后，我们先来看一下iOS的NSURLSession类，看一下苹果是如何提供接口的。

首先来看一下如何生成一个NSURLSession对象。

``` objc
// The Shard session uses the currently set global NSURLCache, NSHTTPCookieStorage and NSURLCredentialStorage objects.
@property (class, readonly, strong) NSURLSession *shareSession;

+ (NSURLSession *)sesionWithConfiguration:(NSURLSessionConfiguration *)configuration;
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(nullable id<NSURLSessionDelegate>)delegate delegateQueue:(nullable NSOperationQueue *)queue;
```

苹果提供了三种构造方式，对于shareSession的定义，可以学习一下。

``` objc
@property (readonly, retain) NSOperationQueue *delegateQueue;
@property (nullable, readonly, retain) id <NSURLSessionDelegate> delegate;
@property (readonly, copy) NSURLSessionConfiguration *configuration;
```
从代码可以看出，三个属性都是只读，delegateQueue和delegate是retain，引用计数加一，release交由autoreleasePool，delegate可空，configuration的copy(copy = 堆上复制 + strong)。

``` objc
- (void)finishTaskAndInvalidate;
- (void)invalidateAndCancel;

- (void)resetWithCompletionHandler:(void (^)(void))completionHandler;
- (void)flushWithCompletionHandler:(void (^)(void))completionHandler;

- (void)getTasksWithCompletionHandler:(void (^)(NSArray <NSURLSessionDataTask *> *dataTasks, NSArray<NSURLSessionUploadTask *> *uploadTasks, NSArray<NSURLSessionDownloadTask *> *downloadTasks))completionHandler;
- (void)getAllTasksWithCompletionHandler:(void (^)(NSArray<__kindof NSURLSessionTask *> tasks))completionHandler;
```
对于后两个方法，由于NSURLSessionDataTask、NSURLSessionUploadTask(NSURLSessionDataTask)、NSURLSessionDownloadTask都是NSURLSessionTask的子类，所以使用了__kinfof来修饰NSURLSessionTask。

``` objc
// NSURLSession 提供通过resumeData创建downloadTask的方式。
- (NSURLSessionDownloadTask *)downloadTaskResumeData:(NSData *)resumeData;
```

#### NSURLSessionTask

接下来我们来看一下NSURLSessionTask。

``` objc
@property (readonly)                    NSUInteger      taskIdentifier;
@property (nullable, readonly, copy)    NSURLRequest    *originalRequest;
@property (nullable, readonly, copy)    NSURLRequest    *currentRequest;
@property (nullable, readonly, copy)    NSURLResponse   *response;
... ...
@property (readonly)                    int64_t         countOfBytesReceived;
@property (readonly)                    int64_t        countOfBytesSent;
@property (readonly)                    int64_t        countOfBytesExpectedToSend;
@property (readonly)                    int64_t        countOfBytesExpectedToReceive;

- (void)cancel;

@property (readonly)                    NSURLSessionTaskState   state;

- (void)suspend;
- (void)resume;
```
任务的重启与暂停，数据传输的信息，请求和响应等。

#### NSURLSessionConfiguration

下面我们来看一下什么是NSURLSessionConfiguration？

``` objc
@property (class, readonly, strong) NSURLSessionConfiguration *defaultSessionConfiguration;
@property (class, readonly, strong) NSURLSessionConfiguration *ephemeralSessionConfiguration;

+ (NSURLSessionConfiguration *)backgroundSessionConfigurationWithIdentifier:(NSString *)identifier;
```
NSURLSessionConfiguration提供了三种构造方式，ephemeralSessionConfiguration不会保存用户的浏览信息，可以用来做无痕浏览。

``` objc
@property NSURLRequestCachePolicy requestCachePolicy;

@property NSTimeInterval timeoutIntervalForRequest;
@property NSTimeInterval timeoutIntervalForResource;
@property NSURLRequestNetworkServiceType networkServiceType;

... ...

@property (nullabel, copy) NSDictionary *connnectionProxyDictionary;
@property SSLProtocol TLSMinimumSupportedProtocol;
@property SSLProtocol TLSMaximumSupportedProtocol;

@property BOOL HTTPShouldUsePipeling;
@property BOOL HTTPShouldSetCookies;

@property NSHTTPCookieAcceptPolicy HTTPCockieAcceptPolicy;

@property (nullable, copy) NSDictionary *HTTPAdditionalHeaders;

@property NSInteger HTTPMaximumConnectionsPerHost;

@property (nullable, retain) NSHTTPCookieStorage *HTTPCookieStorage;
@property (nullable, retain) NSURLCredentialStorage;
@property (nullable, retain) NSURLCache *URLCache;

... ...
```

NSURLSessionConfiguration是有很多属性需要设置的，像超时，SSL的处理，cookie的设置，是否允许pipe，最大连接数，credentials的存储等等。

上文我们大致了解了NSURLSession相关的属性和方法后，接下来我们来看一下如何构造一个NSURLRequest对象，他有哪些属性和方法。

#### NSURLRequest

``` objc
+ (instancetype)requestWithURL:(NSURL *)URL cachePolicy:(NSURLRequestCachePolicy)cachePolicy timeoutInterval:(NSTimeInterval)timeoutInterval;

- (instancetype)requestWithURL:(NSURL *)URL cachePolicy:(NSURLRequestCachePolicy)cachePolicy timeoutInterval:(NSTimeInterval)timeoutInterval;
```
从构造方法中可以看出，request包含URL、cachePolicy和超时时间。

``` objc
@interface NSURLRequest (NSHTTPURLRequest)

@property (nullable, readonly, copy) NSString *HTTPMethod;
@property (nullable, readonly, copy) NSDictionary <NSStirng *, NSString *>allHTTPHeaderFields;

@property (nullable, readonly, copy) NSData *HTTPBody;
@property (nullable, readonly, retain) NSInputStream *HTTPBodyStream;
@property (readonly) BOOL HTTPShouldHandleCookies;
@property (readonly) BOOL HTTPShouldUSePipelining;

- (nullable, NSString *)valueForHTTPHeaderFiled:(NSString *)field;

@end
```
从类别可以看出，主要设置请求头，设置请求体，设置请求方式，是否添加cookie等。

#### NSURLResponse
设置完请求后，我们来看一下响应。
##### NSURLResponse
``` objc
- (instancetype)initWithURL:(NSURL *)URL MIMEType:(nullable NSString *)MIMEType expectedContentLength:(NSInteger)length textEncodingName:(nullable NSString *)name;

@property (nullable, readonly, copy) NSString *suggestedFilename;;
```
NSURLResponse类包含contentLength，编码格式，建议文件名字等。
##### NSHTTPURLResponse

``` objc
- (nullable instancetype)initWithURL:(NSURL *)url statuscode:(NSInteger)statusCode HTTPVersion:(nullable NSString *)HTTPVersion headerFields:(nullable NSDictionary<NSString *, NSString *>)headerFields;

+ (NSString *)localizedStringForStatusCode:(NSInteger)statusCode;
```
NSHTTPURLResponse包含状态码，HTTPVersion，响应头信息。

#### NSURLCredential

接下来谈一谈证书

``` objc
@property (readonly) NSURLCredentialPersistence persistence;
```
证书的持有方式有不保存，由session保存，永久保存，同步四种方式。

##### NSURLCredential(NSInternetPasscode)

``` objc
- (instancetype)initWithUser:(NSString *)user password:(NSString *)passcode persistence:(NSURLCredentialPersistence)persistence;
```

##### NSURLCredential(NSClientCertificate)
``` objc
- (instancetype)initWithIdentity:(SecIdentityRef)identity certificates:(nullable NSArray *)certArray persistence:(NSURLCredentialPersistence)persistence;
```
##### NSURLCredential(NSServerTrust)

``` objc
- (instancetype)initWithTrust:(SecTrustRef)trust;
```

#### NSURLAuthenticationChallenge

``` objc
@protocol NSURLAuthenticationChallengeSender <NSObject>
- (void)useCredential:(NSURLCredential *)credential forAuthenticationChallenge:(NSURLAuthenticationChallenge *)
challenge;
- (void)continueWithOutCredentialForAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge;
- (void)cancelAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge;

@optional
- (void)performDefaultHandlingForAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge;
- (void)rejectProtectionSpaceAndContinueWithChallenge:(NSURLAuthenticationChallenge *)challenge;
@end
```

``` objc
- (instancetype)initWithProtectionSpace:(NSURLProtectionSpace *)space proposedCredential:(nullable NSURLCredential *)credential previousFailureCount:(NSInteger)previousFailureCount failureResponse:(nullable NSURLResponse *)response error:(nullable NSError *)error sender:(id<NSURLAuthenticationChallengeSender>)sender;
- (instancetype)initWithAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge sender:(id<NSURLAuthenticationChallengeSender>)sender;
@property (readonly, copy) NSURLProtectionSpace *protectionSpace;
@property (nullable, readonly, copy) NSURLCredential *proposedCredential;
... ...
@property (nullable, readonly, retain) id<NSURLAuthenticationChallengeSender> sender;
```

#### NSURLProtectionSpace

``` objc
- (instancetype)initWithHost:(NSString *)host port:(NSInteger)port protocol:(nullable NSString *)protocol realm:(nullable NSString *)realm authenticationMethod:(nullable NSString *)authenticationMethod;
- (instancetype)initWithProxyHost:(NSString *)host port:(NSInteger)port type:(nullable NSString *)type realm:(nullable NSString *)realm  authenticationMethod:(nullable NSString *)authenticationMethod;

@property (readonly) BOOL receivesCredentialSecurely;
@property (readonly) BOOL isProxy;
... ... 
```

从上述文章来看，要完整构造一个网络请求，需要涉及方方面面，还是蛮复杂的。接下来我们就来看一个第三方库：AFNetWorking。

### AFNetWorking
上文分析过系统处理网络请求的类之后，我们来看一下AFNetWorking做了些什么？

#### AFURLSessionManager

``` objc
@property (readonly, nonatomic, strong) NSURLSession *session;
@property (readonly, nonatomic, strong) NSOperationQueue *operationQueue;
@property (nonatomic, strong) id <AFURLResponseSerialization>responseSerializer;
@property (nonatomic, strong) AFSecurityPolicy *securityPolicy;
```
manager包含有NSURLSession对象，operationQueue，遵循自定义协议AFURLResponseSerialization的代理responseSerializer，自定义类AFSecurityPolicy的对象。
``` objc
#if !TARGET_OS_WATCH
@property (readwrite, nonatomic, strong) AFNetworkReachabilityManager *reachabilityManager;
#endif
```
manager包含网络监听类AFNetWorkReachabilityManager的对象。

``` objc
@property (nonatomic, strong, nullable) dispatch_queue_t completionQueue;
@property (nonatomic, strong, nullable) dispatch_group_t completionGroup;
@property (nonatomic, assign) BOOL attemptsToRecreateUploadTasksForBackgroundSessions;
```
队列completionQueue，组completionGroup，是否允许在后台创建上传任务。

``` objc
- (instancetype)initWithSessionConfiguration:(nullable NSURLSessionConfiguration *)configuration;
```
构造方法

``` objc
- (void)invalidateSessionCancelingTasks:(BOOL)cancelPendingTasks;
```
注销session同时根据传参决定是否取消任务。

``` objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)reuqest
                            uploadProgress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                          downloadProgress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                         completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError * _Nullable error))completionHandler;
```
创建dataTask，通过Block回调处理返回信息。

``` objc
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromFile:(NSURL *)fileURL
                                         progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                                completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError * _Nullabel error))completionHandler;

- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         framData:(nullable NSData *)bodyData
                                         progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                                    completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError * _Nullable error))completionHandler;
- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request
                       progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                    completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError * _Nullable error))completionHandler;
```
三种创建uploadTask的方式，上传文件，上传数据，将数据整合进httpRequest上传。

``` objc
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                             progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                                            destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                        completionHandler:(nullable void (^)(NSURLRepsonse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler;

- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData
                                                progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                                             destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                        completionHandler:(nullable void (^)(NSURLResponse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler
```
创建downloadTask，提供断点下载方法。

``` objc
- (nullable NSProgress *)uploadProgressForTask:(NSURLSessionTask *)task;
- (nullable NSProgress *)downloadProgressForTask:(NSURLSessionTask *)task;
```
获取上传、下载的进度。

AFNetWorking将系统提供的delegate方法封装成block块执行，block相比于delegate，因为返回处理的代码和初始化的代码在一块，更易于也理解。但是也会造成代码结构不清晰。
如：`- (void)setDownloadTaskDidFinishDownloadingBlock:(nullable NSURL * _Nullable  (^)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location))block;`。

#### AFURLRequestSerialization

##### AFHTTPRequestSerializer
除了编码格式(stringEncoding)，是否允许蜂窝通信(allowsCellularAccess)，cachePolicy，是否添加cookie，是否使用管道流(HTTPShouldUsePipeling)，networkServiceType，超时(timeoutInterval)，请求头(HTTPRequestHeaders)等，我们来看一下AFHTTPRequestSerializer的方法。
``` objc
+ (instancetype)serializer;

- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                 parameters:(nullable id)parameters
                                      error:(NSError * _Nullable __autoreleasing *)error;
- (NSMutableURLRequest *)multipartFromRequestWithMethod:(NSString *)method
                                                URLString:(NSString *)URLString
                                               parameters:(nullable NSDictionary <NSString *, id> *)parameters
                                        constructingBodyWithBlock:(nullable void (^)(id <AFMultipartFormData> formData))block
                                                     error:(NSError * _Nullable __autoreleasing *)error;
```

##### AFJSONRequestSerializer
``` objc
+ (instancetype)serializerWithWritingOptions:(NSJSONWritingOptions)writingOptions;
```
相较于AFHTTPRequestSerializer，AFJSONRequestSerializer多了一个NSJSONWritingOptions
##### AFPropertyListRequestSerializer
``` objc
+ (instancetype)serializerWithFormat:(NSPropertyListFormat)format
                        writeOptions:(NSPropertyListWriteOptions)writeOptions;
```

属性列表的序列化

#### AFURLResponseSerialization
看完请求序列化后，我们来看一下响应序列化

##### AFHTTPResponseSerializer
``` objc
 + (instancetype)serializer;
@property (nonatomic, copy, nullable) NSIndexSet *acceptableStatusCodes;
@property (nonatomic, copy, nullable) NSSet <NSString *> *acceptableContentTypes;
```
状态码，contentTypes

##### AFJSONResponseSerializer

``` objc
+ (instancetype)serializerWithReadingOptions:(NSJSONReadingOptions)readingOptions;

@property (nonatomic, assign) BOOL removesKeysWithNullValues;
```

##### AFXMLParserResponseSerializer

##### AFXMLDocumentResponseSerializer

``` objc
+ (instancetype)serializerWithXMLDoxumentOptions:(NSUinteger)mask;
```

##### AFPropertyListResponseSerializer

``` objc
+ (instancetype)serializerWithFormat:(NSPropwertyListFormat)format
                         readOptions:(NSPropertyListReadOptions)readOptions;
```
和AFPropertyListRequestSerializer一样

##### AFImageResponseSerializer

``` objc
@property (nonatomic, assign) CGFloat imageScale;
@property (nonatomic, assign) BOOL automaticallyInflateResponseImage;
```

#### AFSecurityPolicy

添加SSL证书可以帮助应用阻止中间人攻击，应用在处理用户敏感信息或者是金融交易时强烈建议使用HTTPS。
``` objc
@property (readonly, nonatomic, assign) AFSSLPinningMode SSLPinningMode;
```
验证方式AFSSLPinningModeNone(不验证) AFSSLPinningModePublicKey(通过公钥) AFSSLPiningModeCertificate（通过证书）

``` objc
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode withPinnedCertificates:(NSSet <NSData *> *)pinnedCertificates;

- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                forDomain:(nullable NSString *)domain;
```

#### AFHTTPSessionManager

在我们的应用中，主要是使用AFHTTPSessionManager进行网络请求，我们来看一下他里面有什么属性和方法？
``` objc
// URL
@property (readonly, nonatomic, strong, nullable) NSURL *baseURL;
// 请求序列化
@property (nonatomic, strong) AFHTTPRequestSerializer <AFURLRequestSerialization> * requestSerializer;
// 响应序列化
@property (nonatomic, strong) AFHTTPResponseSerializer <AFURLResponseSerialization> * responseSerializer;
// 安全策略
@property (nonatomic, strong) AFSecurityPolicy *securityPolicy;
```

``` objc
+ (instancetype)manager;
- (instancetype)initWithBaseURL:(nullable NSURL *)url;
- (instancetype)initWithBaseURL:(nullable NSURL *)url
           sessionConfiguration:(nullable NSURLSessionConfiguration *)configuration;
```
三种初始化方法

``` objc
- (nullable NSURLSessionDataTask *)GET:(NSString *)URLString
                            parameters:(nullable id)parameters
                              progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgress
                               success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                               failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;
- (nullable NSURLSessionDataTask *)HEAD:(NSString *)URLString
                    parameters:(nullable id)parameters
                       success:(nullable void (^)(NSURLSessionDataTask *task))success
                       failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;
- (nullable NSURLSessionDataTask *)POST:(NSString *)URLString
                             parameters:(nullable id)parameters
                               progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgress
                                success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                                failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;
- (nullable NSURLSessionDataTask *)POST:(NSString *)URLString
                             parameters:(nullable id)parameters
              constructingBodyWithBlock:(nullable void (^)(id <AFMultipartFormData> formData))block
                               progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgress
                                success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                                failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;
- (nullable NSURLSessionDataTask *)PUT:(NSString *)URLString
                   parameters:(nullable id)parameters
                      success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                      failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;
- (nullable NSURLSessionDataTask *)PATCH:(NSString *)URLString
                     parameters:(nullable id)parameters
                        success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                        failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;
- (nullable NSURLSessionDataTask *)DELETE:(NSString *)URLString
                      parameters:(nullable id)parameters
                         success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                         failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;
```
经过AFNetWorking封装后，我们在进行网络编程是只需要面向AFHTTPSessionManager进行编程，网络请求任务繁重程度减轻了不少。

### HTTPS
HTTPS(Hyptertext Transfer Protocol over Secure Socket Layer)，所以HTTPS = HTTP + SSL。
接下来我们先认识一下加解密。
#### 对称加密
对称加密在对信息进行加密和解密是使用相同的密钥。常见的对称加密有：DES(Data Encryption Standard)、AES(Advances Encryption Standard), RC4、IDEA。
#### 非对称加密
客户端C生成一对公钥(CL)和私钥(CK)，服务器端(S)生成了一对公钥(SL)和私钥(SK)，C想S发信息时用SL加密，S收到信息后，用SK解密；同样，S向C发消息时，用CL加密，C收到信息后用CK解密。公钥就是锁，私钥就是钥匙。信息被拦截后，由于拦截者没有私钥，也打不开锁。信息依然是安全的。一句话来说。非对称加密就是用对方的锁加密信息，整个过程存在两把锁，而对称加密使用一把锁。
#### 摘要算法
采用Hash算法将需要加密的铭文"摘要"成一串固定长度(128位)的密文，不同明文摘要成密文，结果不同。可以确保信息完整性、防止信息被篡改。
#### 数字签名
`明文-> hash运算 -> 摘要 -> 私钥加密 -> 数字签名`
确保信息确实是由发送方签名并发出的。数字签名确保信息的完整性。
#### 数字证书
简单来说，第三方机构确保SL的确是S的锁，AL的确是A的锁。
通信过程如下：
![](https://user-gold-cdn.xitu.io/2018/4/21/162e75a0b6d86136?w=822&h=600&f=png&s=80370)

参考文章：[详解https是如何确保安全的？](http://www.wxtlife.com/2016/03/27/详解https是如何确保安全的？/)

转载请注明出处。
