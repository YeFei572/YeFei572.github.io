---
layout: post
tags: 小程序
categories: 技术分享
title:  "java代码小程序获取的unionId,openId,或者其他参数"
---


> 记录一下java代码获取小程序的一些相关信息

### 1. 先说明一下整个流程:

- 微信通过 `wx.login()` 方法拿到 `code`;
- 然后通过 `wx.getUserInfo` 方法拿到微信给回的加密报文;
- 通过加密报文以及加密算法的初始向量等参数向开发者后台发送请求进行解密报文并获取解密后的用户信息

### 2. 贴一下微信小程序端的部分代码:

小程序端的代码主要是获取到加密报文, code, 还有加密算法的初始向量等参数:

```js
wx.login({
    success: function (res) {
        if (res.code) {
            // 3获取用户信息 encryptedData iv 解密出 unionId
            wx.getUserInfo({
                success: function (respon) {
                    wx.request({
                        // 开发者服务器, 解密报文的接口
                        url: `http://localhost:8080/api/getUnionId`,
                        method: 'POST',
                        header: {
                            'Content-type': 'application/x-www-form-urlencoded'
                        },
                        data: {
                            encryptedData: respon.encryptedData,
                            iv: respon.iv,
                            code: res.code
                        },
                        success: function (data) {
                            if (data.data.status == 1) {
                                wx.setStorageSync('userInfo_self', data.data.userInfo)
                                app.globalData.userInfo_self = data.data.userInfo
                            } else {
                                console.log('unionId解密失败')
                                wx.hideLoading()
                            }
                        },
                        fail: function () {
                            console.log('getUnionId系统错误')
                            wx.hideLoading()
                        }
                    })
                },
                fail: function () {
                    console.log('wx.getUserInfo获取用户信息失败')
                    wx.hideLoading()
                }
            })
        } else {
            console.log('获取用户登录态code失败' + res.errMsg)
            wx.hideLoading()
        }
    }
})
```

等拿到了加密报文`encryptedData`和加密算法的初始向量`iv`还有`code就向开发者服务器解密报文接口发起请求,并获取到解密后的报文,也就是用户的相关信息.

### 3. 开发者服务器解密报文相关代码:

当用户从小程序携带着三个参数到后端来进行解密, 这时候所有的工作就在java代码里面实现了</br>
controller层代码:

```java
/**
 * 通过密文,加密算法的初始向量,jscode获得微信用户基础信息
 * @param encryptedData
 * @param iv
 * @param code
 * @return
 */
@PostMapping(
        value = "/api/getUnionId",
        produces = MediaType.APPLICATION_JSON_VALUE
)
public Map getUnionId(String encryptedData, String iv,
                      String code) {
    return wxAppletUserInfo.getSessionKeyOrOpenId(code, encryptedData, iv);
}
```

service层的代码:

```java
@Component
public class WXAppletUserInfo {
    private static Logger log = Logger.getLogger(WXAppletUserInfo.class);
    private final static String requestUrl = "https://api.weixin.qq.com/sns/jscode2session"; 
    private final static appId = "xxxxx";
    private final static String secret = "xxxxxx";

    /**
     * 获取微信小程序 session_key 和 openid
     *
     * @param code 调用微信登陆返回的Code
     * @return
     * @author YeFei
     */
    public Map getSessionKeyOrOpenId(String code, String encryptedData, String iv) {
        //微信端登录code值
        Map<String, String> requestUrlParam = new HashMap<>();
        requestUrlParam.put("appid", appId);    //开发者设置中的appId
        requestUrlParam.put("secret", secret);    //开发者设置中的appSecret
        requestUrlParam.put("js_code", code);    //小程序调用wx.login返回的code
        requestUrlParam.put("grant_type", GRANT_TYPE);    //默认参数
    
        //发送post请求读取调用微信 https://api.weixin.qq.com/sns/jscode2session 接口获取openid用户唯一标识
        JSONObject res1 = JSON.parseObject(sendPost(requestUrl, requestUrlParam));
        Map result = new HashMap();
        if (res1 != null && res1.get("errcode") != null) {
            result.put("status", 0);
            result.put("msg", "解析失败,请检查入参code!");
            return result;
        } else {
            JSONObject res2 = getUserInfo(encryptedData, res1.get("session_key").toString(), iv);
            if (null != res2) {
                Map userInfo = new HashMap();
                result.put("status", 1);
                result.put("msg", "解密成功!");
                userInfo.put("openId", res2.get("openId"));
                userInfo.put("nickName", res2.get("nickName"));
                userInfo.put("gender", res2.get("gender"));
                userInfo.put("city", res2.get("city"));
                userInfo.put("province", res2.get("province"));
                userInfo.put("country", res2.get("country"));
                userInfo.put("avatarUrl", res2.get("avatarUrl"));
                userInfo.put("unionId", res2.get("unionId"));
                result.put("userInfo", userInfo);
                return result;
            }
        }
        result.put("status", 0);
        result.put("msg", "解析失败,请检查入参!");
        return result;
    }
    
    /**
     * 解密用户敏感数据获取用户信息
     *
     * @param sessionKey    数据进行加密签名的密钥
     * @param encryptedData 包括敏感数据在内的完整用户信息的加密数据
     * @param iv            加密算法的初始向量
     * @return
     * @author YeFei
     */
    public static JSONObject getUserInfo(String encryptedData, String sessionKey, String iv) {
        // 被加密的数据
        byte[] dataByte = Base64.decode(encryptedData);
        // 加密秘钥
        byte[] keyByte = Base64.decode(sessionKey);
        // 偏移量
        byte[] ivByte = Base64.decode(iv);
        try {
            // 如果密钥不足16位，那么就补足.  这个if 中的内容很重要
            int base = 16;
            if (keyByte.length % base != 0) {
                int groups = keyByte.length / base + (keyByte.length % base != 0 ? 1 : 0);
                byte[] temp = new byte[groups * base];
                Arrays.fill(temp, (byte) 0);
                System.arraycopy(keyByte, 0, temp, 0, keyByte.length);
                keyByte = temp;
            }
            // 初始化
            Security.addProvider(new BouncyCastleProvider());
            Cipher cipher = Cipher.getInstance("AES/CBC/PKCS7Padding", "BC");
            SecretKeySpec spec = new SecretKeySpec(keyByte, "AES");
            AlgorithmParameters parameters = AlgorithmParameters.getInstance("AES");
            parameters.init(new IvParameterSpec(ivByte));
            cipher.init(Cipher.DECRYPT_MODE, spec, parameters);// 初始化
            byte[] resultByte = cipher.doFinal(dataByte);
            if (null != resultByte && resultByte.length > 0) {
                String result = new String(resultByte, "UTF-8");
                return JSON.parseObject(result);
            }
        } catch (NoSuchAlgorithmException e) {
            log.error(e.getMessage(), e);
        } catch (NoSuchPaddingException e) {
            log.error(e.getMessage(), e);
        } catch (InvalidParameterSpecException e) {
            log.error(e.getMessage(), e);
        } catch (IllegalBlockSizeException e) {
            log.error(e.getMessage(), e);
        } catch (BadPaddingException e) {
            log.error(e.getMessage(), e);
        } catch (UnsupportedEncodingException e) {
            log.error(e.getMessage(), e);
        } catch (InvalidKeyException e) {
            log.error(e.getMessage(), e);
        } catch (InvalidAlgorithmParameterException e) {
            log.error(e.getMessage(), e);
        } catch (NoSuchProviderException e) {
            log.error(e.getMessage(), e);
        }
        return null;
    }
    
    /**
     * 向指定 URL 发送POST方法的请求
     *
     * @param url      发送请求的 URL
     * @param paramMap 请求参数
     * @return 所代表远程资源的响应结果
     */
    public static String sendPost(String url, Map<String, ?> paramMap) {
        PrintWriter out = null;
        BufferedReader in = null;
        String result = "";

        String param = "";
        Iterator<String> it = paramMap.keySet().iterator();

        while (it.hasNext()) {
            String key = it.next();
            param += key + "=" + paramMap.get(key) + "&";
        }

        try {
            URL realUrl = new URL(url);
            // 打开和URL之间的连接
            URLConnection conn = realUrl.openConnection();
            // 设置通用的请求属性
            conn.setRequestProperty("accept", "*/*");
            conn.setRequestProperty("connection", "Keep-Alive");
            conn.setRequestProperty("Accept-Charset", "utf-8");
            conn.setRequestProperty("user-agent", "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1;SV1)");
            // 发送POST请求必须设置如下两行
            conn.setDoOutput(true);
            conn.setDoInput(true);
            // 获取URLConnection对象对应的输出流
            out = new PrintWriter(conn.getOutputStream());
            // 发送请求参数
            out.print(param);
            // flush输出流的缓冲
            out.flush();
            // 定义BufferedReader输入流来读取URL的响应
            in = new BufferedReader(new InputStreamReader(conn.getInputStream(), "UTF-8"));
            String line;
            while ((line = in.readLine()) != null) {
                result += line;
            }
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }
        //使用finally块来关闭输出流、输入流
        finally {
            try {
                if (out != null) {
                    out.close();
                }
                if (in != null) {
                    in.close();
                }
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }
        return result;
    }
}
```

看一下返回结果:

```json
{
  "msg": "解密成功!",
  "userInfo": {
    "country": "China",
    "unionId": "oCDbm0bTnxxxxVvOvVxxxxxx",
    "gender": 1,
    "province": "Guangdong",
    "city": "Shenzhen",
    "avatarUrl": "https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTxxxxxxp9LaoopLl7Ts4DGTswxSAtpozvKibJicFrVQRArpV9XuaFnEicYazTGSoCoicvBeBw/132",
    "openId": "oKq935xxxxxxhojuANk02xxxxx",
    "nickName": "吉祥如意"
  },
  "status": 1
}
```

### 4. 最后放一些相关依赖

这里放的是gradle依赖, 如果不会用可以在中央仓库转成maven依赖

```js
// https://mvnrepository.com/artifact/org.bouncycastle/bcprov-jdk16
compile group: 'org.bouncycastle', name: 'bcprov-jdk16', version: '1.46'
compile group: 'commons-httpclient', name: 'commons-httpclient', version: '3.1'
// https://mvnrepository.com/artifact/com.alibaba/fastjson
compile group: 'com.alibaba', name: 'fastjson', version: '1.2.47'
```

有什么疑问欢迎留言交流..
