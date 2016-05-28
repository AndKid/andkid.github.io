---
title: Hexo主题之Next
date: 2016-02-16 22:02:58
tags:
- hexo
- blog
---
Next是最近比较流行的Hexo主题

[官方帮助文档](http://theme-next.iissnan.com/getting-started.html)

帮助文档写得很棒，还可以从一些内容中感觉到作者功力相当深厚啊，膜拜ing~
[作者的Next模板](http://notes.iissnan.com/)

我主要讲一下当看到其他人也在用Next主题，发现他的一些元素我们没有的时候怎么快速找到相应的配置

比如在菜单中的这个效果
![github](/uploads/github.png)

1. 右击选“审查元素”
![html](/uploads/html.png)
2. 在Next主题文件夹下找我们看到的class元素'links-of-author'，找到如下代码
```html

<div class="links-of-author motion-element">
  {% if theme.social %}
    {% for name, link in theme.social %}
      <span class="links-of-author-item">
        <a href="{{ link }}" target="_blank" title="{{ name }}">
          {% if theme.social_icons.enable %}
            <i class="fa fa-{{ theme.social_icons[name] || 'globe' }}"></i>
          {% endif %}
          {{ name }}
        </a>
      </span>
    {% endfor %}
  {% endif %}
</div>

```

3. 可以看到需要social.enable和social_icons.enable都要为true，social标签下的key对应social_icons的value，value是超链接
4. 从theme的配置文件`/themes/_config.yml`中果然看到配置项和注解，如下配置就搞定啦
```java

# Social Links
# Key is the link label showing to end users.
# Value is the target link (E.g. GitHub: https://github.com/iissnan)
social:
  #LinkLabel: Link
  Github: https://github.com/AndKid


# Social Links Icons
# Icon Mapping:
#   Map a menu item to a specific FontAwesome icon name.
#   Key is the name of the item and value is the name of FontAwsome icon. Key is case-senstive.
#   When an globe mask icon presenting up means that the item has no mapping icon.
social_icons:
  enable: true
  # Icon Mappings.
  # KeyMapsToSocalItemKey: NameOfTheIconFromFontAwesome
  Github: github
  Twitter: twitter
  Weibo: weibo

```
