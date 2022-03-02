---
layout: post
title:  "jquery异步提交数据"
date:   2019-07-20 16:01:01 +0800
categories: [Tech]
tag: 
  - Jquery
  - Web
---

### JQuery提交普通表单,使用serialize()

使用`fastjson.jar`对返回的数据转换成json数据

form表单:

```html
<form id="form">
    <input type="number" name="page_init" id="page_init" min="0" step="1" max="26">
    <input type="number" name="page_end" id="page_end" min="0" step="1" max="26">
    <input type="button" id="removeButton" value="remove" >
</form>
```

JQuery提交数据:

```js
$.ajax({
    url: '<%= request.getContextPath() %>/pdfRemove',
    type: 'post',
    cache: false,
    data: $("#form").serialize(),
    dataType: "json",
    processData: false,

    success: function(data){
        alert(data.status);
    },
    error: function(err){
        alert("error");
    }
});
```

注意:由于使用FormData函数处理了数据,所以`processData`必须设置为`false`.`processData`的含义为处理数据,默认为`true`,将发送的数据将被转换为对象.

不知为何FormData函数处理普通表单,根本发送不出去...

Servlet处理数据:

```js
JSONObject jsonObject = new JSONObject();
request.setCharacterEncoding("utf-8");
HttpSession se = request.getSession();
String path = (String) se.getAttribute("filePath");
int page_init = Integer.valueOf(request.getParameter("page_init"));
int page_end = Integer.valueOf(request.getParameter("page_end"));
System.out.println("path:"+path+",page_init:"+page_init+",page_end:"+page_end);

//Pdf.remove(new File(path), page_init, page_end);
jsonObject.put("status","成功");
System.out.println(jsonObject.toString());
response.getOutputStream().write(jsonObject.toString().getBytes("utf-8"));
```

### JQuery提交带有文件的表单,使用FormData处理数据

这里使用了`commons-fileupload.jar`来处理接收的数据,使用`fastjson.jar`对返回的数据转换成json数据

form表单:

```html
<form>
    <input type="file" name="file" id="file" >
    <input type="button" id="button" value="submit" >
</form>
```

JQuery提交数据:

```js
var formData = new FormData();
formData.append('file', $('#file')[0].files[0]);
$.ajax({
    url: '',
    type: 'post',
    cache: false,
    processData: false,
    contentType: false,
    data: formData,
    dataType: "json",
    success: function(data) {
        if (data[0].errorFileType == ""){
            $("#page_end").attr("value", data[0].count);
            $("#page_end").attr("max", data[0].count);
            $("#page_init").attr("max", data[0].count);
            $("#removeButton").removeAttr("style");
        } else {
            alert(data[0].errorFileType);
        }
    },
    error: function(){
        alert('未知错误');
    }
});
```

`$('#file')[0].files[0]` 这个似乎是最后一个`[0]`代表第几个文件,多文件时要注意,具体等以后遇到再说.

当将contentType选项设置为false时，会强制jQuery不要添加Content-Type头，否则边界字符串将会丢失。

Servlet处理数据:

```js
if (ServletFileUpload.isMultipartContent(request)) {
    ServletFileUpload upload = new ServletFileUpload(new DiskFileItemFactory());
    upload.setHeaderEncoding("UTF-8");
    List<FileItem> fileItemList;
    try {
        fileItemList = upload.parseRequest(request);
        for(FileItem fileItem: fileItemList) {
            if (fileItem.isFormField()) {
                System.out.println(fileItem.getFieldName() + "," + fileItem.getString());
            } else {
                System.out.println(fileItem.getFieldName());
                String path = this.getInitParameter("path") + fileItem.getName();
                int pages = Pdf.calculatePages(fileItem.getInputStream());

                File file = new File(path);
                if (!file.exists()) {
                    fileItem.write(file);
                }

                HttpSession session = request.getSession(true);
                session.setAttribute("filePath", path);


                response.setCharacterEncoding("utf-8");
                JSONObject jsonObject = new JSONObject();
                jsonObject.put("count", pages);
                jsonObject.put("errorFileType", "");

                JSONArray jsonArray = new JSONArray();
                jsonArray.add(jsonObject);
                System.out.println(jsonArray.toJSONString());
                response.getOutputStream().write(jsonArray.toString().getBytes("utf-8"));
            }
        }
    } catch (FileUploadException e) {
        e.printStackTrace();
    } catch (Exception e) {
        e.printStackTrace();
   }
} else {

}
```

### JQuery提交表单,带有普通表单和文件,使用序列化

JQuery提交数据:

```js
var formData = new FormData();
formData.append('file', $('#file')[0].files[0]);
formData.append('name', $('#name').val());
formData.append('artist', $('#artist').val());
$.ajax({
    url: '${pageContext.request.contextPath}/upload',
    type: 'post',
    cache: false,
    processData: false,
    contentType: false,
    data: formData,
    success: function(data) {
        alert("成功")
    },
    error: function(){
        alert('未知错误');
    }
});
```
