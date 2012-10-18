---
layout: post
title: "learning ios programming1"
description: "save some experiences when learning progress"
category: ios
tags: [ios]
---
{% include JB/setup %}

## 1. error：Cannot assign to 'self' outside of a method in the init family##
原因：只能在init方法中给self赋值，Xcode判断是否为init方法规则：方法返回id，并且名字以init     +大写字母开头+其他  为准则。例如：- (id) initWithXXX;


{% highlight c++ %}
errorcode：
- (id) Myinit{
  self = [super init];
  ……
}

fixit：
- (id) initWithMy
{
  self = [super init];
}
{% endhighlight %}

## 2. Confusing feature *autolayout* in IOS 6##

I'm learning on standford courses CS193P. In lecture 8,there is a very simple sample about using imageview which embeded in scrollview for scrolling and zooming.It works perfect in IOS5 environment, but cann't work in IOS6, that was so annoying.

Implementation:

{% highlight c++ %}
#import "ImaginariumViewController.h"

@interface ImaginariumViewController ()
@property (weak, nonatomic) IBOutlet UIScrollView *scrollView;
@property (weak, nonatomic) IBOutlet UIImageView *imageView;
@end

@implementation ImaginariumViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    self.scrollView.contentSize = self.imageView.image.size;
    self.imageView.frame =
    CGRectMake(0, 0, self.imageView.image.size.width, self.imageView.image.size.height);
}

@end

{% endhighlight %}

I don't make it clearly, but i have found two to solve this problem.

### [1. override `- (void)viewDidAppear:(BOOL)animated`.](http://stackoverflow.com/questions/12619786/embed-imageview-in-scrollview-with-auto-layout-on-ios-6#)###

move `self.scrollview.contentsize = self.imageview.size` from viewDidLoad to viewDidAppear

### [2. uncheck 'use AutoLayout' in file inspector. ](http://stackoverflow.com/questions/11182008/uiscrollview-in-ios-6)###

RT


