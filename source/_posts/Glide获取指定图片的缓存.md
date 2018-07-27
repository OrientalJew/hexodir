---
title: 获取Glide缓存图片
date: 2018-07-27 
tags: 
- Glide
categories:
- Android
---

<!-- toc -->

用Glide这么久了，我一直有个疑问，Glide该如何获取到指定的缓存图片？

原生Glide是没有提供任何Api用来获取缓存图片的，至少我是没找到。

翻看Glide源码（3.7），发现其中一个叫：EngineKey的类，Glide通过该类来查找对应的缓存文件。
该类构造方法参数多达10个，并且不是开放出来的，也就是说，通过自己构造EngineKey这条路是走不通的。
<!--more--> 
难道就没有办法了吗？仔细翻看EngineKey这个类，其中有一个叫getOriginalKey的方法：
    
    public Key getOriginalKey() {
        if (originalKey == null) {
            originalKey = new OriginalKey(id, signature);
        }
        return originalKey;
    }

对应OriginalKey只需要两个参数，并且类的内容也很简单：
    
    class OriginalKey implements Key {

        private final String id;
        private final Key signature;
    
        public OriginalKey(String id, Key signature) {
            this.id = id;
            this.signature = signature;
        }
    
        @Override
        public boolean equals(Object o) {
            if (this == o) {
                return true;
            }
            if (o == null || getClass() != o.getClass()) {
                return false;
            }
    
            OriginalKey that = (OriginalKey) o;
    
            if (!id.equals(that.id)) {
                return false;
            }
            if (!signature.equals(that.signature)) {
                return false;
            }
    
            return true;
        }
    
        @Override
        public int hashCode() {
            int result = id.hashCode();
            result = 31 * result + signature.hashCode();
            return result;
        }
    
        @Override
        public void updateDiskCacheKey(MessageDigest messageDigest) throws UnsupportedEncodingException {
            messageDigest.update(id.getBytes(STRING_CHARSET_NAME));
            signature.updateDiskCacheKey(messageDigest);
        }
    }

再查看getOriginalKey()方法的调用者，发现被一个DecodeJob的两个方法调用到了：

    
    private Resource<T> cacheAndDecodeSourceData(A data) throws IOException {
        long startTime = LogTime.getLogTime();
        SourceWriter<A> writer = new SourceWriter<A>(loadProvider.getSourceEncoder(), data);
        diskCacheProvider.getDiskCache().put(resultKey.getOriginalKey(), writer);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Wrote source to cache", startTime);
        }

        startTime = LogTime.getLogTime();
        Resource<T> result = loadFromCache(resultKey.getOriginalKey());
        if (Log.isLoggable(TAG, Log.VERBOSE) && result != null) {
            logWithTimeAndKey("Decoded source from cache", startTime);
        }
        return result;
    }
    
    public Resource<Z> decodeSourceFromCache() throws Exception {
        if (!diskCacheStrategy.cacheSource()) {
            return null;
        }

        long startTime = LogTime.getLogTime();
        Resource<T> decoded = loadFromCache(resultKey.getOriginalKey());
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Decoded source from cache", startTime);
        }
        return transformEncodeAndTranscode(decoded);
    }

    

了解过Glide源码的人应该知道，这两个方法是在diskCacheStrategy设置为Source时，Glide缓存到本地和从本地取缓存的方法。

查看loadFromCache方法：

    private Resource<T> loadFromCache(Key key) throws IOException {
        File cacheFile = diskCacheProvider.getDiskCache().get(key);
        if (cacheFile == null) {
            return null;
        }

        Resource<T> result = null;
        try {
            result = loadProvider.getCacheDecoder().decode(cacheFile, width, height);
        } finally {
            if (result == null) {
                diskCacheProvider.getDiskCache().delete(key);
            }
        }
        return result;
    }

get方法点进去，发现有两个实现：DiskCacheAdapter和DiskLruCacheWrapper，DiskCacheAdapter只是个空壳，那么只有DiskLruCacheWrapper了。

其中获取缓存文件的方法：
    
    @Override
    public File get(Key key) {
        String safeKey = safeKeyGenerator.getSafeKey(key);
        File result = null;
        try {
            //It is possible that the there will be a put in between these two gets. If so that shouldn't be a problem
            //because we will always put the same value at the same key so our input streams will still represent
            //the same data
            final DiskLruCache.Value value = getDiskCache().get(safeKey);
            if (value != null) {
                result = value.getFile(0);
            }
        } catch (IOException e) {
            if (Log.isLoggable(TAG, Log.WARN)) {
                Log.w(TAG, "Unable to get from disk cache", e);
            }
        }
        return result;
    }
    
    private synchronized DiskLruCache getDiskCache() throws IOException {
        if (diskLruCache == null) {
            diskLruCache = DiskLruCache.open(directory, APP_VERSION, VALUE_COUNT, maxSize);
        }
        return diskLruCache;
    }


那就好办了！我们只需将diskCacheStrategy设置为Source，然后自己构造OriginalKey和DiskLruCache即可。

说干就干！

将Glide下的SafeKeyGenerator和OriginalKey类拷到自己的包下，这两个类也很简单：

    /**
     * A class that generates and caches safe and unique string file names from {@link com.bumptech.glide.load.Key}s.
     */
    class SafeKeyGenerator {
        private final LruCache<Key, String> loadIdToSafeHash = new LruCache<Key, String>(1000);
    
        public String getSafeKey(Key key) {
            String safeKey;
            synchronized (loadIdToSafeHash) {
                safeKey = loadIdToSafeHash.get(key);
            }
            if (safeKey == null) {
                try {
                    MessageDigest messageDigest = MessageDigest.getInstance("SHA-256");
                    key.updateDiskCacheKey(messageDigest);
                    safeKey = Util.sha256BytesToHex(messageDigest.digest());
                } catch (NoSuchAlgorithmException e) {
                    e.printStackTrace();
                } catch (UnsupportedEncodingException e) {
                    e.printStackTrace();
                }
                synchronized (loadIdToSafeHash) {
                    loadIdToSafeHash.put(key, safeKey);
                }
            }
            return safeKey;
        }
    }
    
    //OriginalKey 在上面！
    
然后就可以缓存文件获取了，如果你没有自己构造Signature和修改磁盘缓存位置，那用下面这个方法即可：

    //3.7
    public File getCacheFile(String url) {
        OriginalKey originalKey = new OriginalKey(url, EmptySignature.obtain());
        SafeKeyGenerator safeKeyGenerator = new SafeKeyGenerator();
        String safeKey = safeKeyGenerator.getSafeKey(originalKey);
        try {
            DiskLruCache diskLruCache = DiskLruCache.open(new File(getCacheDir(), DiskCache.Factory.DEFAULT_DISK_CACHE_DIR), 1, 1, DiskCache.Factory.DEFAULT_DISK_CACHE_SIZE);
            DiskLruCache.Value value = diskLruCache.get(safeKey);
            if (value != null) {
                return value.getFile(0);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
    

效果：


当然，如果不是Glide3.7，而是4.x呢？额，亲测在4.0上只需要改一个类即可，将OriginalKey换为DataCacheKey即可：

    /**
     * A cache key for original source data + any requested signature.
     */
    public class DataCacheKey implements Key  {
    
        private final Key sourceKey;
        private final Key signature;
    
        public DataCacheKey(Key sourceKey, Key signature) {
            this.sourceKey = sourceKey;
            this.signature = signature;
        }
    
        public Key getSourceKey() {
            return sourceKey;
        }
    
        @Override
        public boolean equals(Object o) {
            if (o instanceof DataCacheKey) {
                DataCacheKey other = (DataCacheKey) o;
                return sourceKey.equals(other.sourceKey) && signature.equals(other.signature);
            }
            return false;
        }
    
        @Override
        public int hashCode() {
            int result = sourceKey.hashCode();
            result = 31 * result + signature.hashCode();
            return result;
        }
    
        @Override
        public String toString() {
            return "DataCacheKey{"
                    + "sourceKey=" + sourceKey
                    + ", signature=" + signature
                    + '}';
        }
    
        @Override
        public void updateDiskCacheKey(MessageDigest messageDigest) {
            sourceKey.updateDiskCacheKey(messageDigest);
            signature.updateDiskCacheKey(messageDigest);
        }
    }


对应的方法：

    //4.0
    public File getCacheFile2(String url) {
        DataCacheKey dataCacheKey = new DataCacheKey(new GlideUrl(url), EmptySignature.obtain());
        SafeKeyGenerator safeKeyGenerator = new SafeKeyGenerator();
        String safeKey = safeKeyGenerator.getSafeKey(dataCacheKey);
        try {
            int cacheSize = 100 * 1000 * 1000;
            DiskLruCache diskLruCache = DiskLruCache.open(new File(getCacheDir(), DiskCache.Factory.DEFAULT_DISK_CACHE_DIR), 1, 1, cacheSize);
            DiskLruCache.Value value = diskLruCache.get(safeKey);
            if (value != null) {
                return value.getFile(0);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
    
    
写了一个非常简单的Demo ：  [https://github.com/HuangShengHuan/GlideCache](https://github.com/HuangShengHuan/GlideCache)  
