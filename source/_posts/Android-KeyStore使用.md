---
title: Android KeyStore使用，公钥，私钥签名
date: 2018-04-25 22:01:07
tags: [Android]
categories: [Android]
---

> **Android KeyStore**的使用，可以生成公钥和使用私钥签名：
>
> 相比较与Java的RSA生成的方式，使用Android KeyStore会更加安全。
> 具体的使用可以参加如下Java类：

<!-- more --> 

```java
import android.content.Context;
import android.os.Build;
import android.security.KeyPairGeneratorSpec;
import android.util.Base64;

import java.io.IOException;
import java.math.BigInteger;
import java.security.InvalidAlgorithmParameterException;
import java.security.KeyPairGenerator;
import java.security.KeyStore;
import java.security.KeyStoreException;
import java.security.NoSuchAlgorithmException;
import java.security.NoSuchProviderException;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.Signature;
import java.security.UnrecoverableEntryException;
import java.security.cert.CertificateException;
import java.util.Calendar;

import javax.security.auth.x500.X500Principal;

/**
 * Created by Mia W W Tong on 2018/4/24.
 * <p/>
 * Desc: KeyStoreUtils.java
 */
public class KeyStoreUtils {

    private final static String TOUCHID_KEY_STORE = "KeyStore";// Alais可以随意定义

    public static KeyStore initKeyStore(Context context) {
        try {
            KeyStore keyStore = KeyStore.getInstance("AndroidKeyStore");
            keyStore.load(null);

            if (!keyStore.containsAlias(TOUCHID_KEY_STORE)) {
                // 创建KeyPair过程，创建出来的PrivateKey实际上是没法获取的，但是PublicKey可以获取；
                // 但是Privatekey可以直接进行sign这些操作。
                Calendar start = Calendar.getInstance();
                Calendar end = Calendar.getInstance();
                end.add(Calendar.YEAR, 20);
                if (Build.VERSION.SDK_INT >= 18) {
                    KeyPairGeneratorSpec spec = new KeyPairGeneratorSpec.Builder(context)
                            .setAlias(TOUCHID_KEY_STORE)
                            .setSubject(new X500Principal(String.format("CN=%s, O=%s", "SecureLocalStorage", context.getPackageName())))
                            .setSerialNumber(BigInteger.ONE)
                            .setStartDate(start.getTime())
                            .setKeySize(2048)
                            .setEndDate(end.getTime()).build();
                    KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA","AndroidKeyStore");
                    generator.initialize(spec);
                    generator.generateKeyPair();
                }
            }
            return keyStore;
        } catch (IOException | CertificateException | KeyStoreException | NoSuchAlgorithmException | NoSuchProviderException | InvalidAlgorithmParameterException e) {
            LOG.error("initKeyStore exception {}", e);
            return null;
        }
    }

    public static String getPublicKey(Context context) {
        KeyStore store = initKeyStore(context);
        if (store != null) {
            // store key encrypted with keystore key pair
            KeyStore.PrivateKeyEntry privateKeyEntry = null;
            try {
                privateKeyEntry = (KeyStore.PrivateKeyEntry) store.getEntry(TOUCHID_KEY_STORE, null);
                PublicKey publicKey = privateKeyEntry.getCertificate().getPublicKey();
                LOG.debug("PublicKey is {}", Base64.encode(publicKey.getEncoded(), Base64.DEFAULT));
                return Base64.encodeToString(publicKey.getEncoded(), Base64.DEFAULT);
            } catch (NoSuchAlgorithmException | KeyStoreException | UnrecoverableEntryException e) {
                LOG.error("getPublickey {}", e);
                return "";
            }
        }

        return "";
    }

    public static String sign(Context context, String value) throws Exception {
        KeyStore store = initKeyStore(context);
        if (store != null) {
            // store key encrypted with keystore key pair
            KeyStore.PrivateKeyEntry privateKeyEntry = null;
            privateKeyEntry = (KeyStore.PrivateKeyEntry) store.getEntry(TOUCHID_KEY_STORE, null);
            PrivateKey privateKey = privateKeyEntry.getPrivateKey();

            // sign process
            Signature sign = Signature.getInstance("SHA256withRSA");// SHA256withRSA签名算法
            sign.initSign(privateKey);//设置私钥
            sign.update(value.getBytes());//设置明文
            byte[] bytes = sign.sign();//加密
            LOG.debug("Sign Result：" + Base64.encodeToString(bytes, Base64.DEFAULT));
            return Base64.encodeToString(bytes, Base64.DEFAULT);
        }

        return "";
    }

    public static void deleteAlias() throws Exception {
        KeyStore keyStore = KeyStore.getInstance("AndroidKeyStore");
        keyStore.load(null);

        if (keyStore.containsAlias(TOUCHID_KEY_STORE)) {
            keyStore.deleteEntry(TOUCHID_KEY_STORE);
        }
    }
}
```