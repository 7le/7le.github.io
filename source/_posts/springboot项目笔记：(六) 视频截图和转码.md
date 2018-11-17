---
title: springboot项目笔记：(六) 视频截图和转码
date: 2017-06-21 23:50:08
type: "tags"
tags: [springboot]
---

> 项目是一个简单的视频点播系统，涉及到了视频的截图和转码，所用到的工具是ffmpeg，遇到了些坑，做一个完整的记录，利人利己 ^.^

<!--more-->

### windows

先说说在windows下如何使用ffmpeg，下面是截图视频图片的代码

```
public class VideoUtil {

    //截取视频第一帧windows   
    public static boolean processImgWindows(String video_path,String ffmpeg_path) {
        File file = new File(video_path);
        if (!file.exists()) {
            System.err.println("路径[" + video_path + "]对应的视频文件不存在!");
            return false;
        }
        String imgName=System.currentTimeMillis()+".jpg";
        List<String> commands = new java.util.ArrayList<String>();
        commands.add(ffmpeg_path);
        commands.add("-i");
        commands.add(video_path);
        commands.add("-y");
        commands.add("-f");
        commands.add("image2");
        commands.add("-ss");
        commands.add("1");//这个参数是设置截取视频多少秒时的画面
        //commands.add("-t");
        //commands.add("0.001");
        commands.add("-s");
        commands.add("800x600");
        commands.add("/software/video/"+imgName);
        try {
            ProcessBuilder builder = new ProcessBuilder();
            builder.command(commands);
            builder.start();
            System.out.println("截取成功");
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
}
```
其中ffmpeg_path就是ffmpeg.exe的路径，[ffmpeg.exe链接](https://github.com/7le/video/blob/master/ffmpeg.exe) 可以直接下过去用，经过测试完全ok。

### Linux

Linux下使用ffmpeg比较麻烦点，我们需要先去下载 [ffmpeg.tar.bz2链接](https://github.com/7le/video/blob/master/ffmpeg-3.3.tar.bz2) 

将包放到linux下解压，

>进入ffmpeg解压目录：cd ffmpeg/
进行配置：./configure --enable-shared --prefix=/usr/local/ffmpeg
其中：--enable-shared 是允许其编译产生动态库，在以后的编程中要用到这个几个动态库。--prefix设置的安装目录。
编译并安装：make && make install


如果需要需要安装yasm 就如下操作
>wget http://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz
tar zxvf yasm-1.3.0.tar.gz
cd yasm-1.3.0
./configure
make && make install

安装完成以后并不能直接使用 ffmpeg 命令执行，系统会提示并没有这样的命令，需要进一步进行配置Path：
>编辑profile文件：
       vi /etc/profile
       i （插入）
       在文件末尾加上两句话：
       export FFMPEG_HOME=/usr/local/ffmpeg 
       export PATH=$FFMPEG_HOME/bin:$PATH
       保存并退出：按Esc键 输入:wq! 回车
使修改生效：source /etc/profile

我启动的时候报了error while loading shared libraries:libavdevice.so.5x错误，x是什么数字我具体忘了。
解决的方案是
>修改ld.so.conf：vi /etc/ld.so.conf
       在末尾加上一句话：/usr/local/ffmpeg/lib
       保存并退出：按Esc键 输入:wq! 回车
       使修改生效：ldconfig -v

成功的话 就会有如下的显示
![成功](https://github.com/7le/7le.github.io/raw/master/image/springboot/springboot-6-1.png)

安装完毕后，就可以直接上代码了
```
public class VideoUtil {

    public static String processImgLinux(String video_path,String ffmpeg_path) throws Exception {

        File file = new File(video_path);
        if (!file.exists()) {
            throw new Exception("路径[" + video_path + "]对应的视频文件不存在!");
        }
        String imgName=System.currentTimeMillis()+".jpg";
        List<String> commands = new java.util.ArrayList<String>();
        commands.add(ffmpeg_path);
        commands.add("-i");
        commands.add(video_path);
        commands.add("-y");
        commands.add("-f");
        commands.add("image2");
        commands.add("-ss");
        commands.add("1");//这个参数是设置截取视频多少秒时的画面
        //commands.add("-t");
        //commands.add("0.001");
        commands.add("-s");
        commands.add("800x600");
        commands.add("/software/video/img/"+imgName);
        StringBuffer test=new StringBuffer();
        for(int i=0;i<commands.size();i++)
            test.append(commands.get(i)+" ");
        System.out.println(test);
        try {
            Runtime rt = Runtime.getRuntime();
            Process proc = rt.exec(test.toString());
            InputStream stderr = proc.getErrorStream();
            InputStreamReader isr = new InputStreamReader(stderr);
            BufferedReader br = new BufferedReader(isr);
            String line = null;
            while ( (line = br.readLine()) != null);

        } catch (IOException e) {
            throw new Exception("截图图片失败");
        }
        return imgName;
    }

    /**
     * Linux 转码
     * @param video_path 视频路径
     * @param ffmpeg_path ffmpeg路径
     * @param type  要转换的类型
     * @return ./ffmpeg -i /opt/spzh/yysp.avi -f mp4 -acodec libfaac -vcodec libxvid -qscale 7 -dts_delta_threshold 1 -y /opt/spzh/out/yysp8.mp4
     * @throws Exception
     */
    public static String transcodingByLinux(String video_path, String ffmpeg_path, String type) throws Exception {
        File file = new File(video_path);
        if (!file.exists()) {
            throw new Exception("路径[" + video_path + "]对应的视频文件不存在!");
        }
        String videoName=System.currentTimeMillis()+"."+type;
        List<String> commands = new java.util.ArrayList<String>();
        commands.add(ffmpeg_path);
        commands.add("-i");
        commands.add(video_path);
        commands.add("-f");
        commands.add("mp4");
        commands.add("-acodec");
        commands.add("libfaac");
        commands.add("-vcodec");
        commands.add("libxvid");
        commands.add("-qscale");
        commands.add("7");
        commands.add("-dts_delta_threshold");
        commands.add("-y");
        commands.add("/software/video/"+videoName);
        StringBuffer test=new StringBuffer();
        for(int i=0;i<commands.size();i++)
            test.append(commands.get(i)+" ");
        System.out.println(test);
        try {
            Runtime rt = Runtime.getRuntime();
            Process proc = rt.exec(test.toString());
            InputStream stderr = proc.getErrorStream();
            InputStreamReader isr = new InputStreamReader(stderr);
            BufferedReader br = new BufferedReader(isr);
            String line = null;
            while ( (line = br.readLine()) != null);

        } catch (IOException e) {
            throw new Exception("截图图片失败");
        }
        return videoName;
    }
}
```

### java.lang.ProcessBuilder

不知道大家是不是会好奇，java是怎么调用到ffmpeg。其实是使用了ProcessBuilder类，这个类平时用的不对。主要是用来执行本地命令或脚本等工作。那大家就自然明白了，ffmpeg是如何运行了。

上面的demo中，我分别用了ProcessBuilder 和 Process，都实现了执行本地命令或脚本的工作。一个是ProcessBuilder.start()，另一个是Runtime.exec() 方法都被用来创建一个操作系统进程（执行命令行操作）。

有兴趣的朋友可以跟到源码中看下，Process是一个抽象类，而ProcessBuilder是一个final类。我们发现ProcessBuilder比Process更加丰富，通过构造方法就可以直接创建ProcessBuilder的对象。而Process是一个抽象类，一般都通过Runtime.exec()和ProcessBuilder.start()来间接创建其实例。

下面是关于ProcessBuilder和Process的区别：
* 1、命令：是一个字符串列表，它表示要调用的外部程序文件及其参数（如果有）。在此，表示有效的操作系统命令的字符串列表是依赖于系统的。例如，每一个总体变量，通常都要成为此列表中的元素，但有一些操作系统，希望程序能自己标记命令行字符串——在这种系统中，Java 实现可能需要命令确切地包含这两个元素。 （在我们上面的demo中就可以看出命令的不同）
* 2、环境：是从变量 到值 的依赖于系统的映射。初始值是当前进程环境的一个副本（请参阅 System.getenv()）。
* 3、工作目录：默认值是当前进程的当前工作目录，通常根据系统属性 user.dir 来命名。
* 4、redirectErrorStream属性：最初，此属性为false，意思是子进程的标准输出和错误输出被发送给两个独立的流，这些流可以通过 Process.getInputStream() 和Process.getErrorStream() 方法来访问。如果将值设置为true，标准错误将与标准输出合并。这使得关联错误消息和相应的输出变得更容易。在此情况下，合并的数据可从Process.getInputStream(）返回的流读取，而从Process.getErrorStream()返回的流读取将直接到达文件尾。所以就有，ProcessBuilder为进程提供了更多的控制。

最后要注意的就是ProcessBuilder是线程不安全的~

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)