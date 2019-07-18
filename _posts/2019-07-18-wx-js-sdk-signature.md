---
layout: post
title:  "微信js-sdk定位服务签名生成"
date:   2019-07-18 15:51:22
categories: 微信开发
tags: 公众号开发
author: ddmcc
---

* content
{:toc}

公众号要用到定位服务，需要后台提供签名接口，记录一下。

签名是由jsapi_ticket+随机字符串+时间戳+页面请求url拼接后通过SHA-1加密生成的。参数必须全部小写，url是调用定位服务的页面的url。





## 获取token

1,jsapi_ticket是由参数access_token获取的，而access_token又是由appid和secret获取的。所以先要去获取access_token。

    /**
     *
     * 获取accessToken
     *
     * @return  access_token
     */
    public String getAccessToken() {
        Object appId = DictionaryCache.systemConfig.get("h5.app.id");//应用ID
        Object appSecret = DictionaryCache.systemConfig.get("h5.app.secret");//(应用密钥)
        String url ="https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid="+appId+"&secret="+appSecret+"";
        JSONObject backData = doGet(url);
        if (backData == null || backData.getInteger("errcode") != null || (accessToken = backData.getString("access_token")) == null) {
            logger.info(String.format("获取access_token失败 response -> %s   ", backData));
            return null;
        }
        return accessToken;
    }
	
	
因为access_token是有有效期的，现在默认是7200s，所以需要全局缓存，以便后面的请求使用。而且每天这个接口有调用次数限制。好像是2000次？
**这边由于项目用的Spring框架，bean是单例的所以直接保存在本类中了**

	// 微信公众号access_token 默认保存7200秒
    private volatile String accessToken = "";
	
	// 微信公众号TICKET 默认保存7200秒
    private volatile String ticket = "";


## 获取jsapi_ticket

2，获取jsapi_ticket，jsapi_ticket和token一样也是7200秒，就是两小时

    /**
     *
     *  获取jsApiTicket
     *
     * @return  ticket
     */
    public String getJSApiTicket() {
        String accessToken;
        if (StringUtil.isEmpty(this.accessToken)) {
            synchronized (this) {
                if (StringUtil.isEmpty(this.accessToken)) {
                    accessToken = getAccessToken();
                } else {
                    accessToken = this.accessToken;
                }
            }
        } else {
            accessToken = this.accessToken;
        }

        if (StringUtil.isEmpty(accessToken)) {
            return null;
        }
        String url = "https://api.weixin.qq.com/cgi-bin/ticket/getticket?access_token="+accessToken+"&type=jsapi";
        JSONObject backData = doGet(url);
        if (backData == null || backData.getInteger("errcode") != 0 || (ticket = backData.getString("ticket")) == null) {
            logger.info(String.format("获取jsApiTicket失败 response -> %s   ", backData));
            return null;
        }
        return ticket;
    }
	

## 生成签名

	// 获取签名需要传入url，动态的获取url以便其他地方也能用。
    public Map<String, String> getParams(String url) {
        try {
            String ticket;
            if (StringUtil.isEmpty(this.ticket)) {
                synchronized (this) {
                    if (StringUtil.isEmpty(this.ticket)) {
                        ticket = getJSApiTicket();
                    } else {
                        ticket = this.ticket;
                    }
                }
            } else {
                ticket = this.ticket;
            }
            if (StringUtil.isEmpty(ticket)) {
                Map<String, String> result = Maps.newHashMap();
                result.put("flag", "-1");
                return result;
            }
            return sign(ticket, url);
        } catch (Exception e) {
            e.printStackTrace();
            Map<String, String> result = Maps.newHashMap();
            result.put("flag", "-1");
            return result;
        }
    }

	// 签名的方法
    public Map<String, String> sign(String ticket, String url) {
        Map<String, String> ret = Maps.newHashMap();
        String nonceStr = createNonceStr();
        String timestamp = createTimestamp();
        try {
            String string1 = "jsapi_ticket=" + ticket +
                    "&noncestr=" + nonceStr +
                    "&timestamp=" + timestamp +
                    "&url=" + url;

            MessageDigest crypt = MessageDigest.getInstance("SHA-1");
            crypt.reset();
            crypt.update(string1.getBytes("UTF-8"));
            ret.put("signature", byteToHex(crypt.digest()));
        } catch (NoSuchAlgorithmException | UnsupportedEncodingException e) {
            e.printStackTrace();
            ret.put("flag", "-1");
            return ret;
        }
        ret.put("flag", "200");
        ret.put("url", url);
        ret.put("ticket", ticket);
        ret.put("nonceStr", nonceStr);
        ret.put("timestamp", timestamp);
        ret.put("appId", DictionaryCache.systemConfig.get("h5.app.id").toString());
        return ret;
    }

    private String byteToHex(final byte[] hash) {
        Formatter formatter = new Formatter();
        for (byte b : hash) {
            formatter.format("%02x", b);
        }
        String result = formatter.toString();
        formatter.close();
        return result;
    }

    private String createNonceStr() {
        return UUID.randomUUID().toString();
    }

    private String createTimestamp() {
        return Long.toString(System.currentTimeMillis() / 1000);
    }
	
	public JSONObject doGet(String url) {
        logger.info(String.format("HttpGet -> %s", url));
        JSONObject jsonObject = null;
        HttpClient client = HttpClients.createDefault();
        HttpGet httpget = new HttpGet(url);
        HttpResponse response;
        try {
            response = client.execute(httpget);
            HttpEntity resEntity = response.getEntity();
            String jsonString = EntityUtils.toString(resEntity, "UTF-8");
            jsonObject = JSON.parseObject(jsonString);
        } catch (ClientProtocolException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return jsonObject;
    }
	
## 定时刷新	
4，因为有效期是两小时，所以还要设置个定时器去刷新token和ticket。设置为每小时刷新一次。

	@Scheduled(cron = "0 0 0/1 * * ?")
    public void updateAccessToken() {
        service.getAccessToken();
        service.getJSApiTicket();
    }
	
	
	
## 遇到的问题
	
- 在获取token时，接口返回无效的ip。 需要在微信公众号上添加白名单。

- 调用微信wx.config 提示无效的签名。 这是由于前端传到后台的url有参数，且不止一个。（微信规定url有参数的话也要带上），然后后台收到的url参数是不全的。
差不多是http://ip:port/wx/config?index=0&id=1 而在后端只能收到http://ip:port/wx/config?index=0 这一段，所以提示签名无效。

根据微信的提示
>2.invalid signature签名错误。建议按如下顺序检查：
1.确认签名算法正确，可用http://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=jsapisign 页面工具进行校验。
2.确认config中nonceStr（js中驼峰标准大写S）, timestamp与用以签名中的对应noncestr, timestamp一致。
3.确认url是页面完整的url(请在当前页面alert(location.href.split('#')[0])确认)，包括'http(s)://'部分，以及'？'后面的GET参数部分,但不包括'#'hash后面的部分。
4.确认 config 中的 appid 与用来获取 jsapi_ticket 的 appid 一致。
5.确保一定缓存access_token和jsapi_ticket。
6.确保你获取用来签名的url是动态获取的，动态页面可参见实例代码中php的实现方式。如果是html的静态页面在前端通过ajax将url传到后台签名，前端需要用js获取当前页面除去'#'hash部分的链接（可用location.href.split('#')[0]获取,而且需要encodeURIComponent），因为页面一旦分享，微信客户端会在你的链接末尾加入其它参数，如果不是动态获取当前链接，将导致分享后的页面签名失败。

前端没有对url进行转码，加上encodeURIComponent后就可以了。