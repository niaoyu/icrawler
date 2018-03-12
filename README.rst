###########################################
第一次使用爬虫工具。。自己使用的时候，遇到下载下来的图片不正常的问题，修改相关文件后成功。
同时参考了知乎，对代码做了以下简单修改。
python中如何使用requests模块下载文件并获取进度提示？ - 微微寒的回答 - 知乎
https://www.zhihu.com/question/41132103/answer/93438156

1. 解决问题
   filesystem.py
   mode = 'w' if isinstance(data, str) else 'wb'
   =》
   mode = 'wb'

2. 适当的修改--可以中间做一些事情。。。

 download.py
    self.storage.write(filename, response.content)
    =>
    self.storage.write(filename, response)
    
    response = self.session.get(file_url, timeout=timeout)
    =>
    response = self.session.get(file_url, timeout=timeout,stream=True)
    
 filesystem.py
    mode = 'w' if isinstance(data, str) else 'wb'
    with open(filepath, mode) as fout:
         fout.write(data)
      
    ==>
    
     counter = 0
     with open(filepath,"wb") as f:
         for chunk in data.iter_content(chunk_size=1024): 
             if chunk: 
                 f.write(chunk) 
                 f.flush()
                 counter += 1

icrawler
========

.. image:: https://img.shields.io/pypi/v/icrawler.svg
   :target: https://pypi.python.org/pypi/icrawler
   :alt: PyPI Version

.. image:: https://anaconda.org/hellock/icrawler/badges/version.svg
   :target: https://anaconda.org/hellock/icrawler
   :alt: Anaconda Version

.. image:: https://img.shields.io/pypi/pyversions/icrawler.svg
   :alt: Python Version

.. image:: 	https://img.shields.io/github/license/hellock/icrawler.svg
   :alt: License

Introduction
------------

Documentation: http://icrawler.readthedocs.io/

Try it with ``pip install icrawler`` or ``conda install -c hellock icrawler``.

This package is a mini framework of web crawlers. With modularization design,
it can be extended conveniently. It supports media data like images and videos
very well, and can also be applied to texts and other type of files.
Scrapy is heavy and powerful, while icrawler is tiny and flexible.

With this package, you can write a multiple thread crawler easily by
focusing on the contents you want to crawl, avoiding some troublesome problems like
exception handling, thread scheduling and communication.

It also provides built-in crawlers for popular image sites like flickr and
search engines such as Google, Bing and Baidu. If you want to add your own
crawlers into built-in, welcome for pull requests.

Requirements
------------

Python 2.7+ or 3.4+ (recommended).

Samples
-------

Using builtin crawlers is very simple.

.. code:: python

    from icrawler.builtin import GoogleImageCrawler

    google_crawler = GoogleImageCrawler(parser_threads=2, downloader_threads=4,
                                        storage={'root_dir': 'your_image_dir'})
    google_crawler.crawl(keyword='sunny', max_num=1000,
                         date_min=None, date_max=None,
                         min_size=(200,200), max_size=None)

Writing your own crawlers with this framework is also convenient, see the
`tutorials <http://icrawler.readthedocs.io/en/latest/usage.html#write-your-own>`_.

Structure
---------

A crawler consists of 3 main components (Feeder, Parser and Downloader),
they are connected with each other with FIFO queues. The workflow is shown in
the following figure.

.. figure:: http://7xopqn.com1.z0.glb.clouddn.com/workflow.png
   :alt: 

-  ``url_queue`` stores the url of pages which may contain images
-  ``task_queue`` stores the image url as well as any meta data you
   like, each element in the queue is a dictionary and must contain the
   field ``img_url``
-  Feeder puts page urls to ``url_queue``
-  Parser requests and parses the page, then extracts the image urls and
   puts them into ``task_queue``
-  Downloader gets tasks from ``task_queue`` and requests the images,
   then saves them in the given path.

Feeder, parser and downloader are all thread pools, so you can specify the
number of threads they use.
