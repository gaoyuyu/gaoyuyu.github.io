---
title: Okhttp3-编写GET/POST下的Token自动检查并刷新的拦截器（同时适配Okgo库）
date: 2018-05-23 20:53:01
tags:
---

自动检查token的有效性并刷新无效token的需求在项目中很常见，下面是在项目中用到的Okttp3 Token拦截器，实现了

 - 在GET请求下，自动检查token的有效性，并对原有的token参数进行替换更新
 - 在POST请求下，自动检查token的有效性，取出FormBody中原有的请求参数，替换原有的token参数
<!--more-->

```Java
public class OkTokenInterceptor implements Interceptor
{
    private static final String LOG = OkTokenInterceptor.class.getSimpleName();
    private static final Charset UTF_8 = Charset.forName("UTF-8");

    @Override
    public Response intercept(Chain chain) throws IOException
    {
        Request originalRequest = chain.request();

        HttpUrl originalHttpUrl = originalRequest.url();
        String originalUrl = originalHttpUrl.url().toString();
        String method = originalRequest.method();

        Logger.i(LOG, "originalUrl-->" + originalUrl);
        Logger.i(LOG, "method-->" + originalRequest.method());

        final Response originalResponse = chain.proceed(originalRequest);
        // 获取返回的数据字符串
        ResponseBody responseBody = originalResponse.body();
        BufferedSource source = originalResponse.body().source();
        source.request(Integer.MAX_VALUE);
        Buffer buffer = source.buffer();
        Charset charset = UTF_8;
        MediaType contentType = responseBody.contentType();
        if (contentType != null)
        {
            charset = contentType.charset();
            charset = charset == null ? UTF_8 : charset;
        }
        String bodyString = buffer.clone().readString(charset);

        Logger.i(LOG, "拦截前：" + bodyString);

        final String token = DatabaseUtils.getInstance().getToken();
        Logger.i(LOG, "当前的token：" + token);

        //排除登录的API
        if (!originalUrl.equals(EcConstant.LOGIN_API))
        {
            if (!TextUtils.isEmpty(token))
            {
                Retrofit retrofit = new Retrofit.Builder()
                        .baseUrl(EcConstant.BASE_URL)
                        .addConverterFactory(ScalarsConverterFactory.create())
                        .client(new OkHttpClient())
                        .build();

                TokenApi api = retrofit.create(TokenApi.class);

                Call<String> checkTokencall = api.get(EcConstant.CHECK_TOKEN_API + "token/" + token + "/");

                String checkTokenResponse = checkTokencall.execute().body(); // 同步

                Logger.i(LOG, "checkTokencall-->" + checkTokenResponse);

                JSONObject checkTokenObject = JSON.parseObject(checkTokenResponse);
                int checkTokenCode = checkTokenObject.getIntValue("code");
                if (checkTokenCode == ResponseCode.SUCCESS)
                {
                    Logger.i(LOG, "token：" + token + " 有效 ");
                }
                else
                {
                    if (method.equalsIgnoreCase("GET"))
                    {
                        StringBuilder newUrl = null;

                        int index = originalUrl.indexOf("?");

                        newUrl = new StringBuilder(originalUrl.substring(0, index) + "?");

                        Set<String> queryParameterNames = originalHttpUrl.queryParameterNames();

                        for (String key : queryParameterNames)
                        {
                            //循环参数列表
                            if (key.equals("token"))
                            {
                                String tokenValue = originalHttpUrl.queryParameter(key);
                                Logger.i(LOG + "[GET]", "旧token参数：" + tokenValue);
                            }
                            else
                            {
                                newUrl.append(key).append("=").append(originalHttpUrl.queryParameter(key)).append("&");
                            }
                        }

                        //去除最后一个&符
                        newUrl.deleteCharAt(newUrl.length() - 1);

                        Logger.i(LOG + "[GET]", "token：" + token + " 无效 ");

                        Call<String> updateTokenCall = api.get(EcConstant.UPDATE_TOKEN_API + "token/" + token + "/");

                        String updateTokenResponse = updateTokenCall.execute().body();

                        Logger.i(LOG + "[GET]", "updateTokenCall-->" + updateTokenResponse);

                        JSONObject updateTokenObject = JSON.parseObject(updateTokenResponse);
                        int updateTokenCode = updateTokenObject.getIntValue("code");
                        if (updateTokenCode == ResponseCode.SUCCESS)
                        {
                            //获取更新后的token和有效期
                            JSONObject data = updateTokenObject.getJSONObject("data");
                            String newToken = data.getString("token");
                            int outTime = data.getIntValue("outtime");
                            //更新数据库
                            DatabaseUtils.getInstance().updateToken(newToken, outTime, EcUtils.getCurrentTimeMillis() + outTime);

                            newUrl.append("&token=").append(newToken);

                            Logger.i(LOG + "[GET]", "newUrl-->" + newUrl);

                            //重新构建Request
                            Request newRequest = chain.request().newBuilder()
                                    .url(newUrl.toString())
                                    .build();

                            //关闭原先的请求响应
                            originalResponse.body().close();

                            //打印body参数
                            bodyToString(newRequest);

                            //继续返回新的请求
                            return chain.proceed(newRequest);
                        }
                        else
                        {
                            EventBus.getDefault().post(new ReLoginEvent());
                        }
                    }
                    else if (method.equalsIgnoreCase("POST"))
                    {
                        //初始化新的FormBody
                        FormBody.Builder newFormBody = new FormBody.Builder();

                        FormBody oldFormBody = (FormBody) originalRequest.body();

                        for (int i = 0; i < oldFormBody.size(); i++)
                        {
                            //去除旧的参数中的token字段值
                            if (oldFormBody.encodedName(i).equals("token")) continue;
                            //在新的FormBody增加旧的FormBody的参数
                            newFormBody.addEncoded(oldFormBody.encodedName(i), oldFormBody.encodedValue(i));
                        }

                        Logger.i(LOG, "token：" + token + "无效");

                        Call<String> updateTokenCall = api.get(EcConstant.UPDATE_TOKEN_API + "token/" + token + "/");

                        String updateTokenResponse = updateTokenCall.execute().body();

                        Logger.i(LOG, "updateTokenCall-->" + updateTokenResponse);

                        JSONObject updateTokenObject = JSON.parseObject(updateTokenResponse);
                        int updateTokenCode = updateTokenObject.getIntValue("code");
                        if (updateTokenCode == ResponseCode.SUCCESS)
                        {
                            //获取更新后的token和有效期
                            JSONObject data = updateTokenObject.getJSONObject("data");
                            String newToken = data.getString("token");
                            int outTime = data.getIntValue("outtime");
                            //更新数据库
                            DatabaseUtils.getInstance().updateToken(newToken, outTime, EcUtils.getCurrentTimeMillis() + outTime);

                            //添加新的token到FormBody
                            newFormBody.add("token", newToken);

                            //重新生成URL
                            HttpUrl url = chain.request().url()
                                    .newBuilder()
                                    .build();

                            //重新构建Request
                            Request newRequest = chain.request().newBuilder()
                                    .post(newFormBody.build())
                                    .url(url)
                                    .build();

                            //关闭原先的请求响应
                            originalResponse.body().close();

                            //打印body参数
                            bodyToString(newRequest);

                            //继续返回新的请求
                            return chain.proceed(newRequest);
                        }
                        else
                        {
                            EventBus.getDefault().post(new ReLoginEvent());
                        }
                    }
                }
            }
        }
        return originalResponse;

    }

    private static Charset getCharset(MediaType contentType)
    {
        Charset charset = contentType != null ? contentType.charset(UTF_8) : UTF_8;
        if (charset == null) charset = UTF_8;
        return charset;
    }

    /**
     * 打印body参数
     *
     * @param request 请求
     */
    private void bodyToString(Request request)
    {
        try
        {
            Request copy = request.newBuilder().build();
            RequestBody body = copy.body();
            if (body == null) return;
            Buffer buffer = new Buffer();
            body.writeTo(buffer);
            Charset charset = getCharset(body.contentType());
            Logger.i(LOG, "拦截后的body参数：" + buffer.readString(charset));
        }
        catch (Exception e)
        {
            OkLogger.printStackTrace(e);
        }
    }
}
```


----------
由于在项目中也用到了[jeasonlzy/okhttp-OkGo](https://github.com/jeasonlzy/okhttp-OkGo)，读了其中的源码，发现该作者对RequestBody进行了二次封装，外面套了一层ProgressRequestBody，所以，上述的Token拦截器中的POST部分对于该库不在适用，但是GET部分依旧通用，不过要修改下原作者的ProgressRequestBody源码，将真实的RequestBody暴露出来以供使用。
![这里写图片描述](https://img-blog.csdn.net/20180523204917433?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0dhb195dXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```Java
ProgressRequestBody.java

public RequestBody getRequestBody()
{
    if(requestBody == null)
    {
        throw new NullPointerException("There is Null RequestBody in ProgressRequestBody<Okgo>");
    }
    return requestBody;
}

public void setRequestBody(RequestBody requestBody)
{
    this.requestBody = requestBody;
}
```

针对POST部分对拦截器进行修改
```Java
 else if (method.equalsIgnoreCase("POST"))
 {
     //注意：此处originalRequest.body()获取的是ProgressRequestBody
     ProgressRequestBody progressRequestBody = (ProgressRequestBody) originalRequest.body();

     //初始化新的FormBody
     FormBody.Builder newFormBody = new FormBody.Builder();

     //在Ok-go的ProgressRequestBody中增加了暴露RequestBody的方法
     FormBody oldFormBody = (FormBody) progressRequestBody.getRequestBody();

     for (int i = 0; i < oldFormBody.size(); i++)
     {
         //去除旧的参数中的token字段值
         if (oldFormBody.encodedName(i).equals("token")) continue;
         //在新的FormBody增加旧的FormBody的参数
         newFormBody.addEncoded(oldFormBody.encodedName(i), oldFormBody.encodedValue(i));
     }

     CaipuLogger.i(LOG + "[POST]", "token：" + token + "无效");

     Call<String> updateTokenCall = api.get(EcConstant.UPDATE_TOKEN_API + "token/" + token + "/");

     String updateTokenResponse = updateTokenCall.execute().body();

     CaipuLogger.i(LOG + "[POST]", "updateTokenCall-->" + updateTokenResponse);

     JSONObject updateTokenObject = JSON.parseObject(updateTokenResponse);
     int updateTokenCode = updateTokenObject.getIntValue("code");
     if (updateTokenCode == ResponseCode.SUCCESS)
     {
         //获取更新后的token和有效期
         JSONObject data = updateTokenObject.getJSONObject("data");
         String newToken = data.getString("token");
         int outTime = data.getIntValue("outtime");
         //更新数据库
         DatabaseUtils.getInstance().updateToken(newToken, outTime, EcUtils.getCurrentTimeMillis() + outTime);

         //添加新的token到FormBody
         newFormBody.add("token", newToken);
         //设置新的RequestBody
         progressRequestBody.setRequestBody(newFormBody.build());

         //重新生成URL
         HttpUrl url = chain.request().url()
                 .newBuilder()
                 .build();


         //重新构建Request
         Request newRequest = chain.request().newBuilder()
                 .post(progressRequestBody)
                 .url(url)
                 .build();

         //关闭原先的请求响应
         originalResponse.body().close();

         //打印body参数
         bodyToString(newRequest);

         //继续返回新的请求
         return chain.proceed(newRequest);
     }
     else
     {
         EventBus.getDefault().post(new ReLoginEvent());
     }
 }
```