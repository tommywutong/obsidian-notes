---
title: "【iOS】AFNetworking"
slug: "ios-afnetworking"
published: 2025-09-14
description: "文件构成 AFURLSessionManager AFHTTPSessionManager 一次完整的GET请求过程 1. 初始化和配置 2. GET请求 3. 添加请求头的GET POST请求 举例：Spotify内容的获取 文件构成 AFNetworking的构成很简单，主要"
tags: ["iOS", "AFNetworking", "网络"]
category: "iOS"
draft: false
---

- [文件构成](#----)
- [AFURLSessionManager](#afurlsessionmanager)
- [AFHTTPSessionManager](#afhttpsessionmanager) [一次完整的GET请求过程](#---------) [1. 初始化和配置](#1-------)
- [2. GET请求](#2-get--)
- [3. 添加请求头的GET](#3-------get)

- [POST请求](#post--)

- [举例：Spotify内容的获取](#---spotify-----)


## 文件构成


AFNetworking的构成很简单，主要就四个部分：


- Manager : 负责处理网络请求的两个`Manager`，主要实现都在`AFURLSessionManager`中。
- Reachability : 网络状态监控。
- Security : 处理网络安全和`HTTPS`相关的。
- Serialization : 请求和返回数据的格式化器。


## AFURLSessionManager


在AFN中，网络请求的manager主要有AFHTTPSessionManager 和 AFURLSessionManager 构成，二者为父子关系。这两个类职责划分很清晰，父类负责处理一些基础的网络代码，并且接受NSURLRequest 对象，而子类则负责处理和http协议有关的逻辑。


**类名** **继承关系** **主要用途** **适用场景** **AFURLSessionManager** 基类（直接基于 NSURLSession 封装） 管理 session 和 task，偏底层 下载、上传、自定义请求、复杂任务管理 **AFHTTPSessionManager** 继承自 AFURLSessionManager 封装了常见的 HTTP 请求方法 普通 HTTP 请求（GET/POST/PUT/DELETE），业务开发常用


## AFHTTPSessionManager


AFNetworking 中使用最高频率的是 AFHTTPSessionManager，负责各种 HTTP 请求的发起和处理，它继承自 AFURLSessionManager，是各种请求的直接执行者。


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/01-26266f25c6494449bbf26b50b01c34f8.png)


### 一次完整的GET请求过程


这里以 GET 为例，展示发起一次 GET 请求的具体过程。
 AFHTTPSessionManager 支持创建 GET、HEAD、POST、PUT、PATCH、DELETE 等请求，其中 GET 请求支持以下方法发起


```objective-c
- (NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(nullable id)parameters
                      headers:(nullable NSDictionary <NSString *, NSString *> *)headers
                     progress:(nullable void (^)(NSProgress * _Nonnull))downloadProgress
                      success:(nullable void (^)(NSURLSessionDataTask * _Nonnull, id _Nullable))success
                      failure:(nullable void (^)(NSURLSessionDataTask * _Nullable, NSError * _Nonnull))failure
{

    NSURLSessionDataTask *dataTask = [self dataTaskWithHTTPMethod:@"GET"
                                                        URLString:URLString
                                                       parameters:parameters
                                                          headers:headers
                                                   uploadProgress:nil
                                                 downloadProgress:downloadProgress
                                                          success:success
                                                          failure:failure];

    [dataTask resume];

    return dataTask;
}
#### 1. 初始化和配置


首先，要导入AFNetworking头文件


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/02-3daf3946218d4cd78a3cba270e91091c.png)


默认配置设置：


AFHTTPRequestSerializer请求序列化
 AFJSONResponseSerializer用于响应序列化


- 初始化Manager：


```objective-c
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
//本质上是调用：
[[AFHTTPSessionManager alloc] initWithBaseURL:nil sessionConfiguration:nil];
```


- 请求序列化器：

决定请求参数如何拼接/序列化


默认是AFHTTPRequestSerializer(普通HTTP表单)


```objective-c
manager.requestSerializer = [AFJSONRequestSerializer serializer]; //参数转为 JSON
manager.requestSerializer.timeoutInterval = 15; //超时时间
[manager.requestSerializer setValue:@"application/json" forHTTPHeaderField:@"Content-Type"]; //自定义Header
```


- 响应序列化器：

决定返回数据如何解析（默认是 AFJSONResponseSerializer，会把 JSON 自动转成 NSDictionary / NSArray。）


```objective-c
manager.responseSerializer = [AFJSONResponseSerializer serializer];
manager.responseSerializer.acceptableContentTypes = [NSSet setWithObjects:@"application/json", @"text/json", @"text/javascript", @"text/html", nil];
```


**AFHTTPSessionManager使用序列化器在本机对象和网络表示之间进行转换：**


requestSerializer属性（类型为AFHTTPRequestSerializer）将参数转换为URL查询字符串或HTTP正文内容
 responseSerializer属性（类型为AFHTTPResponseSerializer）将原始响应数据转换为可用的对象


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/03-d9eb45a475d5450baf7ce9d323777638.png)


 _默认情况下，AFHTTPSessionManager配置为：
 AFHTTPRequestSerializer请求序列化
 AFJSONResponseSerializer用于响应序列化_


也可以根据API要求自定义：


```objective-c
// For working with JSON APIs
manager.requestSerializer = [AFJSONRequestSerializer serializer];
manager.responseSerializer = [AFJSONResponseSerializer serializer];

// For XML services
manager.responseSerializer = [AFXMLParserResponseSerializer serializer];

// For image downloads
manager.responseSerializer = [AFImageResponseSerializer serializer];
#### 2. GET请求


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/04-b40054b9749a43b390c9469d87173a62.png)


 示例：提出GET请求：


```objective-c
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
manager.responseSerializer = [AFJSONResponseSerializer serializer];

// 发起 GET 请求
[manager GET:urlString parameters:nil headers:nil progress:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
    NSLog(@"数据如下: %@", responseObject);
    //接下来可以在这里处理返回的数据
} failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
    NSLog(@"错误: %@", error);
}];
```


> **URLString（NSString类型**）：表示要发送GET请求的URL字符串，即请求的目标地址。 **parameters（可选的id类型）**：包含GET请求的参数，这些参数会附加到URL字符串中，以便服务器可以根据这些参数返回相应的数据。它通常是一个NSDictionary或其他数据结构，其中包含键值对，表示请求参数。 **headers（可选的NSDictionary类型）**：包含HTTP请求头的字典。HTTP请求头通常包含与请求相关的信息，例如授权令牌、用户代理、接受的数据类型等。这里的参数允许你自定义请求头。 **downloadProgress（可选的NSProgress类型块）**：一个块对象，用于跟踪下载进度。这个块会在下载数据时被调用，可以用来更新UI或记录下载进度等。 **success（可选的块）**：一个成功回调块，当请求成功完成时会被调用。这个块通常接受两个参数，第一个参数是包含响应数据的NSURLSessionDataTask对象，第二个参数是响应数据，通常是一个NSDictionary或其他数据结构。 **failure（可选的块）**：一个失败回调块，当请求失败时会被调用。这个块通常接受两个参数，第一个参数是包含请求任务信息的NSURLSessionDataTask对象，第二个参数是一个NSError对象，包含了关于请求失败的信息。 在这个方法内部，首先通过调用**dataTaskWithHTTPMethod:URLString:parameters:headers:uploadProgress:downloadProgress:success:failure**:方法创建一个NSURLSessionDataTask对象，然后使用resume方法开始执行这个任务（发送GET请求），最后返回该任务对象，以便调用者可以对任务进行进一步操作或取消。


详细说说parameters：看着文字可能会一头雾水，我们来句一个例子
 在天气预报的请求中我们时常会遇到


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/05-d8601658bcce48cebfedb73f028b44f7.png)


 我们需要自己手动拼接字符串，而使用parameters后，我们可以

![请添加图片描述](./csdn-151627674-ios-afnetworking-images/06-5ba3798086b74cf8b97580d6176fe3c8.png)


 这里补充一个细节问题：APIURL是接口的基本路径，而parameters是传给接口的查询参数或请求体，AFN会自动把parameters序列化并拼接到URL或body，但是前提是必须有一个基础URL来承载这些参数。
 至于headers，会在下面讲解。


#### 3. 添加请求头的GET


在HTTP请求中，每个请求头都是一对 键值对，用来给服务器传递额外的信息


HTTP请求示例：


> GET /api/data?param1=value1 HTTP/1.1 Host: example.com Authorization: Bearer BQAG8… Content-Type: application/json App-Name: MyApp App-Key: 123456


里面的Authorization / Content - Type / App-Name / App- Key 都是请求头，他们不属于URL，也不属于请求体，只是告诉服务器如何请求


**Header 名称** **用途** Authorization 身份验证。比如 OAuth 的 Bearer token，告诉服务器你是谁，有访问权限。 Content-Type 告诉服务器请求体的数据类型，如 JSON、表单或 XML。 Accept 告诉服务器你希望返回什么类型的数据，例如 JSON。 User-Agent 告诉服务器客户端信息，例如浏览器类型、App 名称。 自定义 Header（如 App-Name, App-Key） 自定义数据传递，可用于验证 API Key 或标识请求来源。


```objective-c
-(void) getRequestWithHeader {
    NSString *urlString = @"http://";
    AFHTTPSessionManager *manager = [[AFHTTPSessionManager alloc] init];
    manager.responseSerializer = [AFJSONResponseSerializer serializer];
    [manager.requestSerializer setValue:(nullable NSString *) forHTTPHeaderField:(nonnull NSString *)];
    [manager GET:urlString parameters:nil headers:nil progress:^(NSProgress * _Nonnull downloadProgress) {
        NSLog(@"%@", downloadProgress);
    } success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
        NSLog(@"%@",responseObject);
    } failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
        NSLog(@"%@", error);
    }];
}
```


**Q1 : 前文的Manger基础设置中也能决定返回数据如何解析，和请求头里的Accept是什么关系呢？**


> Accept请求头，是高速服务器希望接收的数据类型。比如： application/json → 希望返回 JSON text/html → 希望返回 HTML application/xml → 希望返回 XML


> 而AFN的response Serializer决定了客户端如何解析服务器返回的数据： AFJSONResponseSerializer → 把返回数据解析成 NSDictionary / NSArray AFHTTPResponseSerializer → 只返回 NSData AFXMLParserResponseSerializer → 返回 NSXMLParser


总而言之：Accept决定服务器返回什么格式， 而response Serializer决定客户端如何解析这个格式


**Q2:前面说过，在GET请求的Headers是一个请求头的字典，为什么通过request Serializer设置请求头，而headers设置为nil？**


通过request Serializer设置的请求头，是给整个manager设置的默认请求头，之后这个manager发出的所有请求（GET / POST / PUT）都会带上这个请求头。
 而headers是在单个请求中穿headers参数，仅给这一条GET请求加请求头，和request Serializer不冲突，优先级更高


### POST请求


GET请求通常用于获取资源，不改变服务器状态；
 而POST请求多用于创建和修改资源。


> POST比GET通常会多一个请求体，POST中，参数放在请求体中，服务器通过body解析参数。 而GET的参数全部放在URL上，服务器通过URL解析参数。


```objective-c
#import <AFNetworking/AFNetworking.h>

- (void)doPostRequest {
    //请求 URL
    NSString *urlString = @"https://api.example.com/v1/login";
    //请求参数
    NSDictionary *parameters = @{
        @"username": @"Tom",
        @"password": @"123456"
    };
    //创建 manager
    AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
    manager.requestSerializer = [AFJSONRequestSerializer serializer];
    manager.responseSerializer = [AFJSONResponseSerializer serializer];

    //可以加请求头
    [manager.requestSerializer setValue:@"Bearer Aceess_Token"
                     forHTTPHeaderField:@"Authorization"];
    [manager.requestSerializer setValue:@"application/json"
                     forHTTPHeaderField:@"Accept"];

    //发起 POST 请求
    [manager POST:urlString
       parameters:parameters
          headers:nil
         progress:nil
          success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
              NSLog(@"请求成功: %@", responseObject);
          }
          failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
              NSLog(@"请求失败: %@", error);
          }];
}
```


## 举例：Spotify内容的获取


查看官方文档是很重要的一环。
 通过文档得知，我们通过API访问数据，需要一个访问令牌


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/07-71ed6e74b82644cbac6f7df322edf834.png)


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/08-261f14693ee24d89a37c4fa5422a8ccc.png)


由图可知，访问令牌需要我们获取ClientID和secret，此过程在网站内进行，当前跳过。
 • -X POST → HTTP POST 请求
 • -H → 设置请求头
 • -d → 请求体参数（form-urlencoded）
 我们按照步骤一步步完成


```objective-c
-(void)fetchAccessTokenWithClientID:(NSString *)clientID
                      clientSecret:(NSString *)clientSecret {
    NSString *urlString = @"https://accounts.spotify.com/api/token";

    NSDictionary *parameters = @{
        @"grant_type": @"client_credentials",
        @"client_id": clientID,
        @"client_secret": clientSecret
    };

    AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
    manager.requestSerializer = [AFHTTPRequestSerializer serializer];
    [manager.requestSerializer setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
    manager.responseSerializer = [AFJSONResponseSerializer serializer];

    [manager POST:urlString
       parameters:parameters
          headers:nil
         progress:nil
          success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
              NSString *accessToken = responseObject[@"access_token"];
              self.accessToken = accessToken;
              NSLog(@"Access Token: %@", accessToken);
          }
          failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
              NSLog(@"Error fetching access token: %@", error);
          }];
}
```


运行后得出结果：


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/09-c43016ca24f24e99807e5a2e34e7e9ae.png)


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/10-d8ffc093440446f8b482444d880cc0d6.png)


 在获取到accessToken后，我们在按照官网的流程，GET到对应信息。


```objective-c
- (void)fetchHotSongWithID:(NSString *)artistID
                   market:(NSString *)market {
    NSString *urlString = [NSString stringWithFormat:@"https://api.spotify.com/v1/artists/%@/top-tracks?market=%@", artistID, market];
    AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
    [manager.requestSerializer setValue:[NSString stringWithFormat:@"Bearer %@", self.accessToken] forHTTPHeaderField:@"Authorization"];
    [manager GET:urlString parameters:nil headers:nil progress:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
        NSLog(@"Hot songs info: %@", responseObject);
    } failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
        NSLog(@"Error fetching artist info: %@", error);
    }];
}
```


因为输入的是贾斯汀比伯的ID 最终我们成功请求到他的信息：


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/11-2f62865629e44ab0b90987806a36b358.png)


![请添加图片描述](./csdn-151627674-ios-afnetworking-images/12-3e371720eb534ff1a86dc654ef5b27a3.png)

---

原文发布于 CSDN：[【iOS】AFNetworking](https://blog.csdn.net/2402_86720949/article/details/151627674)
