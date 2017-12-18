---
title: Picasso+Okhttp3缓存优化
date: 2017-12-18 22:06:03
tags:
---


Picasso本身并没有去“实现”本地缓存功能，而是让网络请求层去缓存http响应，其网络请求逻辑在picasso中对应的是DownLoader接口的实现。

<!--more-->

 - 配置gradle

```Java
    compile 'com.squareup.okhttp3:okhttp:3.9.0'
    compile 'com.squareup.okhttp3:logging-interceptor:3.9.0'
    compile 'com.squareup.picasso:picasso:2.5.2'
```
 - 定义OKhttp3.0+ 版本的DownLoader
```Java
import android.net.Uri;

import com.squareup.picasso.Downloader;
import com.squareup.picasso.NetworkPolicy;

import java.io.IOException;

import okhttp3.Cache;
import okhttp3.CacheControl;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.ResponseBody;

public class ImageDownLoader implements Downloader
{
    OkHttpClient client = null;

    public ImageDownLoader(OkHttpClient client)
    {
        this.client = client;
    }

    @Override
    public Response load(Uri uri, int networkPolicy) throws IOException
    {

        CacheControl cacheControl = null;
        if (networkPolicy != 0)
        {
            if (NetworkPolicy.isOfflineOnly(networkPolicy))
            {
                cacheControl = CacheControl.FORCE_CACHE;
            }
            else
            {
                CacheControl.Builder builder = new CacheControl.Builder();
                if (!NetworkPolicy.shouldReadFromDiskCache(networkPolicy))
                {
                    builder.noCache();
                }
                if (!NetworkPolicy.shouldWriteToDiskCache(networkPolicy))
                {
                    builder.noStore();
                }
                cacheControl = builder.build();
            }
        }

        Request.Builder builder = new Request.Builder().url(uri.toString());
        if (cacheControl != null)
        {
            builder.cacheControl(cacheControl);
        }

        okhttp3.Response response = client.newCall(builder.build()).execute();
        int responseCode = response.code();
        if (responseCode >= 300)
        {
            response.body().close();
            throw new ResponseException(responseCode + " " + response.message(), networkPolicy,
                    responseCode);
        }

        boolean fromCache = response.cacheResponse() != null;

        ResponseBody responseBody = response.body();
        return new Response(responseBody.byteStream(), fromCache, responseBody.contentLength());

    }

    @Override
    public void shutdown()
    {

        Cache cache = client.cache();
        if (cache != null)
        {
            try
            {
                cache.close();
            }
            catch (IOException ignored)
            {
            }
        }
    }
}
```
 - 设置拦截器

> 在服务端的响应头里加上Cache-Control 如Cache-Control:
> max-age=3600，3600表示缓存有效时间为1个小时，即在1个小时之内再次请求同一个url都不会访问网络，在有无网络的情况下都读取缓存，此时，如果想请求最新的内容，应该在构造request的时候使用CacheControl的noCache来实现
> 
> 我们可以用okhttp的拦截器Interceptor来实现在本地修改响应头的内容,甚至可以在里面判断wifi,4g,3g来实现不同的缓存策略。


```Java

import android.content.Context;
import android.util.Log;

import com.gaoyy.mishop.util.NetworkUtils;

import java.io.IOException;

import okhttp3.CacheControl;
import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;

/**
 * Created by gaoyy on 2017/12/18 0018.
 */

public class CacheInterceptor implements Interceptor
{

    private Context context;

    public CacheInterceptor(Context context)
    {
        this.context = context;
    }

    @Override
    public Response intercept(Chain chain) throws IOException
    {
        Log.d("tag", "Intercept respone");

        Request request = chain.request();
        //如果没有网络，则启用 FORCE_CACHE
        if (!NetworkUtils.isNetworkConnected(context))
        {
            request = request.newBuilder()
                    .cacheControl(CacheControl.FORCE_CACHE)
                    .build();
        }

        Response originalResponse = chain.proceed(request);
        if (NetworkUtils.isNetworkConnected(context))
        {
            //有网的时候读接口上的@Headers里的配置
            Log.d("tag", "code " + originalResponse.code());
            String cacheControl = request.cacheControl().toString();
            return originalResponse.newBuilder()
                    .header("Cache-Control", cacheControl)
                    .removeHeader("Pragma")
                    .build();
        }
        else
        {
            return originalResponse.newBuilder()
                    .header("Cache-Control", "public, only-if-cached, max-stale=3600")
                    .removeHeader("Pragma")
                    .build();
        }
    }
}

```
 - 在Application中进行配置
 ```Java
    private void initPicasso()
    {
        File file = new File(this.getCacheDir(),"Picasso");
        OkHttpClient client =new OkHttpClient
                .Builder()
                .addInterceptor(new CacheInterceptor(this))
                .cache(new Cache(file, 1024 * 1024 * 100))
                .build();

        Picasso picasso = new Picasso.Builder(this)
                .defaultBitmapConfig(Bitmap.Config.RGB_565)
                .downloader(new ImageDownLoader(client))
                .indicatorsEnabled(true)
                .build();
        Picasso.setSingletonInstance(picasso);
    }
 ```
 
 同样也可以对Retrofit进行使用
 ```Java
 public static void init(Context context)
    {
        // 指定缓存路径,缓存大小100Mb
        Cache cache = new Cache(new File(context.getCacheDir(), "HttpCache"),
                1024 * 1024 * 100);
        OkHttpClient okHttpClient = new OkHttpClient.Builder()
                .cache(cache)
                .retryOnConnectionFailure(true)
                .connectTimeout(1500000, TimeUnit.MILLISECONDS)
                .writeTimeout(2000000, TimeUnit.MILLISECONDS)
                .readTimeout(2000000, TimeUnit.MILLISECONDS)
                .addInterceptor(new CacheInterceptor(context))
                .addNetworkInterceptor(new CacheInterceptor(context))
                .build();


        Retrofit apiRetrofit = new Retrofit.Builder()
                .baseUrl(Constant.API_BASE)
                .client(okHttpClient)
                .addConverterFactory(GsonConverterFactory.create())
                .build();

        sApiService = apiRetrofit.create(Api.class);

    }
 ```

参考：
http://www.jianshu.com/p/6241950f9daf
http://www.jianshu.com/p/093ca3c1447d
