---
title: Java md5加密
date: 2016-05-28 22:48:28
categories: java
tags:
	- java
---

在应用开发或者web开发中，经常需要对用户信息进行存储，但是明文存储是不安全的，我们需要对信息进行加密。MD5作为不可逆的加密算法，被广泛使用，即原理上不能解密，只能穷举。
<!--more-->

### 代码 

	```java
	/**
     * md5加密函数
     * @param str 要加密的字符串
     * @return 加密后的字符串
     */
    public static String md5(String str){
        try {
            //指定加密算法类型
            MessageDigest md5 = MessageDigest.getInstance("MD5");
            //将要加密的字符串转换成byte数组
            byte[] bytes = md5.digest(str.getBytes());
            StringBuffer stringBuffer = new StringBuffer();
            for (byte b : bytes){
                int i = b & 0xff;
                //将int类型的i转换成16进制字符
                String hexString = Integer.toHexString(i);
                if(hexString.length() < 2){
                    hexString = "0"+hexString;
                }
                //将字符串拼接成32位
                stringBuffer.append(hexString);
            }
            return stringBuffer.toString();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        return null;
    }
	```
