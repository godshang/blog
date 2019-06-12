---
title: 'Java生成Word文件'
layout: post
categories: 技术
tags:
    - Java
---

## 方案比较 ##

1、POI：Apache POI包括一系列的API，它可以操作Office的各种格式文件，通过这些API在Java中读写Excel、Word等文件。POI对excel的处理很强大，但对于word的支持不是很好，目前只能实现一些简单文件的操作。 

2、使用XML做模板：word支持XML格式描述，因此利用Velicity等模板引擎，将动态数据渲染到XML模板中，从而生成word文档。这种方法比较简单，而且处理速度很快，样式和内容的控制也非常好，打印的话也不走样。可以说是生成Word文档的最佳方案。

使用XML进行Word生成，又有两种不同的方法：

在word中编辑好样式，另存为XML格式。用文本编辑器打开这个XML文档，会发现是使用标准的XML语言描述的文档内容和样式。之后将需要动态渲染数据的位置，使用模板语言的占位符描述；同时还可以使用模板语言中的一些循环、条件等控制结构，比如迭代生成表格的行。经过这样处理后的XML文档，就可以作为模板使用了。同使用其他模板一样，将动态数据打入模板后，所得到的文本流写入到另一个文件中，并另存为doc格式，这样就得到了生成后的word文档。

这种方法有一个问题，就是最后得到的文档，虽然被另存为doc格式，也能够被office word正常的打开和编辑，但如果用文本编辑器打开，会发现它依然是个XML文档，底层的数据存储方式并未发生变化。这对于只生成、编辑和打印word来说并不是个问题，这种方法完全满足这些需求。但如果进一步，想使用OpenOffice将这个word转为PDF的话，就会出现问题了。你会发现，转换后的PDF，页面全是XML标签。。

因此，如果想得到“真真正正”的word文档来说，就得使用第二种方法了。第二种方法完全解决了这个问题，但同时又有另一个局限，它只能针对word2007格式，即docx这种格式。不过目前来说，office 2003虽然没完全淘汰，但2007、2010的使用率也是绝大多数。这个问题就当它不是个问题好了。。

介绍第二种方法前，首先来了解word2007文档的格式。Word2007文档本身是以XML文件格式存储的，然后将相关的XML文件和资源文件压缩后得到。你如果将一个已经含有内容的word 2007的文档的扩展名由docx改为zip，然后你会发现这个压缩包下的目录结构类似这样：

```
| [Content_Types].xml
|-_rely
|   .rels
|-customXml
|   |-_rels
|        item1.xml.rels
|       item1.xml
|       itemProps1.xml
|-docProps
|     app.xml
|     core.xml
|-word
|   |-rels
|   |media
|   |theme
|   document.xml
|   endnotes.xml
|   fonTable.xml
|   footer1.xml
|   footnotes.xml
|   header1.xml
|   numbering.xml
|   settings.xml
|   styles.xml
|   webSettings.xml
```

目录结构比较复杂，但从名字也能够大概猜出来一些，document.xml中存储的是页面中的内容，header1.xml存储的是页眉的内容。。。用文本编辑器打开这几个文件，也验证了猜想。

在了解了word2007文档的格式后，也就有了解决的方案。构建word2007文档的问题也就转化成了修改一个压缩文件中的xml问题。同样可以使用模板技术来向XML文档中动态的打入数据，然后替换压缩文件中的相应XML文档，就可以实现生成word 2007的需求。

## 技术实现 ##

以下用一个简单的实例介绍具体的实现步骤。

### 第一步：创建模板 ###

首先，新建一个word2007文档,并完成内容和样式的编辑。然后将这个docx文档的扩展名改为zip，使用压缩工具打开，将其中的document.xml解压出来，然后从压缩文件中删除document.xml。然后将压缩文件名修改为template.docx。

### 第二步：编写代码 ###

大致流程为

1. 将document.xml中使用模板语言进行预处理。
2. 使用模板引擎将数据加载到document.xml文件中。
2. 使用Java中的ZipInputStream/ZipOutputStream操纵zip文档，将上一步生成的document.xml压缩到docx中。
3. 最后输出为一个新的docx文件。

关键代码片段为

```
private String generateWord(HttpServletRequest req, HttpServletResponse resp) {

    String documentFilePath = ...; // 使用模板语言处理后的document.xml文件位置
    String templatePath = ...; // template.docx的文件位置

    String tempDocumentFilePath = ...; // 填充数据后的document.xml文件位置
    String tempWordFilePath = ...; // 最后生成的word 2007文件位置
    
    File tempWordFile = new File(tempWordFilePath);
    File document = new File(tempDocumentFilePath);
            
    try {
        generateDocument(documentFilePath, tempDocumentFilePath);
        
        createWordFile(tempWordFilePath, templatePath);

        addDocumentToTemplate(tempWordFile, document, header);
    } catch (Exception e) {
        throw new RuntimeException("Error when generating word file.", e);
    } finally {
        if(document != null) {
            document.delete();
        }
        if(header != null) {
            header.delete();
        }
    }
    return tempWordFilePath;
}

//利用模板引擎填充document.xml
private void generateDocument(String documentFilePath, String tempDocumentFilePath) throws Exception {
    PerformOrderData data = DataSource.getDataList();
    Map<String, Object> dataMap = new HashMap<String, Object>();
    dataMap.put("performOrderName", data.getPerformOrderName());
    dataMap.put("agentName", data.getAgentName());
    dataMap.put("version", data.getVersion());
    dataMap.put("custName", data.getCustName());
    dataMap.put("url", data.getUrl());
    dataMap.put("directSaler", data.getDirectSaler());
    dataMap.put("channelSaler", data.getChannelSaler());
    dataMap.put("products", data.getProducts());

    VelocityUtils.fill(documentFilePath, tempDocumentFilePath, dataMap);
}

//将document.xml加入到template.docx中
private void addDocumentToTemplate(File tempWordFile, File document, File header) throws Exception {
    // get a temp file
    File tempFile = File.createTempFile(tempWordFile.getName(), null);
    // delete it, otherwise you cannot rename your existing zip to it.
    tempFile.delete();

    boolean renameOk = tempWordFile.renameTo(tempFile);
    if (!renameOk) {
        throw new RuntimeException("could not rename the file "
                + tempWordFile.getAbsolutePath() + " to "
                + tempFile.getAbsolutePath());
    }

    ZipInputStream zin = null;
    ZipOutputStream zout = null;
    InputStream in = null;
    try {
        zin = new ZipInputStream(new FileInputStream(tempFile));;
        zout = new ZipOutputStream(new FileOutputStream(tempWordFile));;
        
        ZipEntry entry = zin.getNextEntry();
        while (entry != null) {
            String name = entry.getName();
            boolean notInFiles = true;
            if (document != null) {
                if (document.getName().equals(name) || header.getName().equals(name)) {
                    notInFiles = false;
                }
            }
            if (notInFiles) {
                // Add ZIP entry to output stream.
                zout.putNextEntry(new ZipEntry(name));
                // Transfer bytes from the ZIP file to the output file
                write(zin, zout);
            }
            entry = zin.getNextEntry();
        }
        // Compress the files
        if (document != null) {
            in = new FileInputStream(document);
            // Add ZIP entry to output stream.
            zout.putNextEntry(new ZipEntry("word/document.xml"));
            // Transfer bytes from the file to the ZIP file
            write(in, zout);
            // Complete the entry
            zout.closeEntry();
            in.close();
        }
        if (header != null) {
            in = new FileInputStream(header);
            zout.putNextEntry(new ZipEntry("word/header1.xml"));
            write(in, zout);
            zout.closeEntry();
            in.close();
        }
        tempFile.delete();
    } finally {
        if (zin != null) {
            zin.close();
        }
        if (zout != null) {
            zout.close();
        }
        if (in != null) {
            in.close();
        }
    }
}

private void write(InputStream in, OutputStream out) throws Exception {
    byte[] buf = new byte[1024];
    int len;
    while ((len = in.read(buf)) > 0) {
        out.write(buf, 0, len);
    }
    out.flush();
}

private void createWordFile(String targetWordPath, String templatePath) throws Exception {
    InputStream in = null;
    OutputStream out = null;
    try {
        in = this.getClass().getResourceAsStream(templatePath);
        out = new FileOutputStream(new File(targetWordPath));
        write(in, out);
    } finally {
        if(in != null) {
            in.close();
        }
        if(out != null) {
            out.close();    
        }
    }
}
```

这种方法的关键点就是这样。方法本身还是很好理解，麻烦的在于模板的处理上。这里还没反应出这种麻烦，因为只是生成了word文档而已，等到进行转换pdf的时候，还会有新的问题出现。。。