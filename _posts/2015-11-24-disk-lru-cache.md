---
layout: post
title: Android磁盘缓存
tags: cache
categories: Android
date: 2015-11-24
---

### 1.概述
在上一篇文章中简单介绍了[内存缓存](http://www.yuxingxin.com/2015/11/23/LruCache/)，其核心就是LruCache这个类，我们知道它的优点就是直接可以读取内存，当然速度就会很快，但是它同时也有下面不足的地方：
1. 手机内存空间十分有限，所以我们不能随意的设置内存缓存大小。
2. 内存紧张时可能会优先被GC回收掉。
3. 退出应用时就回收掉，不能离线存储数据

基于以上原因，于是就有了磁盘缓存，Android开源届Jake大神为我们提供了一种解决方案:[DiskLruCache](https://github.com/JakeWharton/DiskLruCache),也是现在应用的最广泛的一种，并且已经获得谷歌官方认可。

### 2.使用
#### 打开缓存
通过源码我们可以看到DiskLruCache的构造方法时私有的，所以不能通过new的方法来获取到它的实例，但是作者给我们提供了一种方式来创建实例，通过调用open方法，其中的四个参数刚好与构造函数中的四个参数相吻合:
1. directory指定缓存的目录
2. appVersion指定应用程序的版本号
3. valueCount指定一个key可以对应多少个缓存文件，一般情况下我们都指定这个参数为1
4. maxSize指定我们最多可以缓存多大的字节数
```
  private DiskLruCache(File directory, int appVersion, int valueCount, long maxSize) {
    this.directory = directory;
    this.appVersion = appVersion;
    this.journalFile = new File(directory, JOURNAL_FILE);
    this.journalFileTmp = new File(directory, JOURNAL_FILE_TEMP);
    this.journalFileBackup = new File(directory, JOURNAL_FILE_BACKUP);
    this.valueCount = valueCount;
    this.maxSize = maxSize;
  }

    /**
   * Opens the cache in {@code directory}, creating a cache if none exists
   * there.
   *
   * @param directory a writable directory
   * @param valueCount the number of values per cache entry. Must be positive.
   * @param maxSize the maximum number of bytes this cache should use to store
   * @throws IOException if reading or writing the cache directory fails
   */
  public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
      throws IOException {
    if (maxSize <= 0) {
      throw new IllegalArgumentException("maxSize <= 0");
    }
    if (valueCount <= 0) {
      throw new IllegalArgumentException("valueCount <= 0");
    }

    // If a bkp file exists, use it instead.
    File backupFile = new File(directory, JOURNAL_FILE_BACKUP);
    if (backupFile.exists()) {
      File journalFile = new File(directory, JOURNAL_FILE);
      // If journal file also exists just delete backup file.
      if (journalFile.exists()) {
        backupFile.delete();
      } else {
        renameTo(backupFile, journalFile, false);
      }
    }

    // Prefer to pick up where we left off.
    DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    if (cache.journalFile.exists()) {
      try {
        cache.readJournal();
        cache.processJournal();
        cache.journalWriter = new BufferedWriter(
            new OutputStreamWriter(new FileOutputStream(cache.journalFile, true), Util.US_ASCII));
        return cache;
      } catch (IOException journalIsCorrupt) {
        System.out
            .println("DiskLruCache "
                + directory
                + " is corrupt: "
                + journalIsCorrupt.getMessage()
                + ", removing");
        cache.delete();
      }
    }

    // Create a new empty cache.
    directory.mkdirs();
    cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    cache.rebuildJournal();
    return cache;
  }
```
然后缓存路径为：/sdcard/Android/data/<application package>/cache 这个路径下面，但这里我们通常的做法就是判断是否有SD卡，没有的话就放在内部存储里面，缓存路径为：/data/data/<application package>/cache 
```
public File getDiskCacheDir(Context context, String uniqueName) {  
    String cachePath;  
    if (Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())  
            || !Environment.isExternalStorageRemovable()) {  
        cachePath = context.getExternalCacheDir().getPath();  
    } else {  
        cachePath = context.getCacheDir().getPath();  
    }  
    return new File(cachePath + File.separator + uniqueName);  
}  
```
这里的uniqueName值作为我们区分不同的数据的一个唯一值，如存放图片的缓存文件夹（images），存放视频的缓存文件夹(videos)，存放文本文件的缓存文件夹（txt）。
第二个参数获取版本号就很简单了
```
public int getAppVersion(Context context) {  
    try {  
        PackageInfo info = context.getPackageManager().getPackageInfo(context.getPackageName(), 0);  
        return info.versionCode;  
    } catch (NameNotFoundException e) {  
        e.printStackTrace();  
    }  
    return 1;  
}  
```
第三个参数上面提到了为1，第四个参数假定我们设为10M，那么就有了：
```
DiskLruCache mDiskLruCache = null;  
try {  
    File cacheDir = getDiskCacheDir(context, "images");  
    if (!cacheDir.exists()) {  
        cacheDir.mkdirs();  
    }  
    mDiskLruCache = DiskLruCache.open(cacheDir, getAppVersion(context), 1, 10 * 1024 * 1024);  
} catch (IOException e) {  
    e.printStackTrace();  
}  
```

#### 存缓存到磁盘
这点类似SharePreferences,DiskLruCache给我们提供了一个内部类Editor，它用来完成针对缓存文件的读取、储存、删除、修改等等操作：通过调用实例的edit方法传入key值来把文件缓存到磁盘，然后针对函数返回的Editor实例调用commit方法完成最后的提交。
```
mDiskLruCache.edit(key).commit();  
mDiskLruCache.flush();  
```
这里的key值我们一般使用MD5加密后的字符串来做唯一处理，当然也可以调用abort来放弃提交。

#### 读取缓存文件
读取缓存文件通过get(key)方法来进行：
```
InputStream is = mDiskLruCache.get(key).getInputStream(0);
//TODO 然后再对流做进一步的处理
```

#### 移除缓存
移除缓存调用remove(key)方法：
```
mDiskLruCache.remove(key);
```

#### 获取缓存大小
通过size()方法返回当前缓存路径下面所有缓存数据的字节数，以byte为单位。

#### 关闭缓存
调用close()方法，通常在界面销毁的时候调用

#### 删除缓存
调用delete()方法,即可清除缓存



