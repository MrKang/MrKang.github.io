---
layout: page
title: About
---

{% comment %}
  This inserts the "about" photo and text from `_config.yml`.
  You can edit it there (jekyll needs restart!) or remove it and provide your own photo/text.
  Don't forget to add the `me` class to the photo, like this: `![alt](src){:.me}`.
{% endcomment %}

{% if site.author.photo %}
  ![{{ site.author.name }}]({{ site.author.photo }}){:.me}
{% endif %}

{{ site.author.about }}
>哦对了，我现居北京，这里仅以记录对iOS学习的一些笔记，以及对各方面的吐槽，哈哈。   
>虽然这里留下了我的Twitter，但其实我用的并不多，如果有要联系我的话可以发我Gmail：   
>kangzlong@gmail.com   
