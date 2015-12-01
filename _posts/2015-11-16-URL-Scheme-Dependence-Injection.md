---
layout: post
title:  使用URL Scheme方式减少ViewController间依赖
date:   2015-11-16 14:30:00
---

------

# 引  #

一直以来，我们在iOS中进行视图控制器跳转都习惯于使用`pushViewController:animated`方式。但这种方式存在一个弊端，就是ViewControllerA中混杂了ViewControllerB的代码，或者说ViewControllerA依赖了ViewControllerB，如：  

```Objective-C
- (void)skipToNextViewController
{
    UREFirstViewController *viewController = [UREFirstViewController new];
    [self.navigationController pushViewController:viewController animated:YES];
}
```

解决代码间依赖常见的方式可使用依赖注入（DI）解决，但DI在OC中使用的不是很多。另外值得一提的是苹果为了解决应用间跳转引入了URL Scheme，其初衷是通过类似HTTP URL的方式调起指定APP并传入参数，由目标APP根据参数来处理后续逻辑。  

基于此，一些开发者便开源了一些框架来做应用间的ViewController跳转，已知的有`JLRoute`、`HHRoute`以及`DeepLinkKit`。其中DeepLinkKit是新近诞生的，其不仅支持基于Block的路由策略还支持复杂的基于类的路由，同时已支持iOS 9引入的`universal links`，因此当下DeepLinkKit拥有较多优势。  

# 实战  

首先，我们新建URLRouteExample工程并使用CocoaPods进行依赖管理：  

```
# Uncomment this line to define a global platform for your project
platform :ios, '7.0'
# Uncomment this line if you're using Swift
# use_frameworks!

target 'URLRouteExample' do
	pod "Masonry"
	pod "DeepLinkKit"
end

target 'URLRouteExampleTests' do

end

target 'URLRouteExampleUITests' do

end
```

执行`pod install/update`后打开`URLRouteExample.xcworkspace`即可。  

我们在ViewController之外新建两个ViewController用于演示：`UREFirstViewController`、`URESecondViewController`。可以在`AppDelegate`中初始化并添加`DeepLinkKit`的路由逻辑，但当项目逐渐膨胀之后此法便不再可取，因此这里新建一个`URERoutes`类用于处理路由逻辑。  

## 路由  

在路由配置部分根据ViewController className的SHA1进行比对以确定最终的路由路径。  

```Objective-C
- (void)configURLPath
{
    __weak URERoutes *weakSelf = self;
    self.deepLinkRoute[@"/viewController/:hashString"] = ^(DPLDeepLink *link) {
        __strong URERoutes *strongSelf = weakSelf;
        NSString *hashString = link.routeParameters[@"hashString"];
        NSString *viewControllerTitle = link.queryParameters[@"title"];
        NSInteger colorType = [link.queryParameters[@"colorType"] integerValue];
        NSArray *colorTypes = @[[UIColor whiteColor], [UIColor orangeColor], [UIColor redColor], [UIColor greenColor]];
        if (colorType >= [colorTypes count] || colorType < 0) {
            colorType = [colorTypes count] - 1;
        }
        NSLog(@"%@\t%@\t", link.URL, hashString);
        if ([hashString isEqualToString:[[NSStringFromClass([URESecondViewController class]) SHA1] uppercaseString]]){
            // SHA1:6D2FB4B73E4927873EEF0A78DF96863FC3C83878
            URESecondViewController *viewController = [URESecondViewController new];
            viewController.title = [NSString stringWithFormat:@"SecondViewController-%@", viewControllerTitle];
            viewController.view.backgroundColor = colorTypes[colorType];
            [strongSelf showViewController:viewController];
        } else if ([hashString isEqualToString:[[NSStringFromClass([UREFirstViewController class]) SHA1] uppercaseString]]) {
            // SHA1:4658CF27F926A5EFB3A70E28FCC1906E4D751335
            NSNumber *hashID = link.queryParameters[@"hashID"];
            UREFirstViewController *viewController = [UREFirstViewController new];
            viewController.hashID = hashID;
            viewController.title = [NSString stringWithFormat:@"FirstViewController-%@", viewControllerTitle];
            viewController.view.backgroundColor = colorTypes[colorType];
            [strongSelf showViewController:viewController];
        }
    };
}
```

另外在`AppDelegate`中相应的代理方法中调用`DeepLinkKit`的路由即可（这里是`URERoutes`的单例）：  

```Objective-C
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
{
    return [[URERoutes shardInstance].deepLinkRoute handleURL:url withCompletion:NULL];
}

- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary*)options
{
    return [[URERoutes shardInstance].deepLinkRoute handleURL:url withCompletion:NULL];
}

- (BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity restorationHandler:(void (^)(NSArray * _Nullable))restorationHandler
{
    return [[URERoutes shardInstance].deepLinkRoute handleUserActivity:userActivity withCompletion:NULL];
}
```

另外为了方便各ViewController的跳转需要获取当前可是界面最上层的ViewController以确定是```pushViewController```还是```presentViewController```。  

最后值得一提的是，由于通过URL Scheme方式各ViewController变得完全透明，因此以往的基于`delegate`方式的ViewController回传数据已不再可用，演示代码中通过`NSNotificationCenter`的方式由`UREFirstViewController`向`URESecondViewController`传值，由于是Demo性质的因此权当作一个引子，关于`NSNotificationCenter`的一对多通知的处理还需完善；当然也可通过`NSUserDefaults`的方式来透传。  

## 更新

由于`NSNotificationCenter`的跨层弊端，目前已改为常用的`Delegate`的方式了。其关键点在于获取Objective-C对象在内存中的地址并将其转换为对应的对象：

```Objective-C
uintptr_t hex = strtoull(self.memAddress.UTF8String, NULL, 0);
id object = (__bridge id)((void*)hex);
if ([object conformsToProtocol:@protocol(UREViewControllerDelegate)] && [object respondsToSelector:NSSelectorFromString(@"justLog:")]) {
    [object performSelector:NSSelectorFromString(@"justLog:") withObject:[NSString stringWithFormat:@"%d", arc4random_uniform(100)]];
}
```

完整的例子可从[这里](https://github.com/LuoLee/URLRouteExample)获取。  
