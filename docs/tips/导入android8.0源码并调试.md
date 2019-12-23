导入android8.0源码并调试

假设已经编译好android源码，下面执行以下四个步骤

1. "source build/envsetup.sh" (source可以用 . 代替，即". build/envsetup.sh")

2. "lunch"，并选择要编译的项目(aosp_arm64-eng我选择的)

3. "make idegen -j4" (这里的 -j4 表示用4线程来编译，可以不加)

4. "sudo development/tools/idegen/idegen.sh" (我的电脑需要管理员权限才能执行成功，所以我一般会在前面加上"sudo")

完成上述四个步骤后会生成如下三个文件

完成以上四个步骤之后，会发现在源码根目录下出现了三个新的文件(也有可能是两个)

    1. android.iml (记录项目所包含的module、依赖关系、SDK版本等等，类似一个XML文件)
    2. android.ipr (工程的具体配置，代码以及依赖的lib等信息，类似于Visual Studio的sln文件)
    3. android.iws (主要包含一些个人的配置信息，也有可能在执行上述操作后没有生成，这个没关系，在打开过一次项目之后就会自动成了)
 "android.iml"和"android.ipr"一般是"只读"的属性，我们这里建议大家，把这两个文件改成可读可写，否则，在更改一些项目配置的时候可能会出现无法保存的情况，执行如下两条命令即可。

```
sudo chmod 777 android.iml
sudo chmod 777 android.ipr
```

如果你的电脑性能足够好(内存大于16G，代码下载在SSD上)，那么可以直接打开Android Studio，点击"Open an existing Android Studio project"选项，找到并选中刚刚生成的"android.ipr"文件，点击OK，就可以开始导入项目了。 第一次导入，这个过程可能会持续很久，几十分钟或者超过一个小时。不过成功之后，以后再打开项目就会快很多了。
 如果电脑性能一般的话，我建议，可以在导入项目前，手动对"android.iml"文件进行一下修改，可以使我们导入的时间尽可能的缩短一些。

首先，要保证"android.iml"文件已经添加了"可写入"的属性(上文中已经介绍了如何修改文件属性)。
接下来，使用文本编辑器打开"android.iml"文件，并执行以下修改(仅代表我的个人习惯，也可以根据同学们的喜好自己调整)：

1. 搜索关键字"orderEntry"，我一般会将所有带有这个关键字的标签项全部删除，仅保留以下三行，大概如下

```
......
    </content>
    <orderEntry type="sourceFolder" forTests="false" />
    <orderEntry type="inheritedJdk" />
    <orderEntryProperties />
  </component>
</module>
```

2. 搜索”excludeFolder“关键字，对这里进行一些修改，将我们不需要看的代码Exclude掉。通过这个步骤，能极大地提升第一次加载项目的速度。

    等项目加载完成后，我们还可以通过Android Studio对Exclude的Module进行调整，所以也不用害怕这里Exclude掉了有用的代码，或少Exclude了一部分代码，在项目加载完以后再进行调整就行了。

    以下是我的配置，大家可以参考(由于我比较关注Framework以及Telephony相关的代码，所以重点保留了这两部分，而其他一些如kernel、bootloader的代码，我就Exclude掉了，同学们也可以根据自己的需求来进行修改)。

    ```
    <excludeFolder url="file://$MODULE_DIR$/.repo" />
    <excludeFolder url="file://$MODULE_DIR$/art" />
    <excludeFolder url="file://$MODULE_DIR$/bionic" />
    <excludeFolder url="file://$MODULE_DIR$/bootable" />
    <excludeFolder url="file://$MODULE_DIR$/build" />
    <excludeFolder url="file://$MODULE_DIR$/compatibility" />
    <excludeFolder url="file://$MODULE_DIR$/dalvik" />
    <excludeFolder url="file://$MODULE_DIR$/developers" />
    <excludeFolder url="file://$MODULE_DIR$/developers/samples" />
    <excludeFolder url="file://$MODULE_DIR$/development" />
    <excludeFolder url="file://$MODULE_DIR$/device/google" />
    <excludeFolder url="file://$MODULE_DIR$/device/sample" />
    <excludeFolder url="file://$MODULE_DIR$/docs" />
    <excludeFolder url="file://$MODULE_DIR$/external" />
    <excludeFolder url="file://$MODULE_DIR$/flashing-files" />
    <excludeFolder url="file://$MODULE_DIR$/frameworks/base/docs" />
    <excludeFolder url="file://$MODULE_DIR$/kernel" />
    <excludeFolder url="file://$MODULE_DIR$/libcore" />
    <excludeFolder url="file://$MODULE_DIR$/libnativehelper" />
    <excludeFolder url="file://$MODULE_DIR$/out" />
    <excludeFolder url="file://$MODULE_DIR$/pdk" />
    <excludeFolder url="file://$MODULE_DIR$/platform_testing" />
    <excludeFolder url="file://$MODULE_DIR$/prebuilt" />
    <excludeFolder url="file://$MODULE_DIR$/prebuilts" />
    <excludeFolder url="file://$MODULE_DIR$/shortcut-fe" />
    <excludeFolder url="file://$MODULE_DIR$/test" />
    <excludeFolder url="file://$MODULE_DIR$/toolchain" />
    <excludeFolder url="file://$MODULE_DIR$/tools" />
    ```

    完成之后，按照上面说的步骤，使用Android Studio选中"android.ipr"打开项目即可。

    1. 选择SDK版本

    点击"File -> Project Structure..."，中间的窗口选择"android"(首字母小写的那一个)，在弹出的窗口中左边栏中选择"Modules"，而后在右边的窗口中选择"Dependencies"。在下拉菜单中选择系统源代码相应的SDK版本(如：8.0的代码就选择API 26，9.0的版本就选择API 28)。

    ![](..\tips\image\选择SDK版本.png)

 如果在下拉菜单中没有找到相应的SDK版本，就打开Android Studio自带的SDK Manager下载即可。

![](..\tips\image\选择SDK版本.png)

**PS: 这步非常关键，如果这里不选择Android API，而是使用JDK 1.8之类的话，无法进行系统源码的单步调试。**

2. 指定项目的minsdkversion

在阅读源代码的时候，经常会看到类似的这种错误"Field requires API level xx (current min is 1): android.xx.xx"，这是由于我们只对项目指定了targetSdkVersion，但没有指定minSdkVersion。

解决办法如下：

点击"File -> Project Structure..."，在弹出的窗口中左边栏中选择"Modules"，中间的窗口选择"Android"(首字母大写的那一个)，而后在右边的窗口中选择"Structure"。如下图所示，将这三行配置改为你自己的代码目录即可(不一定非要使用这个AndroidManifest.xml文件以及res和assets目录，你可以选择你喜欢的任意一个)，完成后点击Apply或者OK。



![](..\tips\image\选择SDK版本.png)

接下来，找到刚刚选择的那个AndroidManifest.xml，打开并在manifest标签下的任意一行添加如下代码并保存即可

```
<uses-sdk android:minSdkVersion="26" />
```

 这里的数字根据你源代码的版本来填写，比如你导入的是8.0的源代码，就写26；9.0的源代码就写28，以此类推。由于是系统源代码，所以minSdkVersion与前面第2步中设置的Android API版本保持一致即可。

3. 启动模拟器
4. debug到指定的线程

![](..\tips\image\debug到指定的线程.png)

5. 成功

![](..\tips\image\成功.png)

6. 进入断点

![](..\tips\image\进入断点.png)

