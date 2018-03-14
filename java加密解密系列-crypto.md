---
title: JAVA加密与解密
date: 2016-07-28 14:17:59
tags: Java
---
# Java加密与解密

## 1、摘要算法

  消息摘要算法包含MD、SHA和MAC共3大系列，常用于验证数据的完整性，是数字签名算法的核心算法。

  任何消息经过散列函数处理后，都会获得唯一的散列值。这一过程称为“消息摘要”，其散列值称为“数字指纹”，自然其算法就是“消息摘要算法”。

  消息摘要算法一直是非对称加密算法中一项举足轻重的关键性算法。

### 1.1、MD（消息摘要算法）

MD5算法是典型的消息摘要算法，其前身有MD2、MD3和MD4算法，它由MD4、MD5、MD2算法改进而来。不论是哪一种MD算法，它们都需要获得一个随机长度的信息产生一个128位的信息摘要。

#### 1.1.1 MD算法实现

```java
public abstract class MDCoder {

  public static byte[] encodeMD5ByJDK(byte[] data) throws NoSuchAlgorithmException {
      // 初始化MessageDigest
      MessageDigest md = MessageDigest.getInstance("MD5");
      // 执行消息摘要
      return md.digest(data);
  }

  public static byte[] encodeMD5ByBC(byte[] data) throws NoSuchAlgorithmException {
      // 加入BouncyCastleProvider支持
      Security.addProvider(new BouncyCastleProvider());
      // 初始化MessageDigest
      MessageDigest md = MessageDigest.getInstance("MD5");
      // 执行消息摘要
      return md.digest(data);
  }


  public static byte[] encodeMD5ByCC(byte[] data) throws NoSuchAlgorithmException {
      return DigestUtils.md5(data);
  }
}
```

#### 1.1.2、三种实现方式的差异

三种实现方式以Sun提供的实现为基础，在算法支持上和方法易用性上提供了
更好的扩展与支持。

- Sun

Sun提供的算法实现较为底层，支持MD2和MD5两种算法。但缺少相应的进制转换实现，不能将其字节数组形式的摘要信息转为十六进制字符串。

- Bouncy Castle

Bouncy Castle是对Sun的友善补充，提供了对MD4算法的支持。支持多种形式
的参数，支持十六进制字符串形式的摘要信息。

- Commons Codec

如果仅仅需要实现MD5算法的话，使用Commons Codec完成摘要处理是一个不错的选择。它支持多种形式的参数，支持十六进制字符串形式的摘要信息。

### 1.2、MAC（消息认证码算法）

  MAC算法结合了MD5和SHA算法的优势，并加入密钥的支持，是一种更为安全的消息摘要算法，也常把MAC称为HMAC。

  MAC算法主要集合了MD和SHA两大系列摘要算法。MD系列算法有HmacMD2、HmacMD4和HmacMD5三种算法；SHA系列算法HmacSHA1、HmacSHA224、HmacSHA256、HmacSHA384和HmacSHA5122五种算法。

#### 1.2.1、实现

在JDK中，提供了HmacMD5、HmacSHA1、HmacSHA256、HmacSHA384和HmacSHA512四种算法，而Bouncy Castle补充了HmacMD2、HmacMD4和HmacSHA224三种算法支持。

```java
public abstract class MACCoder {

  /**
   * 初始化HmacMD5密钥
   *
   * @return byte[] 密钥
   * @throws NoSuchAlgorithmException
   */
  public static byte[] initHmacMD5KeyByJDk() throws NoSuchAlgorithmException {
      // 初始化KeyGenerator
      KeyGenerator keyGenerator = KeyGenerator.getInstance("HmacMD5");
      // 产生密钥
      SecretKey secretKey = keyGenerator.generateKey();
      // 获得密钥
      return secretKey.getEncoded();
  }


  public static byte[] encodeHmacMD5ByJDK(byte[] data, byte[] key) throws NoSuchAlgorithmException, InvalidKeyException {
      // 还原密钥
      SecretKey secretKey = new SecretKeySpec(key, "HmacMD5");
      // 实例化
      Mac mac = Mac.getInstance(secretKey.getAlgorithm());
      // 初始化Mac
      mac.init(secretKey);
      // 执行消息摘要
      return mac.doFinal(data);
  }

  public static byte[] initHmacMD5KeyByBC() throws NoSuchAlgorithmException {
      // 加入BouncyCastleProvider支持
      Security.addProvider(new BouncyCastleProvider());
      // 初始化
      KeyGenerator keyGenerator = KeyGenerator.getInstance("HmacMD5");
      // 产生密钥
      SecretKey secretKey = keyGenerator.generateKey();
      // 获得密钥
      return secretKey.getEncoded();
  }

  public static byte[] encodeHmacMD5ByBC(byte[] data, byte[] key) throws NoSuchAlgorithmException, InvalidKeyException {

      Security.addProvider(new BouncyCastleProvider());
      // 还原密钥
      SecretKey secretKey = new SecretKeySpec(key, "HmacMD5");
      // 实例化
      Mac mac = Mac.getInstance(secretKey.getAlgorithm());
      // 初始化Mac
      mac.init(secretKey);
      // 执行消息摘要
      return mac.doFinal(data);
  }
}
```

### 1.3、SHA（安全散列算法）

  SHA算法基于MD4算法基础上，作为MD算法的继任者，成为了新一代的消息摘要算法的代表。SHA与MD算法不同之处主要在于摘要长度，SHA算法的摘要长度更长，安全性更高。

  在JDK6中，MessageDigest类支持的SHA算法几乎涵盖了我们目前所知的全部SHA系统算法，主要包含SHA-1、SHA-256、SHA-384和SHA-512四种算法，通过Bouncy Castle
  可支持SHA-224算法。

#### 1.3.1 SHA实现

```java
public abstract class SHACoder {

  public static byte[] encodeSHAByJDK(byte[] data) throws NoSuchAlgorithmException {
      // 初始化MessageDigest
      MessageDigest md = MessageDigest.getInstance("SHA");
      // 执行消息摘要
      return md.digest(data);
  }

  public static byte[] encodeSHAByBC(byte[] data) throws NoSuchAlgorithmException {
      Security.addProvider(new BouncyCastleProvider());
      // 初始化MessageDigest
      MessageDigest md = MessageDigest.getInstance("SHA");
      // 执行消息摘要
      return md.digest(data);
  }

  public static byte[] encodeSHAByCC(byte[] data) throws NoSuchAlgorithmException {
      return DigestUtils.sha1(data);
  }
}
```

<!-- # Shiro提供的摘要算法

<center>
  <img src="/java-crypto/2016728141946.png"/>
  Shiro提供的摘要算法结构图
</center> -->

## 2、对称加密算法

根据加密对称算法加密方式又分为密码和分组密码，其分组密码工作模式又可为ECB、CBC、CFB、OFB和CTR等，密钥长度决定了加密算法的安全性。

目前已知的可通过Java语言实现的对称加密算法大约有20多种，JDK6仅仅提供了部分算法实现，如DES、DESede、AES、Blowfish，以及RC2和RC4算法等。其他算法（如IDEA算法）需要通过第三方加密包Bouncy Castle提供了实现。在Java实现层面上，DES、DECede、AES和IDEA这4种算法略有不同。

- DES和DESede算法在使用密钥材料还原密钥时，建议使用各自相应的密钥材料实现类（DES算法对应DESKeySpec类、DESede算法对应DESedeKeySpec类）完成相应转换操作。

- AES算法在使用密钥材料还原密钥时，则需要使用一般密钥材料实现类(SecretKeySpec类)完成相应转换操作。其他对称加密算法可参照该方式实现，如RC2、RC4、Blowfish以及IDEA等算法可参照AES算法实现方式做相应实现。

- IDEA算法实现JDK6未能提供，需要依赖第三方加密组件包Bouncy Castle提供支持的对称加密算法可参照该算法实现方式做相应实现。

### 2.1、数据加密标准——DES

### 2.2、高级数据加密标准——AES

### 2.3、基于口令加密——PBE

PBE算法是一种基于口令的加密算法，在特点在于口令由用户自己掌管，采用随机数杂凑多重加密
    等方法保证数据的安全性。

## 3、非对称加密算法

  非对称加密算法与对称加密算法的主要差别在于非对称加密算法和解密的密钥不同，一个公开， 称为公钥；
  一个保密，称为私钥。非对称加密算法解决了对称加密算法密钥分配问题，并极大地提高了算法安全性。
  多种B2C或B2B应用均使用非对称算法作为数据加密的核心算法。

### 3.1、密钥交换算法——DH

### 3.2、RSA

    RSA算法公钥长度远小于私钥长度，并遵循“公钥加密，私钥解密”和“私钥加密，公钥解密”这两项加密/解密原则。

### 3.3、非对称算法应用

  非对称算法并不直接对网络数据进行加密/解密，而是用于交换对称加密算法的秘密密钥。最终使用对称加密算法进行真正的加密/解密。

## 4、数字签名算法

  数字签名算法可以看做是一种带有密钥的消息摘要算法，并且这种密钥包含了公钥和私钥。也就是说，数字签名算法是非对称加密算法和消息
