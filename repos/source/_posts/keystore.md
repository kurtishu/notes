﻿---
title: Keystore
date: 2016-07-15
categories: Android
tags: [Keystore, SSL]
---

**Keytool**  命令   
Keytool 是一个JAVA环境下的安全钥匙与证书的管理工具.它管理一个存储了私有钥匙和验证相应公共钥匙的与它们相关联的X.509 证书链的keystore(相当一个数据库，里面可存放多个X.509标准的证书)..

<br/>

###  Keytool 常用命令：
检查一个keystore

    keytool -list -v -keystore c:\server.jks 
 
检查一个keystore的内容

    keytool -list -v -keystore c:\server.jks 
 
添加一个信任根证书到keystore文件

    keytool -import -alias newroot -file root.cer -keystore server.jks

导入CA签署好的证书 

    keytool -import -keystore c:\server.jks -alias tomcat -file c:\cert.txt 

从 KEYSTORE中导出一个证书文件

    keytool -export -alias myssl -keystore server.jks -rfc -file server.cer 
    *备注： "-rfc" 表示以base64输出文件，否则以二进制输出。

从KEYSTORE中删除一个证书

   keytool -delete -keystore server.jks -alias tomcat   
   *备注：删除了别名为tomcat的证书。

---- 
推荐一个生成Keystore的工具 ** Keystore GUI 1.6 **
----  

### Tomcat 支持SSL配置  

修改tomcat安装目录下的conf\server.xml文件  
 
``` 
<Connector port="8443" protocol="HTTP/1.1" SSLEnabled="true"
               maxThreads="150" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS"
 			   keystoreFile="C:\home\tomcat.jks" keystorePass="123456"/>

```

<br/>
...


```java 
private static KeyStore readKeyStore() throws Exception {
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());

        InputStream fis = null;
        InputStream fis2 = null;
        try {
            CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
            fis = PSSApp_.getInstance().getAssets().open("pss.cer");
            fis2 = PSSApp_.getInstance().getAssets().open("openfire.cer");
            keyStore.load(null);
            keyStore.setCertificateEntry("localhost", certificateFactory.generateCertificate(fis));
            keyStore.setCertificateEntry("openfire", certificateFactory.generateCertificate(fis2));
        } finally {
            if (fis != null) {
                fis.close();
            }
            if (fis2 != null) {
                fis2.close();
            }
        }
        return keyStore;
    }


    public static void doSSLTest() {

        OkHttpClient.Builder mBuilder = new OkHttpClient.Builder();

        try {
            KeyStore keyStore = readKeyStore(); //your method to obtain KeyStore
            SSLContext sslContext = SSLContext.getInstance("SSL");
            TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            trustManagerFactory.init(keyStore);
            KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
            keyManagerFactory.init(keyStore, "123456".toCharArray());
            sslContext.init(null, trustManagerFactory.getTrustManagers(), new SecureRandom());
            mBuilder.sslSocketFactory(sslContext.getSocketFactory());

        } catch (Exception e) {
            Log.i("Kurtis", "Exception" + e.getMessage());
        }

        HostnameVerifier hostnameVerifier = new HostnameVerifier() {
            @Override
            public boolean verify(String hostname, SSLSession session) {
                Log.d(TAG, "Trust Host :" + hostname);
                return true;
            }
        };
        mBuilder.hostnameVerifier(hostnameVerifier);

        OkHttpClient mClient = mBuilder.build();
        okhttp3.Request request = new okhttp3.Request.Builder()
                .get()
                .url("https://192.168.188.17:9091/")
                .build();

        mClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.i("Kurtis", "onFailure");
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                Log.i("Kurtis", "onResponse");
            }
        });

    }

```  
