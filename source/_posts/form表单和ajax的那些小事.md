---
title: form表单和ajax的那些小事
date: 2017-05-18 00:05:36
type: "tags"
tags: [前端]
---

> 学习的道路就是不停的踩坑...

<!--more-->

#### form表单点击button，ajax提交自动刷新

话不多说，直接上代码
```
<form>
    //省略一堆代码
    <div class="hr-line-dashed"></div>
        <div class="form-group">
            <div class="col-sm-4 col-sm-offset-2">
            <button class="btn btn-primary"  onclick="submitAcc()">保存内容</button>
            <button class="btn btn-white">取消</button>
        </div>
    </div>
</form>

<script>
function submitAcc() {
        $.ajax({
            type: "POST",
            url: "/backend/account",
            data: $('#accAdd').serialize(),
            success: function (result) {
                if(result.success==true){
                    swal({title: "你的小账本~", text: result.msg, type: "success"}, function(){
                        //跳转至列表界面
                        location.href = "/backend/account/list";
                    });
                }else {
                    swal({title: "你的小账本~", text: result.msg, type: "error"}, function(){
                        //刷新
                        location.reload();
                    });
                }
            },
            error: function (result) {
                swal({title: "你的小账本~", text: result.msg, type: "error"}, function(){
                    //刷新
                    location.reload();
                });
            }
        });
    }
</script>
```

本来就是想一个添加数据的功能，每次点击提交后硬是给我刷新页面，搞的头晕脑胀。后来各种尝试，
发现form中存在button标签时，用ajax异步提交表单就会刷新。

原因：button 存在时会再次提交一下表单，所以页面被刷新了。（之前认为button type='submit' 时）button才有提交表单的功能。

解决：button标签换成a标签就可以了，或者不使用form。

当时犯傻的地方   [已修正后的页面](https://github.com/7le/shine/blob/1.0-dev/web/WEB-INF/views/backend/account/accountAdd.ftl) ^.^

#### ajax使用FormData对象上传文件

首先给大家推荐一个[图床](https://sm.ms/)。

在对接接口的时候，需要使用FormData上传，于是就想办法解决了下。

下面是关键的代码
```
<form id="uploadForm" enctype="multiple/form-data" >
    <input id="smfile" name="smfile" type="file" multiple >
    <input type="button" onclick="send()" value="send">
</form>
```

```
<script type="text/javascript">
function send() {
 
 var formData = new FormData($('#uploadForm')[0]);

 //console.log($('#uploadForm')[0]);
 
 formData.append('format','json');

 $.ajax({
        type:'post',
        url: "https://sm.ms/api/upload",
        data: formData,
        processData: false,
        contentType: false,
        success: function (data) {
            console.log(data);
        }
    });
}
</script>
```
最后补充几点

* 1、input中声明的是什么，接收的时候就用什么。例如上面的就是smfile
* 2、processData设置为false。因为data值是FormData对象，不需要对数据做处理。
* 3、form标签添加enctype="multipart/form-data"属性。
* 4、cache设置为false，上传文件不需要缓存。contentType设置为false。因为是由form表单构造的FormData对象，且已经声明了属性enctype="multipart/form-data"，所以这里设置为false。

完。^.^

[更多精彩 戳我](http://7le.top)