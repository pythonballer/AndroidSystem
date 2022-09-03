# 小肩膀教育安卓定制系统
> 持续输出高质量课程，课程咨询小肩膀QQ：24358757。
```
小肩膀定制系统百度网盘地址：
链接：https://pan.baidu.com/s/1bfkY8MOKdWGpVRWa522C2A 
提取码：9822 
```

## 定制系统目前功能

* 脱壳
* 追踪函数调用
* smali trace
* 防止root检测
* 支持adb remount
* 支持直接替换系统文件
* WebView强制可调试
* 一定程度防VPN检测
* 一定程度防AOSP检测
* 默认开启USB调试
* Frida持久化

## 操作方法
### 1.整体加固脱壳
无需额外操作
打开app，只要dex有加载，就会保存在APP私有目录之下
```shell
/data/data/包名/xiaojianbang
```

### 2.抽取加固脱壳
需要额外开启。
创建目录 `/data/local/tmp/xiaojianbang/包名/saveDex`，即开启该功能
打开app等待一分钟自动开启主动调用，脱壳完成后 logcat中会显示 call run over
将app私有目录下 `/data/data/包名/xiaojianbang` 中的 xxx.dex和xxx.bin 文件拿出来
使用 dexfixer.jar 修复
```shell
java -jar dexfixer.jar xxx.dex xxx.bin out.dex
```
out.dex就是最终脱壳完毕的dex
脱壳完毕，记得删除 `/data/local/tmp/xiaojianbang/包名` 目录，否则每次打开都会脱壳

### 3.抽取加固脱壳 ---- 指定类
需要额外开启
创建文件 `/data/local/tmp/xiaojianbang/包名/saveDex/ClassName.txt`，即开启该功能
如果开启该功能，则该文件中必须有数据，否则会跳过所有类
文件内写入类的路径，支持多行，一行一个类即可
比如，要脱`com.xiaojianbang.app.Encrypt`类和`com.native.app.Test`类，文件内写入
```shell
com.xiaojianbang
com.native
或者
com
```
可以使用以下命令写入
```shell
echo com.xiaojianbang >> /data/local/tmp/xiaojianbang/包名/saveDex/ClassName.txt
echo com.native >> /data/local/tmp/xiaojianbang/包名/saveDex/ClassName.txt
```
写入的内容最后不要带空格，其余步骤与第二点相同

### 4.追踪函数调用
需要通过hook开启,
hook libc.so的strstr函数，当参数1为xiaojianbang_javaCall或者xiaojianbang_jniCall时，打印参数0和参数1即可，frida hook代码如下：
```javascript
function hook_strstr() {
    var libcModule = Process.getModuleByName("libc.so");
    var strstrAddr = libcModule.getExportByName("strstr");
    Interceptor.attach(strstrAddr, {
        onEnter: function (args) {
            this.arg0 = ptr(args[0]).readUtf8String();
            this.arg1 = ptr(args[1]).readUtf8String();
            if (this.arg1.indexOf("xiaojianbang_javaCall") != -1) {
                LogPrint(this.arg1 + "--" + this.arg0);
            }
            if (this.arg1.indexOf("xiaojianbang_jniCall") != -1) {
                LogPrint(this.arg1 + "--" + this.arg0);
            }
        }, onLeave: function (retval) {
             if (this.arg1.indexOf("javaCall") != -1) {
                 retval.replace(1);
             }
             if (this.arg1.indexOf("jniCall") != -1) {
                retval.replace(1);
             }
}
    });
}
```
### 5.smali trace
需要通过hook开启,
hook libc.so的strstr函数，当参数1为shouldTrace时，将函数返回值设置为1
当参数1为traceLog时，打印参数0和参数1即可，frida hook代码如下：
```javascript
function hook_strstr() {
    var libcModule = Process.getModuleByName("libc.so");
    var strstrAddr = libcModule.getExportByName("strstr");
    Interceptor.attach(strstrAddr, {
        onEnter: function (args) {
            this.arg0 = ptr(args[0]).readUtf8String();
            this.arg1 = ptr(args[1]).readUtf8String();
            if (this.arg1.indexOf("traceLog") != -1) {
               LogPrint(this.arg1 + "--" + this.arg0);
            }
        }, onLeave: function (retval) {
		    if (this.arg1.indexOf("shouldTrace") != -1) {
                retval.replace(1);
            }
		}
    });
}
```
### 6.抓包证书移动到系统目录
#### 6.1 将证书安装到用户目录
从Charles中保存根证书，这里将其保存为Charles.pem
将证书推送到手机sdcard目录
```shell
adb push Charles.pem /sdcard/
```
打开设置 --> 安全 --> 加密与凭据 --> 从存储设备安装
选择sdcard目录，然后点击Charles.pem证书安装
#### 6.2 移动证书到系统目录
依次运行以下adb命令，挂载system为可读写
```shell
adb root
adb disable-verity
adb reboot
```
待手机重启完毕后，运行以下命令
```shell
adb remount
```
移动用户目录证书到系统证书目录
```shell
mv -f /data/misc/user/0/cacerts-added/* /system/etc/security/cacerts
```
#### 6.3 删除用户凭据
打开设置 --> 安全 --> 加密与凭据 --> 用户凭据 --> 选择一个用户凭据删除
#### 6.4 证书效期
注意：证书是有效期的，Charles证书一般是一年，过了有效期，按上述方法重新操作
证书效期可按以下方法查看：  
打开设置 --> 安全 --> 加密与凭据 --> 信任的凭据 --> 系统 --> 点击一张证书

### 7.Frida持久化
需要额外配置开启
将GadgetConfig.js推送至 `/data/local/tmp/xiaojianbang/包名`，即开启该功能
GadgetConfig.js文件名称不可更改
GadgetConfig.js中的内容与Frida官网介绍的配置内容一致
Frida官网Gadget介绍地址：
`https://frida.re/docs/gadget/`
比如，GadgetConfig.js中的内容为：
```json
{
  "interaction": {
    "type": "script",
    "path": "/data/local/tmp/xiaojianbang.js"
  }
}
```
表示让Gadget在app运行时，直接加载hook脚本 `/data/local/tmp/xiaojianbang.js`
hook脚本路径及名称可自行指定
Gadget的so存放于 `/system/lib/libxiaojianbang.so` 和 `/system/lib64/libxiaojianbang.so`
可自行替换版本，支持官网Gadget运行所支持的任意模式
