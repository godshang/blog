---
title: '使用OpenOffice将word转换为pdf'
layout: post
categories: 技术
tags:
    - Java
---

## 引言 ##

上一篇文章中，介绍了如何使用模板语言生成Word 2007格式的文档。这篇文章更进一步，介绍如何将word文档转换为pdf文档。

将word转换为pdf的方法也有好多，但考虑到平台的问题，真正可行的也就只有OpenOffice这条路了。。

## 技术实现 ##

### 需要用到的软件 ###

- OpenOffice 下载地址：http://www.openoffice.org/
- JodConverter 下载地址：http://www.artofsolving.com/opensource/jodconverter

OpenOffice的安装十分简单，一路Next即可。OpenOffice除了作为类Office的文档处理工具外，本身还可以作为服务启动，它自身的Writer软件（类似微软的Word、苹果的Pages）又提供了导出pdf的功能，因此也就可以利用OpenOffice的服务来实现将文档转换为pdf格式。

JodConverter是个Java库，提供了文档转换的功能。它实际上就是调用OpenOffice的服务来实现的。这里只是用到了它提供的API接口。

### 启动OpenOffice服务 ###

在CMD中使用如下命令将OpenOffice作为服务启动

```
soffice.exe -headless -accept="socket,host=127.0.0.1,port=8100;urp;
```

soffice.exe位于OpenOffice的安装目录下的program目录下。启动后，OpenOffice将监听8100端口。

这种启动服务的方式有个缺点，就是启动后的进程一直存在，并且占用一定内存。如果不想这样，实际上可以通过Java代码在需要转换的时候再启动OpenOffice服务。后面的代码中可以看到。

### 示例代码 ###

```
public static int word2PDF(String sourceFile, String destFile) {
    try {
        File inputFile = new File(sourceFile);
        if (!inputFile.exists()) {
            return -1;// 找不到源文件, 则返回-1
        }

        // 如果目标路径不存在, 则新建该路径
        File outputFile = new File(destFile);
        if (!outputFile.getParentFile().exists()) {
            outputFile.getParentFile().mkdirs();
        }

        String OpenOffice_HOME = "C:\\Program Files (x86)\\OpenOffice.org 3";//这里是OpenOffice的安装目录
        // 如果从文件中读取的URL地址最后一个字符不是 '\'，则添加'\'
        if (OpenOffice_HOME.charAt(OpenOffice_HOME.length() - 1) != '\\') {
            OpenOffice_HOME += "\\";
        }
        // 启动OpenOffice的服务
        String command = OpenOffice_HOME
                + "program\\soffice.exe -headless -accept=\"socket,host=127.0.0.1,port=8100;urp;\"";
        Process pro = Runtime.getRuntime().exec(command);
        
        // connect to an OpenOffice.org instance running on port 8100
        OpenOfficeConnection connection = new SocketOpenOfficeConnection(
                "127.0.0.1", 8100);
        connection.connect();

        // convert
        DocumentConverter converter = new OpenOfficeDocumentConverter(connection);
        converter.convert(inputFile, outputFile);

        // close the connection
        connection.disconnect();

        // 关闭OpenOffice服务的进程
        pro.destroy();

        return 0;
    } catch (FileNotFoundException e) {
        e.printStackTrace();
        return -1;
    } catch (ConnectException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }

    return 1;
}
```

以上就是使用OpenOffice完成pdf转换的过程。这种方法的缺点是速度比较慢。。如果将OpenOffice部署到服务器上，远程调用该服务的话，算上网络传输的时间大致在7秒左右。这还仅仅是pdf转换的时间，如果是查询数据->渲染模板->生成word->转换pdf这个过程的话。。。你懂的。。。

这里还有另一个大坑，稍不注意就死的很惨。。。

如果你是首先利用模板生成的word文档的话，在你将document.xml从docx中拿出来并且想要填写模板语言时，你肯定会那文本编辑器打开这个document.xml文件。乖乖，一坨XML代码就分了两行，第一行是XML的标准头，剩下所有的内容都在第二行。。虽然对机器来说这一点问题没有，但可惜你不是机器啊，你肯定会利用IDE格式化一下代码，待会儿往里写模板语言时候方便点。OK，你格式化了代码，然后写上模板语言的占位符、控制结构这些东西，然后用程序向这个模板打入动态的数据，再将这个document.xml放入docx中，压缩后成功的得到了含有动态数据的word文档。你用Office word打开这个docx文档，所有的样式、内容都十分完美，完全符合你的想法。但先别高兴的太早，这个时候如果你用OpenOffice的Writer打开这个docx，你肯定会骂娘，这格式全乱了，文字间的间距大的不行，一个表格的单元格的高度恨不得占满一页。。。

想想为什么会发生这种情况？与原始的docx相比，你只做了两件事儿，一是格式化了document.xml；二是用模板打入了动态数据。用模板实际上只是改变了某些展现的数据，与样式的关系应该不大。那只能是格式化造成的了。格式化的时候为了缩进，IDE会插入大量空白符，这是不是造成格式乱掉的原因呢？

如果你做个试验的话，将一个docx文档用winrar打开，将document.xml拿出来，用IDE打开，格式化，然后保存，再把格式化后的document.xml放回docx文档中。此时拿OpenOffice打开，果然乱掉了。。。

造成这样的原因不详，我猜测是Microsoft Office和OpenOffice对XML解析的方式不一样造成的。

怎么解决这个问题呢？方法也很简单，只需要再documentxml写完模板语言后，再还原成原来的两行XML的样子就行了。。。