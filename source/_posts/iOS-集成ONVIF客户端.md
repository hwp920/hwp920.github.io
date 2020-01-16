title: iOS 集成ONVIF客户端
author: Cyrus
tags:
  - ONVIF
categories: []
date: 2020-01-16 23:10:00
---
### 一、编译openssl
最简单的方法，github上找一个脚本，如：https://github.com/gitusrs/openssl-ios-build-shell-script 下载openssl源码，根据版本，改一下脚本的** OPENSSL_COMPRESSED_FN="openssl-1.0.2e.tar.gz" ** 与源码版本相对应，运行一下，就ok了。

### 二、创建OC项目
1、将上篇文章生成的 onvif c语言文件及封装的OnvifDevice和OnvifControl 文件复制到项目中，将上面编译成的openssl静态库，设置一下头文件和静态库的路径，如下图：
![](onvif_oc_1.png)

#### 1、搜索设备
搜索设备，把设备地址放到数组并展示到列表中：
~~~
- (IBAction)searchDeviceAction:(id)sender {
    ONVIF_DetectDevice(cb_discovery, (__bridge void *)self);
}

void cb_discovery(void *userinfo, struct wsdd__ProbeMatchesType *wsdd__ProbeMatches)
{
    if (NULL == wsdd__ProbeMatches) {
        return;
    }
    ViewController *vc = (__bridge ViewController *)(userinfo);
    [vc gotDeviceAddrs:wsdd__ProbeMatches];
}

- (void)gotDeviceAddrs:(struct wsdd__ProbeMatchesType *)wsdd__ProbeMatches
{
    for(int i = 0; i < wsdd__ProbeMatches->__sizeProbeMatch; i++) {
        struct wsdd__ProbeMatchType *probeMatch = wsdd__ProbeMatches->ProbeMatch + i;
        printf("%s\n", probeMatch->XAddrs);
        [_addrs addObject:[NSString stringWithUTF8String:probeMatch->XAddrs]];
    }
    [self.tableView reloadData];
}
~~~
![](onvif_oc_2.png)


#### 2、连接设备并进行相关操作
~~~
- (IBAction)connectButtonAction:(id)sender {
    _device = createDevice(_addr.UTF8String, _nameTf.text.UTF8String, _passwdTf.text.UTF8String);
    
    int result = ONVIF_GetCapabilities(_device);
    if (result != SOAP_OK) {
        NSLog(@"ONVIF_GetCapabilities fail");
        return;
    }
    
    result = ONVIF_GetProfile(_device);
    if (result != SOAP_OK) {
        NSLog(@"ONVIF_GetProfile fail");
        return;
    }
    
    // 播放，简单起见，直接用vlc
    VLCViewController *vlcVC = [[VLCViewController alloc] init];
    vlcVC.device = self.device;
    [self.navigationController pushViewController:vlcVC animated:YES];
}
~~~
![](onvif_oc_3.png)
![](onvif_oc_4.png)

#### 3、播放
~~~
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view from its nib.
    
    _player = [[VLCMediaPlayer alloc] init];
    _player.drawable = self.playView;
    
    [self queryStreamURI];
    
    self.navigationItem.rightBarButtonItem = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemSearch target:self action:@selector(infoButtonAction:)];
}

- (void)queryStreamURI
{
	// ipc 有两个profile，对应不同分辨率的配置，这里用按钮是否选中区别分辨率
    char uri[128] = {0};
    ONVIF_GetStreamUri(_device, _uriButton.selected, uri, sizeof(uri));
    [self playWithURI:[NSString stringWithUTF8String:uri]];
}

- (void)playWithURI:(NSString *)uri
{
    if (_player.playing) {
        [_player stop];
    }
    
    VLCMedia *media = [[VLCMedia alloc] initWithURL:[NSURL URLWithString:uri]];
    [_player setMedia:media];
    [_player play];
}

#pragma mark - user interface
// 方向控制，4个按钮分别对应0-3
- (IBAction)directionButtonAction:(UIButton *)sender {
    ONVIF_PTZRelativeMove(_device, sender.tag);
}

- (IBAction)switchUriStream:(UIButton *)sender {
    sender.selected = !sender.selected;
    [self queryStreamURI];
}
~~~
![](onvif_oc_5.png)
![](onvif_oc_6.png)