---
layout: post
title: "将InputStream解压到文件夹"
mathjax: true
tags: kotlin, apache commons
---

老大把基础库封了jar包给我用，里面是有一个类似的函数的。但是我用起来发现它有bug，在解压有好几层目录的zip文件时会报一个找不到文件夹的错误。

反编译了jar包，想找找bug，但是他这个库是kotlin写的，目前没找到将`.class`文件直接反编译到kotlin的工具，只有反编译到java的，但是反编译出来的东西看起来实在费劲：

```java
   public final void unzipStreamToDir(@NotNull InputStream inputStream, @NotNull String destDir) {
      Intrinsics.checkNotNullParameter(inputStream, "inputStream");
      Intrinsics.checkNotNullParameter(destDir, "destDir");
      File dir = new File(destDir);
      if (!dir.exists()) {
         dir.mkdirs();
      }

      byte[] buffer = new byte[16384];
      Closeable var5 = (Closeable)(new ZipInputStream(inputStream));
      boolean var6 = false;
      boolean var7 = false;
      Throwable var31 = (Throwable)null;

      try {
         ZipInputStream zis = (ZipInputStream)var5;
         int var9 = false;

         for(ZipEntry ze = zis.getNextEntry(); ze != null; ze = zis.getNextEntry()) {
            String fileName = ze.getName();
            File newFile = new File(destDir, fileName);
            (new File(newFile.getParent())).mkdirs();
            Closeable var13 = (Closeable)(new FileOutputStream(newFile));
            boolean var14 = false;
            boolean var15 = false;
            Throwable var33 = (Throwable)null;

            try {
               FileOutputStream fos = (FileOutputStream)var13;
               boolean var17 = false;

               while(true) {
                  int len = zis.read(buffer);
                  if (len <= 0) {
                     Unit var34 = Unit.INSTANCE;
                     break;
                  }

                  fos.write(buffer, 0, len);
               }
            } catch (Throwable var27) {
               var33 = var27;
               throw var27;
            } finally {
               CloseableKt.closeFinally(var13, var33);
            }

            zis.closeEntry();
         }

         Unit var32 = Unit.INSTANCE;
      } catch (Throwable var29) {
         var31 = var29;
         throw var29;
      } finally {
         CloseableKt.closeFinally(var5, var31);
      }

   }
```

而且老大已经把库封成了jar包，我找到bug也得他来改，他还不一定有时间。求人不如求己，我就自己写了一个。

## apache commons, YYDS！
apache commons compress 模块提供了操作压缩文件的api，还提供了使用样例：[样例代码](https://commons.apache.org/proper/commons-compress/examples.html)，
对于zip文件还有一些扩展的支持： [zip page](https://commons.apache.org/proper/commons-compress/zip.html)

照着文档的例子，翻译成了kotlin，:

```kotlin
// 参考: https://commons.apache.org/proper/commons-compress/examples.html
// 为什么用 ZipFile: https://commons.apache.org/proper/commons-compress/zip.html#ZipArchiveInputStream_vs_ZipFile
fun unzipInputStream(inputStream: InputStream, targetPath: String) {
    println("Started unzipping.")
    val targetDir = File(targetPath)
    if (!targetDir.isDirectory && !targetDir.mkdirs()) {
        throw IOException("Failed to create directory $targetDir")
    }
    // in memory archive with org.apache.commons.compress.utils.SeekableInMemoryByteChannel
    val ch = SeekableInMemoryByteChannel(IOUtils.toByteArray(inputStream))
    
    ZipFile(ch).use { zipFile ->
        for (entry in zipFile.entries) {
            println("entry: $entry")
            val name = FilenameUtils.concat(targetPath, entry.name)
            val f = File(name)
            
            if (entry.isDirectory) {
                if (!f.isDirectory && !f.mkdirs()) {
                    throw IOException("Failed to create directory $f")
                }
            } else {
                val parent = f.parentFile
                if (!parent.isDirectory && !parent.mkdirs()) {
                    throw IOException("Failed to create directory $parent")
                }
                FileUtils.copyInputStreamToFile(zipFile.getInputStream(entry), f)
            }
        }
    }
    println("Finished unzipping")
}
```
1. 用kotlin 的 `use` 代替了样例java代码里的 `try-with-resources`,
2. 按照zip文件的文档里说的，用`ZipFile`代替了`ZipArchiveInputStream`,
3. 将输入的流保存到了`ByteArray`中，新建一个指向这个字节数组的`SeekableInMemoryByteChannel`，再通过这个channel新建一个`ZipFile`对象，这样就实现了内存中“虚拟文件”的效果。
也可以先保存到硬盘，但是此处没有这个需求，就直接在内存中处理了,
4. `ZipFile.entries`方法返回从zip文件所在目录开始的相对路径列表，如：
```
Started unzipping.
entry: a.txt
entry: folder/
entry: folder/b.png
entry: folder/another-folder/
entry: folder/another-folder/c.json
Finished unzipping
```
   会先返回文件夹entry，然后返回文件夹中的文件。因此，在遍历时，对于文件夹类型的entry，需要保证文件夹存在，对文件类型的entry，需要保存文件内容。

## 结论
kotlin + apache commons = 我等 调包boy(crud boy, api boy) 的 天堂