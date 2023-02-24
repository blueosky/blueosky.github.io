## 前言

        最近看了一些关于Android V1签名的文章，了解到主要是**MANIFEST.MF  ->  CERT.SF  ->  CERT.RSA**这三个文件，分别代表了V1签名的三个步骤。怀着好奇决定拆解一个apk手动验证这三个步骤，前两步比较简单，网上搜到的文章有举例验证（如下列），但是最后一步关于CERT.RSA的都是一笔带过，于是决定自己找办法手动验证，并将过程记录下来，以供参考。

- [Android apk签名原理_互联网小熊猫的博客-CSDN博客_apk签名](https://blog.csdn.net/weixin_42600398/article/details/122843107)
- [APK签名机制之——JAR签名机制详解](https://blog.csdn.net/zwjemperor/article/details/80877305)



## 准备

### 拆解对象——[AndroidProject](https://github.com/getActivity/AndroidProject)

​	为了方便，本次验证过程选用了工程中有jks文件和apk的开源项目[AndroidProject](https://github.com/getActivity/AndroidProject)：


1. clone工程：https://github.com/getActivity/AndroidProject
2. 下载[AndroidProject_v13.1.apk](https://github.com/getActivity/AndroidProject/releases/download/13.1/AndroidProject.apk)，并解压到文件夹
3. 查看解压的文件夹下的META-INF中存在以下三个文件：CERT.RSA、CERT.SF、MANIFEST.MF

准备完毕！接下来分三步分别验证以上的三个文件。




## 验证过程

### 一、MANIFEST.MF

```java
Manifest-Version: 1.0
Built-By: Signflinger
Created-By: Android Gradle 4.1.2

Name: AndroidManifest.xml
SHA-256-Digest: q2KzfJordL6OBIlXFqWn1odDSSnOngMdFwOpD1tn8OE=

Name: assets/com.tencent.open.config.json
SHA-256-Digest: GNaNSRJAVJ4ANMiIeazMUXRPi0OovefuARO794o6ZGo=

Name: assets/h5_qr_back.png
SHA-256-Digest: bBdxfVo0wuFVXEgVd73IvsYCJEM535U7whfnS7otYiQ=

...以下省略
```

>         此文件中保存的内容其实就是逐一遍历 APK 中的所有条目，如果是目录就跳过，如果是一个文件，就用 SHA1（或者 SHA256）消息摘要算法提取出该文件的摘要然后进行 BASE64 编码后，作为「SHA1-Digest」属性的值写入到 MANIFEST.MF 文件中的一个块中。该块有一个「Name」属性， 其值就是该文件在 APK 包中的路径。
>

#### 手动验证：

以第一个文件AndroidManifest.xml为例，MANIFEST.MF中的是SHA-256-Digest因此取SHA256摘要：

- 先通过命令行的certutil命令获取AndroidManifest的SHA256摘要值

 ```java
 PS D:\> certutil -hashfile E:\f_Android_Studio\GitHub\AndroidProject\testV1\AndroidProject\AndroidManifest.xml SHA256          SHA256 的 E:\f_Android_Studio\GitHub\AndroidProject\testV1\AndroidProject\AndroidManifest.xml 哈希:
 ab62b37c9a2b74be8e04895716a5a7d687434929ce9e031d1703a90f5b67f0e1
 CertUtil: -hashfile 命令成功完成。
 ```

- 由上得SHA256值为ab62b37c9a2b74be8e04895716a5a7d687434929ce9e031d1703a90f5b67f0e1，但需注意这个值是16进制，不能直接进行Base64，可使用下面这段代码进行转换和计算：

 ```java
 package com.android.signapk;
 
 import java.util.Arrays;
 import java.util.Base64;
 
 public class TestBase64 {
 
     public static void main(String[] args) {
         test();
     }
 
     public static void test() {
         String data = "ab62b37c9a2b74be8e04895716a5a7d687434929ce9e031d1703a90f5b67f0e1";
 //        System.out.println(new String(Base64.getEncoder().encode("a".getBytes())));
         System.out.println(Arrays.toString(Base64.getEncoder().encode(hexStringToByteArray(data))));
         System.out.println(new String(Base64.getEncoder().encode(hexStringToByteArray(data))));
     }
     public static byte[] hexStringToByteArray(String s) {
         int len = s.length();
         byte[] data = new byte[len / 2];
         for (int i = 0; i < len; i += 2) {
             data[i / 2] = (byte) ((Character.digit(s.charAt(i), 16) << 4)
                     + Character.digit(s.charAt(i + 1), 16));
         }
         return data;
     }
 }
 
 ```

- 运行上面代码，打印结果如下

 ```java
 [113, 50, 75, 122, 102, 74, 111, 114, 100, 76, 54, 79, 66, 73, 108, 88, 70, 113, 87, 110, 49, 111, 100, 68, 83, 83, 110, 79, 110, 103, 77, 100, 70, 119, 79, 112, 68, 49, 116, 110, 56, 79, 69, 61]
 q2KzfJordL6OBIlXFqWn1odDSSnOngMdFwOpD1tn8OE=
 
 Process finished with exit code 0
 ```

- 结果为q2KzfJordL6OBIlXFqWn1odDSSnOngMdFwOpD1tn8OE=，与MANIFEST.MF文件中的AndroidManifest.xml条目值一致，验证成功。


### 二、CERT.SF

```java
Signature-Version: 1.0
Created-By: Android Gradle 4.1.2
SHA-256-Digest-Manifest: 2px3isU48tBZckoHtbgKVEWOCeJxbJt3QU+R2J4QOfs=
X-Android-APK-Signed: 2

Name: AndroidManifest.xml
SHA-256-Digest: kGfA0Q72nB0mp9ARlmUyKWuuO4pU2JfG3SF+GHTtAzE=

Name: assets/com.tencent.open.config.json
SHA-256-Digest: +VmZVXwqpdcf0xotqRgBXCaPTsm9SQ29eFMkyt1zFi0=

Name: assets/h5_qr_back.png
SHA-256-Digest: dt6DMMFX0xBViNp9ZZtkEleHCKSy3OYacXv5pVUY92M=

...以下省略
```

> CERT.SF文件的内容是根据MANIFEST.MF中的具体内容生成的：
> SHA1-Digest-Manifest：对整个 MANIFEST.MF 文件做 SHA1（或者 SHA256）后再用 Base64 编码
> SHA1-Digest：对 MANIFEST.MF 的各个条目做 SHA1（或者 SHA256）后再用 Base64 编码

#### 手动验证：

- 同样以AndroidManifest.xml为例，MANIFEST.MF文件中AndroidManifest对应的条目内容为（**注意！后面两个换行是必须的，不能去掉！**）：


```java
Name: AndroidManifest.xml
SHA-256-Digest: q2KzfJordL6OBIlXFqWn1odDSSnOngMdFwOpD1tn8OE=


```

- 这一步需要对上面这四行内容取摘要，这里可以把内容复制进一个文本文件，然后certutil命令对文件获取SHA256摘要。**注意！此处最好不要使用一些在线SHA网站计算，因为这段内容中存在换行，而“换行”有几种（CR、LF、CRLF），在复制粘贴时和在保存到文件时的“换行”有可能会不一样，从而导致计算出来的摘要不一样！**


```java
PS D:\> certutil -hashfile E:\f_Android_Studio\GitHub\AndroidProject\testV1\test.txt SHA256
SHA256 的 E:\f_Android_Studio\GitHub\AndroidProject\testV1\test.txt 哈希:
9067c0d10ef69c1d26a7d011966532296bae3b8a54d897c6dd217e1874ed0331
CertUtil: -hashfile 命令成功完成。
```

- 与第一步对MANIFEST.MF的处理一样，对获得的十六进制摘要9067c0d10ef69c1d26a7d011966532296bae3b8a54d897c6dd217e1874ed0331进行Base64编码，打印如下：


```java
[107, 71, 102, 65, 48, 81, 55, 50, 110, 66, 48, 109, 112, 57, 65, 82, 108, 109, 85, 121, 75, 87, 117, 117, 79, 52, 112, 85, 50, 74, 102, 71, 51, 83, 70, 43, 71, 72, 84, 116, 65, 122, 69, 61]
kGfA0Q72nB0mp9ARlmUyKWuuO4pU2JfG3SF+GHTtAzE=

Process finished with exit code 0
```
- 与CERT.SF文件中的AndroidManifest.xml条目值一致，验证成功，其余条目的计算也是依此类推。


### 三、CERT.RSA

​		第三步，最关键的一环，也是本篇介绍的重点。

> ​		“把之前生成的 CERT.SF 文件，用私钥计算出签名, 然后将签名以及包含公钥信息的数字证书一同写入 CERT.RSA 中保存。这里要注意的是，Android APK 中的 CERT.RSA 证书是自签名的，并不需要这个证书是第三方权威机构发布或者认证的，用户可以在本地机器自行生成这个自签名证书。Android 目前不对应用证书进行 CA 认证"
>

​		上面这段是其他文章的介绍，下面进行说明。

首先查看CERT.RSA文件的十六进制内容如下（这里推荐一个开源的可查看十六进制的工具——[imhex-1.24.3-Windows-Portable](https://github.com/WerWolv/ImHex)）：


```java
Hex View  00 01 02 03 04 05 06 07  08 09 0A 0B 0C 0D 0E 0F
 
00000000  30 82 04 F5 06 09 2A 86  48 86 F7 0D 01 07 02 A0  
00000010  82 04 E6 30 82 04 E2 02  01 01 31 0F 30 0D 06 09  
00000020  60 86 48 01 65 03 04 02  01 05 00 30 0B 06 09 2A  
00000030  86 48 86 F7 0D 01 07 01  A0 82 03 3B 30 82 03 37  
00000040  30 82 02 1F A0 03 02 01  02 02 04 5C C3 EC 9B 30  
00000050  0D 06 09 2A 86 48 86 F7  0D 01 01 0B 05 00 30 4B  
00000060  31 0B 30 09 06 03 55 04  06 13 02 43 4E 31 12 30  
00000070  10 06 03 55 04 08 13 09  67 75 61 6E 67 64 6F 6E  
00000080  67 31 12 30 10 06 03 55  04 07 13 09 67 75 61 6E  
00000090  67 7A 68 6F 75 31 14 30  12 06 03 55 04 03 13 0B  
000000A0  67 65 74 41 63 74 69 76  69 74 79 30 20 17 0D 31  
000000B0  39 30 36 32 37 30 32 34  30 33 36 5A 18 0F 32 31  
000000C0  31 39 30 36 30 33 30 32  34 30 33 36 5A 30 4B 31  
000000D0  0B 30 09 06 03 55 04 06  13 02 43 4E 31 12 30 10  
000000E0  06 03 55 04 08 13 09 67  75 61 6E 67 64 6F 6E 67  
000000F0  31 12 30 10 06 03 55 04  07 13 09 67 75 61 6E 67  
00000100  7A 68 6F 75 31 14 30 12  06 03 55 04 03 13 0B 67  
00000110  65 74 41 63 74 69 76 69  74 79 30 82 01 22 30 0D  
00000120  06 09 2A 86 48 86 F7 0D  01 01 01 05 00 03 82 01  
00000130  0F 00 30 82 01 0A 02 82  01 01 00 92 94 46 79 AF  
00000140  69 EC CA AE 09 D5 10 6D  D3 1B 5B 85 49 46 31 CD  
00000150  F7 77 DD 26 B8 A5 D3 AC  31 8E 80 39 AC 16 EB C5  
00000160  AC 71 14 8C 14 CF 60 CE  26 E5 7D 7C 2B 59 E8 2D  
00000170  13 18 7B DC C8 4C 36 FC  24 CC 30 75 A8 8D 6D E4  
00000180  DB 32 77 D0 49 B0 C6 CC  26 E4 8E 7F F7 26 9C D7  
00000190  F9 DE 9F 09 97 E1 8B C0  CA 19 37 13 9F 0E 74 D4  
000001A0  75 F6 AB F9 89 71 92 AF  01 F4 3E 01 8C BD B0 0B  
000001B0  6E 84 28 AA 36 A4 18 9F  84 AC 34 DC 9C C2 CC AB  
000001C0  05 28 57 0C 0B A4 FB 39  3D C1 48 D9 C6 DA EE 6F  
000001D0  F9 C2 17 2C 15 7A 72 E4  38 C7 F9 82 A5 E4 49 FE  
000001E0  0A 27 BC AC 00 52 43 52  E8 8D 65 36 F8 BC 69 76  
000001F0  9E 22 82 B4 65 15 9D 89  8F E7 2D 74 4A 40 90 85  
00000200  A7 AC F7 52 A8 BF 2E CE  5B A6 7D 14 A9 59 67 DC  
00000210  52 AC F3 DD CC C7 0C 0B  B8 5B 43 F4 AF D6 96 02  
00000220  C3 66 76 7B 32 B7 7E 59  15 32 6E F3 F1 B7 57 F4  
00000230  E2 11 DC D5 47 BF 72 0F  D1 81 A1 02 03 01 00 01  
00000240  A3 21 30 1F 30 1D 06 03  55 1D 0E 04 16 04 14 90  
00000250  73 5D B3 1F 74 7B 7B C0  5B 3F A8 60 C7 FD 03 6C  
00000260  69 E6 A7 30 0D 06 09 2A  86 48 86 F7 0D 01 01 0B  
00000270  05 00 03 82 01 01 00 65  02 4B 44 D7 C3 6F B3 F4  
00000280  55 EF F1 25 4D 9E 9E 30  78 8E 9B EE 1B E2 C0 AB  
00000290  A4 36 6D 6E D2 83 49 27  BB 8F CB 31 29 30 3C 46  
000002A0  8A 8B 7A 7D 08 B2 85 9A  A3 0A 68 EA CB 3D 44 52  
000002B0  5B D9 C7 4D 1D BE C9 6A  5D 18 22 85 4E A9 3A 21  
000002C0  3B 73 00 46 6C 15 E4 02  5C A0 A4 9F 79 36 C9 11  
000002D0  B3 2A 11 C4 B1 69 8A D1  B8 62 C3 00 B6 CB BA D5  
000002E0  3C 46 B0 93 0B C3 90 E1  6B BB 06 A8 3E 24 60 39  
000002F0  99 16 B8 50 64 E5 DE E4  3B B9 E8 75 EC F3 24 7A  
00000300  71 24 E4 13 0A B6 70 AA  9D 62 64 83 03 B8 02 4E  
00000310  67 1F B3 CA 50 69 1A 51  00 F2 8B 39 BA 3A 62 C1  
00000320  FD F3 50 37 18 A3 96 23  57 7B FE EA 6C 4E 30 C8  
00000330  BC 6F B0 A9 22 DB 12 98  56 17 7C FC 65 CB D9 CF  
00000340  4C F0 62 C9 27 22 E0 EE  D6 1E 9C 82 16 17 C7 BE  
00000350  81 A1 73 4C B4 D8 1B A3  D3 18 17 80 57 91 A5 63  
00000360  0D 8D 3C 97 C1 46 C8 2C  E1 99 87 80 E4 9B 93 FD  
00000370  2A 24 A5 B3 A4 EC 13 31  82 01 7E 30 82 01 7A 02  
00000380  01 01 30 53 30 4B 31 0B  30 09 06 03 55 04 06 13  
00000390  02 43 4E 31 12 30 10 06  03 55 04 08 13 09 67 75  
000003A0  61 6E 67 64 6F 6E 67 31  12 30 10 06 03 55 04 07  
000003B0  13 09 67 75 61 6E 67 7A  68 6F 75 31 14 30 12 06  
000003C0  03 55 04 03 13 0B 67 65  74 41 63 74 69 76 69 74  
000003D0  79 02 04 5C C3 EC 9B 30  0D 06 09 60 86 48 01 65  
000003E0  03 04 02 01 05 00 30 0D  06 09 2A 86 48 86 F7 0D  
000003F0  01 01 01 05 00 04 82 01  00 28 13 E0 48 8B F1 BD  
00000400  7D 7C 14 EE 9D 3A E4 AC  BE 6E AD 6C EE F9 32 F9  
00000410  81 96 6B 36 42 10 6A 27  F0 8A 5B 4C 79 4D B9 15  
00000420  9F F0 AE 6C 86 6C BA D0  A6 1E 5C D3 C8 96 5F F4  
00000430  3F 56 53 4E C1 BB 3C 77  D4 39 27 08 48 E4 21 E4  
00000440  14 A9 D0 0E 98 ED AC A2  E1 03 93 B3 2B 2A C4 E7  
00000450  5F 14 A0 A6 95 5E 73 54  4F EC F8 D3 02 B7 9E F1  
00000460  0E 25 85 75 44 06 24 A5  26 9C 3F 81 5E B2 3E 98  
00000470  F5 8B 9C AF 10 AE B7 1E  EA 16 05 FB D0 CA 80 EC  
00000480  6E 63 E2 80 12 6E DB 26  A6 68 98 FE 33 B5 69 A0  
00000490  A6 32 42 EF CF 4A 20 76  35 7B 55 C8 56 F4 E8 79  
000004A0  80 85 5C 2F 65 77 C3 82  8F 99 96 5A BB 44 B1 94  
000004B0  17 7E 81 10 81 53 66 CC  C8 14 1F D9 47 09 9C 31  
000004C0  A7 FC E0 02 31 A0 51 66  D7 55 22 46 BC A5 EA E1  
000004D0  EA A4 6D 29 49 92 E6 D3  1E 1F EA 4E 78 A0 B3 D3  
000004E0  61 39 C8 2F 88 A9 47 94  BC 74 D3 F6 7D 93 0E A9  
000004F0  9B 9E CE 41 FF E7 15 B4  A2                       
```

#### 验证

​	第三步简而言之就是使用[AndroidProject](https://github.com/getActivity/AndroidProject)工程中AppSignature.jks的私钥对CERT.SF文件数据进行签名获得签名结果数据，而这个结果数据就构成了CERT.RSA文件的一部分，本次验证的目标就是找出这部分数据。

- 手动验证需要的代码如下，填写好jks文件路径、keyStorePassword、CERT.SF文件路径后即可运行。

```java
package com.android.signapk;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.security.*;
import java.security.cert.CertificateException;
import java.security.interfaces.RSAPrivateKey;

public class RSAUtils {
    private static String RSA = "RSA";

    public static void main(String[] args) {
        ver();
    }

    public static void ver(){
        RSAPrivateKey pri = (RSAPrivateKey)getPrivateKey(
                "E:\\f_Android_Studio\\GitHub\\AndroidProject\\app\\AppSignature.jks",
                "AndroidProject", "AndroidProject", "AndroidProject")
        ;
        System.out.println("pri----N1:"+toHexString(pri.getModulus().toByteArray()));
        System.out.println("pri----N1:"+toHexString(pri.getPrivateExponent().toByteArray()));

        byte[] data;
        data = fileToBytes("E:\\f_Android_Studio\\GitHub\\AndroidProject\\testV1\\AndroidProject\\META-INF\\CERT.SF");

        System.out.println("待签名数据:\n" + new String(data));
        System.out.println("签名(SHA1):" + sign(data, pri));
        System.out.println("签名(SHA256):" + sign256(data, pri));
    }

    /**
     * 使用SHA1withRSA签名算法产生签名
     *
     * @param privateKey privateKey 签名时使用的私钥(16进制编码)
     * @return String 签名的返回结果(16进制编码)。当产生签名出错的时候，返回null。
     */
    public static String sign(byte[] data, PrivateKey privateKey) {
        try {
            Signature sigEng = Signature.getInstance("SHA1WithRSA");
            sigEng.initSign(privateKey);
            sigEng.update(data);
            byte[] signature = sigEng.sign();
            return toHexString(signature);
        } catch (Exception e) {
//            String info = "sign failed: " + src + " | " + e.getMessage();
            return null;
        }
    }

    /**
     * 用私钥进行数字签名
     *签名  SHA256WithRSA
     * @param data 加密数据
     * @param privateKey  私钥
     * @return 数字签名
     */
    public static String sign256(byte[] data, PrivateKey privateKey) {
        try {
            Signature sigEng = Signature.getInstance("SHA256withRSA");
            sigEng.initSign(privateKey);
            sigEng.update(data);
            byte[] signature = sigEng.sign();
            return toHexString(signature);
        } catch (Exception e) {
//            String info = "sign failed: " + src + " | " + e.getMessage();
            return null;
        }
    }

    public static byte[] fileToBytes(String filePath) {
        byte[] buffer = null;
        File file = new File(filePath);

        FileInputStream fis = null;
        ByteArrayOutputStream bos = null;

        try {
            fis = new FileInputStream(file);
            bos = new ByteArrayOutputStream();

            byte[] b = new byte[1024];

            int n;

            while ((n = fis.read(b)) != -1) {
                bos.write(b, 0, n);
            }

            buffer = bos.toByteArray();
        } catch (FileNotFoundException ex) {
//            Logger.getLogger(JmsReceiver.class.getName()).log(Level.SEVERE, null, ex);
        } catch (IOException ex) {
//            Logger.getLogger(JmsReceiver.class.getName()).log(Level.SEVERE, null, ex);
        } finally {
            try {
                if (null != bos) {
                    bos.close();
                }
            } catch (IOException ex) {
//                Logger.getLogger(JmsReceiver.class.getName()).log(Level.SEVERE, null, ex);
            } finally{
                try {
                    if(null!=fis){
                        fis.close();
                    }
                } catch (IOException ex) {
//                    Logger.getLogger(JmsReceiver.class.getName()).log(Level.SEVERE, null, ex);
                }
            }
        }
        return buffer;
    }

    /**
     * byte[]转16进制HEXString
     * @param b
     * @return
     */
    private static final String HEX_CHARS = "0123456789ABCDEF";
    public static String toHexString(byte[] b) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < b.length; i++) {
            sb.append(RSAUtils.HEX_CHARS.charAt(b[i] >>> 4 & 0x0F));
            sb.append(RSAUtils.HEX_CHARS.charAt(b[i] & 0x0F));
        }
        return sb.toString();
    }

    public static byte[] hexStringToByteArray(String s) {
        int len = s.length();
        byte[] data = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            data[i / 2] = (byte) ((Character.digit(s.charAt(i), 16) << 4)
                    + Character.digit(s.charAt(i + 1), 16));
        }
        return data;
    }

    /**
     * 由keystore证书密钥库文件获取私钥
     *
     * @param keyStorePath     密钥库文件路径
     * @param keyStorePassword 密钥库文件密码
     * @param alias            指定密钥对的别名
     * @param aliasPassword    密钥密码
     * @return key   私钥，PrivateKey类型
     * @throws NoSuchAlgorithmException
     * @throws KeyStoreException
     * @throws UnrecoverableKeyException
     * @throws IOException
     * @throws CertificateException
     */
    public static PrivateKey getPrivateKey(String keyStorePath, String keyStorePassword,
                                           String alias, String aliasPassword){
        KeyStore ks = null;
        try {
            ks = getKeyStore(keyStorePath, keyStorePassword);
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (CertificateException e) {
            e.printStackTrace();
        }
        PrivateKey key = null;
        if (null != ks) {
            try {
                key = (PrivateKey) ks.getKey(alias, aliasPassword.toCharArray());
            } catch (KeyStoreException e) {
                e.printStackTrace();
            } catch (NoSuchAlgorithmException e) {
                e.printStackTrace();
            } catch (UnrecoverableKeyException e) {
                e.printStackTrace();
            }
        }
        return key;
    }

    public static KeyStore getKeyStore(String keyStorePath, String keyStorePassword) throws KeyStoreException, NoSuchAlgorithmException, CertificateException {
        FileInputStream is = null;
        KeyStore ks = null;
        try {
            is = new FileInputStream(keyStorePath);
//            ks = KeyStore.getInstance(KEY_STORE);
            ks = KeyStore.getInstance("jks");
            ks.load(is, keyStorePassword.toCharArray());
        } catch (IOException | CertificateException e) {
//            log.info("IO流异常：{}", e);
        } finally {
            if (is != null) {
                try {
                    is.close();
                } catch (IOException e) {
                    is = null;
//                    log.info("关闭流异常：{}", e);
                }
            }
        }
        return ks;
    }
}
```

- 运行上面的代码，打印结果中的签名如下：

```java
签名(SHA256):2813E0488BF1BD7D7C14EE9D3AE4ACBE6EAD6CEEF932F981966B3642106A27F08A5B4C794DB9159FF0AE6C866CBAD0A61E5CD3C8965FF43F56534EC1BB3C77D439270848E421E414A9D00E98EDACA2E10393B32B2AC4E75F14A0A6955E73544FECF8D302B79EF10E258575440624A5269C3F815EB23E98F58B9CAF10AEB71EEA1605FBD0CA80EC6E63E280126EDB26A66898FE33B569A0A63242EFCF4A2076357B55C856F4E87980855C2F6577C3828F99965ABB44B194177E8110815366CCC8141FD947099C31A7FCE00231A05166D7552246BCA5EAE1EAA46D294992E6D31E1FEA4E78A0B3D36139C82F88A94794BC74D3F67D930EA99B9ECE41FFE715B4A2

Process finished with exit code 0
```

- 在imhex中搜索以上签名结果的16进制字符串，最后显示签名结果就是CERT.RSA文件的最后一段（28 13 E0 48............E7 15 B4 A2），验证成功！！！





## 参考/引用

- [Android apk签名原理_互联网小熊猫的博客-CSDN博客_apk签名](https://blog.csdn.net/weixin_42600398/article/details/122843107)
- [APK签名机制之——JAR签名机制详解](https://blog.csdn.net/zwjemperor/article/details/80877305)
- https://github.com/getActivity/AndroidProject
- https://github.com/WerWolv/ImHex
