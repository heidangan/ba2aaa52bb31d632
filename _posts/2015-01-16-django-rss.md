---
layout: post
title: Django实现RSS订阅功能
---

{{ page.title }}
================
<p class="date">{{ page.date | date_to_string }} - evi1m0</p>

1. Create APP

    python manage.py startapp feeds
    
2. Edit ```views.py```

		#!/usr/bin/env python
		# coding=utf8
		# author=evi1m0

        from django.shortcuts import render
        from django.core.urlresolvers import reverse
        from django.contrib.syndication.views import Feed

        from blog.apps.posts.models import topic

        class LatestEntries(Feed):
            title = "Linux.im"
            link = "/"
            description = "Updates on changes and additions to www.linux.im"

            def items(self):
                post = topic.objects.filter(status=1)[:20]
                return post

            def item_title(self, item):
                return item.title

            def item_description(self, item):
                return None

            def item_link(self, item):
                return '/posts/' + item.id + '/'


3. Edit ```url.py```

        from blog.apps.feed.views import *
        
        urlpatterns += patterns('blog.apps.feed.views',
    		url(r'^feeds/', LatestEntries(), name='feeds'),
		)
