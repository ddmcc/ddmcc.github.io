---
layout: post
title:  "爬取网易云的用户,评论,歌曲等等"
date:   2018-04-24 18:24:21
categories: Crawl
tags: Crawl 爬虫
author: ddmcc
---

* content
{:toc}




## 动机

觉得可以爬取很多的用户，评论，歌曲，然后对这些数据进行分析，会
很有意思，就开始了。

## 过程
- 获取API接口
- 分析API接口
- 调用接口获取数据
- 解析数据，持久化


### 获取API接口
要获取数据，首先要知道它的请求接口。首先打开一个歌单

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fqny0qy5luj30n20npn1i.jpg)




打开NetWork查看连接，发现有个获取列表的请求，`playlist？id=924680166`，还是个 **GET** 请求，可以在url看见参数

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fqny3wgj85j30m301b743.jpg)

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fqny6hnen3j30r20bawfx.jpg)

这样的话，寻找到其他的接口，按照它的参数格式去调用，不就ok了？走你

随便点进一首歌，寻找获取评论和播放的接口。点进去之后的地址栏

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fqnypd68zxj309700x3yb.jpg)

id应该就是歌曲的id了，`id=553543014` ，查看请求

发现了一个应该是获取评论的 `Ajax` 请求，并且是 **POST**，说明参数是以附件的形式传送过去了

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fqnyqnemv0j309201aglf.jpg)

查看它的请求参数，发现了 **params** 和 **encSecKey** 两个参数，后面跟着一串的经过加密之后的参数！！
大厂就是大厂啊....那我咋知道它是怎么加密的啊？ 查之！

![](http://ww1.sinaimg.cn/large/0060GLrDgy1fqnys8h6twj30hf0f4401.jpg)


### 分析API接口

在 [某乎](https://www.zhihu.com/question/36081767) 发现有好多大牛都对接口的请求参数进行了分析，其中
有一个更是直接用Java内置的ScriptEngine调用JS引擎来解析，[什么是ScriptEngine？](https://blog.csdn.net/u012660667/article/details/49821811)，并且把两万多行
的core.js代码缩减到只有一千多行，这样就简单了，直接用Java代码就可以对参数进行加密，然后进行接口的调用。

通过这位大牛对core.js代码的修改，对外提供了一个 **`myFunc`** 这么一个入口函数，只需要调用这个函数，把参数传入，就可以得到
经过加密的打包参数了。

```js
public class JSSecret {
    private static Invocable inv;
    public static final String encText = "encText";
    public static final String encSecKey = "encSecKey";

    /**
     * 从本地加载修改后的 core.js 文件到 ScriptEngine
     */
    static {
        try {
            Path path = Paths.get("core.js");
            byte[] bytes = Files.readAllBytes(path);
            String js = new String(bytes);
            ScriptEngineManager factory = new ScriptEngineManager();
            ScriptEngine engine = factory.getEngineByName("JavaScript");
            engine.eval(js);
            inv = (Invocable) engine;
            System.out.println("Init completed");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static ScriptObjectMirror get_params(String paras) throws Exception {
        ScriptObjectMirror so = (ScriptObjectMirror) inv.invokeFunction("myFunc", paras);
        return so;
    }

    public static HashMap<String, String> getDatas(String paras) {
        try {
            ScriptObjectMirror so = (ScriptObjectMirror) inv.invokeFunction("myFunc", paras);
            HashMap<String, String> datas = new HashMap<>();
            datas.put("params", so.get(JSSecret.encText).toString());
            datas.put("encSecKey", so.get(JSSecret.encSecKey).toString());
            return datas;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

从本地读取 `core.js` 文件，传入parameters，获取经过加密的打包datas。

### 调用接口获取数据

获取加密的参数之后就可以对接口进行调用，来获取我们想要的数据了。

APi对应接口的参数类

```js
package netease;

public class Api {

	private final static String BaseURL = "http://music.163.com/";

	/**
	 * 获取用户歌单
	 *
	 * @param uid
	 * @return
	 */
	public static UrlParamPair getPlaylistOfUser(String uid) {
        UrlParamPair upp = new UrlParamPair();
        upp.setUrl(BaseURL + "weapi/user/playlist?csrf_token=");
        upp.addPara("uid", uid);
        upp.addPara("limit", 10);
        upp.addPara("offset",0);
        upp.addPara("csrf_token", "nothing");
        return upp;
    }
	 

	/**
	 * 获取歌单详情
	 *
	 * @param playlist_id
	 * @return
	 */
	 public static UrlParamPair getDetailOfPlaylist(String playlist_id) {
	        UrlParamPair upp = new UrlParamPair();
	        upp.setUrl(BaseURL + "weapi/v3/playlist/detail?csrf_token=");
	        upp.addPara("id", playlist_id);
	        upp.addPara("offset", 0);
	        upp.addPara("total", "True");
	        upp.addPara("limit", 1000);
	        upp.addPara("n", 1000);
	        upp.addPara("csrf_token", "nothing");
	        return upp;
	    }

	// todo:analyse more api
	/**
	 * 搜索歌曲
	 *
	 * @param s
	 *            ;
	 * @return
	 */
	public static UrlParamPair SearchMusicList(String s, String type) {
		UrlParamPair upp = new UrlParamPair();
		upp.addPara("s", s);
		upp.addPara("type", type);
		upp.addPara("offset", 0);
		upp.addPara("total", "True");
		upp.addPara("limit", 100);
		upp.addPara("n", 1000);
		upp.addPara("csrf_token", "nothing");
		return upp;
	}
	
	public static UrlParamPair SearchUrlList(String s) {
		UrlParamPair upp = new UrlParamPair();
		upp.addPara("ids", "["+s+"]");
		upp.addPara("br", 320000);
		upp.addPara("csrf_token", "nothing");
		return upp;
	}

}
```

- 获取一首歌的评论

```js
UrlParamPair upp = Api.getDetailOfPlaylist(music_id);
String req_str = upp.getParas().toJSONString();
response = Jsoup.connect("http://music.163.com/weapi/v1/resource/comments/R_SO_4_"+ music_id + "?csrf_token=")
		.userAgent("Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:57.0) Gecko/20100101 Firefox/57.0")
		.header("Accept", "*/*")
		.header("Cache-Control", "no-cache")
		.header("Connection", "keep-alive")
		.header("Host", "music.163.com")
		.header("Accept-Language", "zh-CN,en-US;q=0.7,en;q=0.3")
		.header("DNT", "1")
		.header("Pragma", "no-cache")
		.header("Content-Type", "application/x-www-form-urlencoded")
		.data(JSSecret.getDatas(req_str))
		.method(Connection.Method.POST).ignoreContentType(true)
		.timeout(15000).execute();
```


- 获取用户的歌单

```js
UrlParamPair upp = Api.getPlaylistOfUser(String.valueOf(cuser.getUserId()));
String req_str = upp.getParas().toJSONString();
response = Jsoup.connect("http://music.163.com/weapi/user/playlist?csrf_token=")
		.userAgent("Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:57.0) Gecko/20100101 Firefox/57.0")
		.header("Accept", "*/*")
		.header("Cache-Control", "no-cache")
		.header("Connection", "keep-alive")
		.header("Host", "music.163.com")
		.header("Accept-Language", "zh-CN,en-US;q=0.7,en;q=0.3")
		.header("DNT", "1")
		.header("Pragma", "no-cache")
		.header("Content-Type", "application/x-www-form-urlencoded")
		.data(JSSecret.getDatas(req_str))
		.method(Connection.Method.POST).ignoreContentType(true)
		.timeout(15000).execute();
String list = response.body();
```


- 获取音乐的播放链接

```js
UrlParamPair upp = Api.SearchUrlList(String.valueOf(music_id));
String req_str = upp.getParas().toJSONString();
response = Jsoup.connect(
		.userAgent("Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:57.0) Gecko/20100101 Firefox/57.0")
		.header("Accept", "*/*").header("Cache-Control", "no-cache")
		.header("Connection", "keep-alive")
		.header("Host", "music.163.com")
		.header("Accept-Language", "zh-CN,en-US;q=0.7,en;q=0.3")
		.header("DNT", "1").header("Pragma", "no-cache")
		.header("Content-Type", "application/x-www-form-urlencoded")
		.data(JSSecret.getDatas(req_str))
		.method(Connection.Method.POST).ignoreContentType(true)
		.timeout(10000).execute();
String list = response.body();
```


等等还有一些其他的接口，加密的方式都是一样的。

### 解析数据，持久化

接口返回的数据 **response.body();** 是一个Json字符串，所以对字符串进行解析即可得到数据。

返回的评论数据

```
{"isMusician":false,
 "userId":-1,
 "topComments":[],
 "moreHot":true,
 "hotComments":[
热门
{"user":{"locationInfo":null,"userId":384740724,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/3gOhLA71cNcXZYi-zxmZ4A==/109951163252233050.jpg","experts":null,"nickname":"用户已被查封丶","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":5460,"time":1492040700361,"liked":false,"commentId":355257109,"content":"数学老师写下一句话，“我爱你”，要求改成逆否命题。 我们都说，“你不爱我” “不是的”老师说。 他先把它变成了这种形式，“如果有一个人是我，那么这个人爱你。” 老师接着开始改逆否命题了。 最后一刻，他停笔的瞬间，教室很安静。 “如果一个人不爱你，那么，这个人，不是我。"},
{"user":{"locationInfo":null,"userId":5776792,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/RXaSX8Pr6XG-AH4XvwAiaA==/18569651883453829.jpg","experts":null,"nickname":"茶走啡走","userType":0,"vipType":11},"beReplied":[],"pendantData":null,"likedCount":3859,"time":1412343342625,"liked":false,"commentId":4662836,"content":"旁边的人跟朋友视频玩飞行棋，这首歌缓慢的节奏搭着他们的喧哗声，突然莫名其妙好难过，for no reason."},
{"user":{"locationInfo":null,"userId":18031377,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/uzdrPFaDBmdxuXixkmpNZg==/3294136841961887.jpg","experts":null,"nickname":"CancerCathy","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":2673,"time":1412160724409,"liked":false,"commentId":4624564,"content":"在图书馆突然听到这首歌，可以译成 且听风吟 么？安静，触动人心，除了流泪没有别的方式表达这支曲带来的感动。"},
{"user":{"locationInfo":null,"userId":9380975,"expertTags":["轻音乐","流行"],"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/NMVictjyR-CW3Qwumegoyw==/109951162801439858.jpg","experts":null,"nickname":"Printing","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":2185,"time":1467345090514,"liked":false,"commentId":176888581,"content":"，我本可以忍受孤独了，至死的那种，但遇到一个人，就不行了。[哀伤]"},
{"user":{"locationInfo":null,"userId":45913247,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/v8kJO2z5LEu6wLpoiECgxw==/7838418395141586.jpg","experts":null,"nickname":"鸿雁不归意缭乱","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":2183,"time":1434881345541,"liked":false,"commentId":23023466,"content":"“由于怀着爱的希望，孤独才是可以忍受的，甚至是甜蜜的。”_周国平。听风吟，风吹暖，心是浩渺，可以在方寸之地无限制地畅想。"},
{"user":{"locationInfo":null,"userId":59195546,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/PSFBiMxE6Og8QYgrreTaow==/109951163245284077.jpg","experts":null,"nickname":"月刊少女胡小喵","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":1690,"time":1495441831537,"liked":false,"commentId":389664832,"content":"我所有的自负皆来自我的自卑，所有的英雄气概都来自于我的软弱。嘴里振振有词是因为心里满是怀疑，深情是因为痛恨自己无情。这世界没有一件事情是虚空而生的，站在光里，背后就会有阴影，这深夜里一片寂静，是因为你还没有听见声音。"},
{"user":{"locationInfo":null,"userId":37621110,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/ELGZhDJG6OUakaG7zWOIJA==/3231464674454522.jpg","experts":null,"nickname":"姐夫尹","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":1186,"time":1415616481514,"liked":false,"commentId":5647901,"content":"电影叫云图"},
{"user":{"locationInfo":null,"userId":32399294,"expertTags":["Bossa Nova","爵士","古典"],"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/GEz-aqYB0OSs2VrFy9L0HA==/109951163026055373.jpg","experts":null,"nickname":"江河故人","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":834,"time":1430975381469,"liked":false,"commentId":17475824,"content":"极静极寂"},
{"user":{"locationInfo":null,"userId":264931387,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/VnZiScyynLG7atLIZ2YPkw==/18686200114669622.jpg","experts":null,"nickname":"帐号已注销","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":770,"time":1490680489127,"liked":false,"commentId":341041910,"content":"看這首歌的評論就像看一個人的傷口一樣，自己都疼。"},
{"user":{"locationInfo":null,"userId":239907,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/cMldetUmk9S-Ev3N-W_RnA==/109951162862405439.jpg","experts":null,"nickname":"云汐花开","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":700,"time":1434811237589,"liked":false,"commentId":22933319,"content":"风在夜空下歌唱，星星降落在湖面，鱼儿都睡了，你的沉默如大海。晚安[爱心]"},
{"user":{"locationInfo":null,"userId":344851685,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/2LFUt8tbi4_hq_45mqLjlQ==/109951163097694688.jpg","experts":null,"nickname":"铭空间","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":687,"time":1507645813912,"liked":false,"commentId":582013988,"content":"所有人都近乎发疯，运转机器的齿轮费力转动的快要静止，没有停顿的空间。他们没有乐趣，阳光照在背面，风吹来只有冰凉，食物的色味黯淡，他们无话可说，等待钟表指针的划动，左手插在口袋，右手为它撑伞，它(世界)俗不可耐。 ​​​"},
{"user":{"locationInfo":null,"userId":55044793,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/6Ews95Sp0YCbHR7pwtDOHg==/18833534674307181.jpg","experts":null,"nickname":"Booyan","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":547,"time":1427905470801,"liked":false,"commentId":14068188,"content":"既定的命运，如何去挣扎"},
{"user":{"locationInfo":null,"userId":34278757,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/sgc4gV1UiwXuRgf6Xs1iKQ==/18909400975016242.jpg","experts":null,"nickname":"六酱酱酱酱","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":465,"time":1424002099840,"liked":false,"commentId":10848330,"content":"晴空寂夜，星辰璀璨"},
{"user":{"locationInfo":null,"userId":33558127,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/1nBzLadqVHMltEEHU3KwmQ==/1394180746807747.jpg","experts":null,"nickname":"守候ll","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":373,"time":1422534418361,"liked":false,"commentId":9677101,"content":"封面的OVI 想起了诺基亚[流感]"},
{"user":{"locationInfo":null,"userId":59130904,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/a7XVjh_tIzQCv7yQKkrg-Q==/109951162939131254.jpg","experts":null,"nickname":"鈴木家的冬天和夏天","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":253,"time":1434815009718,"liked":false,"commentId":22935784,"content":"一阵一阵的海风总带着咸咸的味道，海浪没有规律的拍打着礁石，我一回过头，便看到对我微笑的你……"}],

"code":200,
"comments":[
普通评论，第一页的数据
{"user":{"locationInfo":null,"userId":510418565,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/tEdQo8xtxqQ60kreNgMUtQ==/19135900370044951.jpg","experts":null,"nickname":"江南青山","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":0,"time":1524554545101,"liked":false,"commentId":1096091063,"content":"你的毡帽看起来真的太傻了。","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":579791324,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/3kgEEUzuF7-EZDhewJRffw==/109951163183716247.jpg","experts":null,"nickname":"摇滚影魔","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":0,"time":1524545934551,"liked":false,"commentId":1095982234,"content":"丧失 越坠越深","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":326392811,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/l_FpB7_isCGBWvhqcJRi0w==/109951163035801434.jpg","experts":null,"nickname":"Loliiitaa","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":0,"time":1524542061606,"liked":false,"commentId":1095922450,"content":"月光柔和的洒在深夜的海上 我独自一人 走在海边 听着 海浪卷着风的声音","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":326392811,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/l_FpB7_isCGBWvhqcJRi0w==/109951163035801434.jpg","experts":null,"nickname":"Loliiitaa","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":0,"time":1524542001889,"liked":false,"commentId":1095947719,"content":"感觉 像自己深夜走在海边","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":345764368,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/Gm_GxmvppzPkUjCqIqxleQ==/109951163255574336.jpg","experts":null,"nickname":"Hcksbfoo","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":0,"time":1524525440787,"liked":false,"commentId":1095789763,"content":"风之歌","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":617752356,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/h-V5R8ZqYtbzCY6U_c6QSw==/109951163235061890.jpg","experts":null,"nickname":"温柔十里","userType":0,"vipType":0},"beReplied":[{"user":{"locationInfo":null,"userId":1386098665,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/j_pZyoeVeDFmuSWsolKu0w==/109951163170393931.jpg","experts":null,"nickname":"此去经年_以风为伴","userType":0,"vipType":0},"content":"兄弟下次喝酒叫上我！","status":0}],"pendantData":null,"likedCount":0,"time":1524482448296,"liked":false,"commentId":1095217048,"content":"可以的","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":98606695,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/UziAMRtA90NACSpUMLNWVQ==/1374389538563091.jpg","experts":null,"nickname":"繁华-流尽","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":1,"time":1524413150159,"liked":false,"commentId":1094556769,"content":"有些歌只适合耳机听","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":115223202,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/2bHVnmThHaULSu4GXLJOiw==/109951163262581818.jpg","experts":null,"nickname":"Sleene_","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":1,"time":1524403728123,"liked":false,"commentId":1094288222,"content":"希望我们都能战胜恐惧。战胜自己。过好这一生。","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":1386098665,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/j_pZyoeVeDFmuSWsolKu0w==/109951163170393931.jpg","experts":null,"nickname":"此去经年_以风为伴","userType":0,"vipType":0},"beReplied":[{"user":{"locationInfo":null,"userId":617752356,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/h-V5R8ZqYtbzCY6U_c6QSw==/109951163235061890.jpg","experts":null,"nickname":"温柔十里","userType":0,"vipType":0},"content":"一个人听歌，一个人喝酒，一个没朋友的人","status":0}],"pendantData":null,"likedCount":0,"time":1524398358977,"liked":false,"commentId":1094196973,"content":"兄弟下次喝酒叫上我！","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":274515919,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/Af9KBESoIvc1xHBAdhAm4A==/19220562765600543.jpg","experts":null,"nickname":"Maestro-Music","userType":0,"vipType":10},"beReplied":[{"user":{"locationInfo":null,"userId":55044793,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/6Ews95Sp0YCbHR7pwtDOHg==/18833534674307181.jpg","experts":null,"nickname":"Booyan","userType":0,"vipType":0},"content":"既定的命运，如何去挣扎","status":0}],"pendantData":null,"likedCount":0,"time":1524386537072,"liked":false,"commentId":1093983220,"content":"过去已定，未来可期","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":73990126,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/igue_V12v1Snm6Fjw1NBJA==/109951163265396234.jpg","experts":null,"nickname":"不叫美人","userType":0,"vipType":11},"beReplied":[],"pendantData":null,"likedCount":0,"time":1524386417552,"liked":false,"commentId":1093975287,"content":"我是真的太差了吗…","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":479453291,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/Kqw4ZfTs1KmPHCM7AKEOAQ==/109951163147366555.jpg","experts":null,"nickname":"茗尚温","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":0,"time":1524383671520,"liked":false,"commentId":1093955801,"content":"Do not leave me.","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":617752356,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/h-V5R8ZqYtbzCY6U_c6QSw==/109951163235061890.jpg","experts":null,"nickname":"温柔十里","userType":0,"vipType":0},"beReplied":[{"user":{"locationInfo":null,"userId":319860700,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/N4YQEHJfj-EcCtHipEtPvQ==/18710389369923911.jpg","experts":null,"nickname":"赛宾斯基","userType":0,"vipType":0},"content":"一个人唱歌，一个人去医院，一个人搬家","status":0}],"pendantData":null,"likedCount":1,"time":1524382342088,"liked":false,"commentId":1093913356,"content":"一个人听歌，一个人喝酒，一个没朋友的人","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":607498344,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/RitAIt1ATd8V9QkSyd60XQ==/18626826488197471.jpg","experts":null,"nickname":"哎忘了TA吧","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":0,"time":1524374076845,"liked":false,"commentId":1093761237,"content":"我一直知道我没有你","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":117486376,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/x6gyrTDMTKQjwJbPYyC1hw==/18810444930168248.jpg","experts":null,"nickname":"M-limi","userType":0,"vipType":0},"beReplied":[{"user":{"locationInfo":null,"userId":5776792,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/RXaSX8Pr6XG-AH4XvwAiaA==/18569651883453829.jpg","experts":null,"nickname":"茶走啡走","userType":0,"vipType":11},"content":"旁边的人跟朋友视频玩飞行棋，这首歌缓慢的节奏搭着他们的喧哗声，突然莫名其妙好难过，for no reason.","status":0}],"pendantData":null,"likedCount":1,"time":1524367091289,"liked":false,"commentId":1093641809,"content":"这热闹是他们的，而我什么都没有","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":397071462,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/Md2mgXaeYm7ddMR3Bc_kow==/109951163256049218.jpg","experts":null,"nickname":"7_AM","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":0,"time":1524323630464,"liked":false,"commentId":1093145762,"content":"来过","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":395211866,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/5kFG5jLeKpp4e3WwuTSd_Q==/19047939439980971.jpg","experts":null,"nickname":"杏仁奶糖贩卖机","userType":0,"vipType":0},"beReplied":[{"user":{"locationInfo":null,"userId":384740724,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/3gOhLA71cNcXZYi-zxmZ4A==/109951163252233050.jpg","experts":null,"nickname":"用户已被查封丶","userType":0,"vipType":0},"content":"数学老师写下一句话，“我爱你”，要求改成逆否命题。 我们都说，“你不爱我” “不是的”老师说。 他先把它变成了这种形式，“如果有一个人是我，那么这个人爱你。” 老师接着开始改逆否命题了。 最后一刻，他停笔的瞬间，教室很安静。 “如果一个人不爱你，那么，这个人，不是我。","status":0}],"pendantData":null,"likedCount":0,"time":1524320125653,"liked":false,"commentId":1093032143,"content":"补充一句原命题和逆否命题的真假性相同","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":447931805,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/1OFPepVr8LXWw6mIz12V-A==/18801648837040171.jpg","experts":null,"nickname":"你的阿瑾","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":1,"time":1524299928911,"liked":false,"commentId":1092616264,"content":"我还是一个人","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":543625535,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/6ASkHhwgRfD7CY8cGqr8LQ==/19194174486243100.jpg","experts":null,"nickname":"Higanbana谶","userType":0,"vipType":11},"beReplied":[],"pendantData":null,"likedCount":2,"time":1524275874497,"liked":false,"commentId":1092165317,"content":"我把这首歌推荐给他。\n他问我：你害怕孤独吗？\n我说：说不害怕是假的。💔","isRemoveHotComment":false},
{"user":{"locationInfo":null,"userId":410376991,"expertTags":null,"authStatus":0,"remarkName":null,"avatarUrl":"http://p1.music.126.net/AP2SAMpRexuV8ycHAh4cVQ==/109951163248461907.jpg","experts":null,"nickname":"一絲一毫的雲","userType":0,"vipType":0},"beReplied":[],"pendantData":null,"likedCount":0,"time":1524273260420,"liked":false,"commentId":1092143593,"content":"聽著好難受","isRemoveHotComment":false}],"total":4367,"more":true}
```

对数据进行解析就可以了

```js
JSONObject j = new JSONObject(response.body());
JSONArray a = (JSONArray) j.get("comments");
Iterator<Object> hot = a.iterator();
while (hot.hasNext()) {
	CloudComment hotMent = new CloudComment();
	JSONObject obj = (JSONObject) hot.next();
	hotMent.setCommentId(obj.getLong("commentId"));
	hotMent.setTime(obj.getLong("time"));
	hotMent.setLikedCount(obj.getInt("likedCount"));
	hotMent.setContent(obj.getString("content"));
	JSONObject u = new JSONObject(obj.get("user").toString());			
	hotMent.setUserId(u.getLong("userId"));			
	if(!Insert.getInstance().get(obj.getLong("commentId"))){
		Insert.getInstance().insert(CloudComment.class,hotMent);
	}			
	CloudUser user = new CloudUser();		
	user.setUserId(u.getLong("userId"));
	user.setAvatarUrl(u.getString("avatarUrl"));
	user.setNickname(u.getString("nickname"));
	user.setVipType(u.getInt("vipType"));
	user.setUserType(u.getInt("userType"));
	if(!Insert.getInstance().getListByCondit(user.getUserId())){
		CloudUser usr = update(user);
		Insert.getInstance().insert(CloudUser.class,usr);
		System.out.println("本次成功已爬取数据 "+(count++)+" 条");
	}
}
```