---
layout: post
title:  "设计模式之工厂方法模式"
date:   2019-05-21 21:51:03
categories: 设计模式
tags: 设计模式 工厂方法模式
author: ddmcc
---

* content
{:toc}




## 什么是工厂方法模式

**定义一个创建对象的接口,但由子类决定要实例化的类是哪一个。工厂方法让类把实例化推迟到子类。**

![](http://ww1.sinaimg.cn/large/0060GLrDgy1g39aavm6fbj30hh0dyta5.jpg)

Creator是一个类(接口),有一个工厂方法 `factoryMethod()` ,由子类去实现。

ConcreteCreator实现了 `factoryMethod()` 以实际制造出一种或多种产品。如ConcreteProduct。所有的产品实现公同的接口Product,这样

生产出的产品就可以引用这个接口,而不是真正的实现类。


## 如何设计

假设有一个音乐播放器,可以根据用户的喜好去推送音乐。可以有中文和英文歌曲,下面又有分类:电子(Electronic),华语(Chinese),流行(Popular)等。


代码如下：

### 定义工厂接口

```java
package creator;

import music.Music;

/**
 *
 * 音乐生产接口
 *
 */
public interface MusicCreator {


    int POPULAR = 1;

    int ROCK = 2;

    int ELECTRONIC = 3;

    Music createMusic(int style);

}
```

### 具体的生产类


```java
package creator;

import music.Music;
import music.ChineseElectronicMusic;
import music.ChinesePopularMusic;
import music.ChineseRockMusic;

/**
 *
 * 中文歌曲工厂具体实现类
 *
 */
public class ChineseMusicCreator implements MusicCreator {


    @Override
    public Music createMusic(int style) {
        Music music;
        switch (style) {
            case POPULAR:
                music =  new ChinesePopularMusic();
                break;
            case ROCK:
                music = new ChineseRockMusic();
                break;
            case ELECTRONIC:
                music = new ChineseElectronicMusic();
                break;
            default:
                music = new ChinesePopularMusic();
        }
        return music;
    }
}



package creator;

import music.EnglishElectronicMusic;
import music.EnglishPopularMusic;
import music.EnglishRockMusic;
import music.Music;

/**
 *
 * 英文歌曲工厂具体实现类
 *
 */
public class EnglishMusicCreator implements MusicCreator {

    @Override
    public Music createMusic(int style) {
        Music music;
        switch (style) {
            case POPULAR:
                music = new EnglishPopularMusic();
                break;
            case ROCK:
                music = new EnglishRockMusic();
                break;
            case ELECTRONIC:
                music = new EnglishElectronicMusic();
                break;
            default:
                music = new EnglishPopularMusic();
        }
        return music;
    }
}

```

### 歌曲接口


```java
package music;

/**
 *
 * 歌曲接口
 *
 */
public interface Music {

    // 播放音乐
    void playMusic();

}
```


### 具体的歌曲

```java
package music;

public class ChineseElectronicMusic implements Music {

    @Override
    public void playMusic() {
        System.out.println("中文电子音乐~~~~");
    }
}



package music;

public class ChinesePopularMusic implements Music {

    @Override
    public void playMusic() {
        System.out.println("中文流行音乐!!!");
    }
}



package music;


public class ChineseRockMusic implements Music {

    @Override
    public void playMusic() {
        System.out.println("中文摇滚音乐~~~~~");
    }
}



package music;

public class EnglishElectronicMusic implements Music {

    @Override
    public void playMusic() {
        System.out.println("英文电子音乐~~~~");
    }
}




package music;

public class EnglishPopularMusic implements Music {

    @Override
    public void playMusic() {
        System.out.println("英文流行音乐!!!");
    }
}



package music;


public class EnglishRockMusic implements Music {

    @Override
    public void playMusic() {
        System.out.println("英文摇滚音乐~~~~~");
    }
}
```


### 客户端代码
```java

import creator.ChineseMusicCreator;
import creator.EnglishMusicCreator;
import creator.MusicCreator;
import music.Music;

public class Main {

    public static void main(String[] args) {

        MusicCreator englishMusicCreator = new EnglishMusicCreator();
        Music englishMusic = englishMusicCreator.createMusic(MusicCreator.POPULAR);
        englishMusic.playMusic();

        MusicCreator chineseMusicCreator = new ChineseMusicCreator();
        Music chineseMusic = chineseMusicCreator.createMusic(MusicCreator.POPULAR);
        chineseMusic.playMusic();
    }
}
```


![](http://ww1.sinaimg.cn/large/0060GLrDgy1g39cp387qej31g10ohgoy.jpg)



在简单工厂模式中，我们会把所有的产品在一个类中去判断，然后生产出要的产品。这就导致如果有新的种类产品的时候

我们就要去修改原来的工厂方法，这就违反了开-闭原则。工厂方法模式是对简单工厂模式进一步的解耦，因为在工厂方法模式中是一个或多个子类对应一个工厂类，

而这些工厂类都实现于一个抽象接口。这相当于是把原本会因为业务代码而庞大的简单工厂类，拆分成了一个个的工厂类，这样代码就不会都耦合在同一个类里了，就像把一个大蛋糕切成了多个小蛋糕。




## 设计原则

- 依赖倒置原则,即不能让高级组件依赖低级组件,而且不管高级组件或低级组件,都应该依赖抽象。

在上面的例子中MusicCreator是高级组件,具体的音乐实现类是低级组件,它们都依赖了Music这个抽象。所以遵循了依赖倒置原则。