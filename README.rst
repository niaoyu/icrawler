image\_crawler
==============

Introduction
------------

This python package is a mini framework of image crawlers. Python 2.x is
not supported for the moment.

Stucture
--------

It consists of 3 main components (Feeder, Parser and Downloader) and 2
FIFO queues (url\_queue and task\_queue). The workflow is shown in the
following figure.

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

Feeder, parser and downloader are all thread managers, which means they
start threads to finish corresponding tasks, so you can specify the
number of threads they use.

Quick start
-----------

Installation
~~~~~~~~~~~~

This package is under development and not available at PyPI currently,
so you cannot install it using the command
``pip install image_crawler``. Instead, you can install it by

::

    python setup.py develop

Then you should have all the dependency installed. If there is any
problem with it, you can install the dependency manually.

::

    pip install -r requirements.txt

This framework uses the HTTP library **requests** for sending requests
and the the parsing library **beautifulsoup4** for parsing HTML pages.

Use built-in crawlers
~~~~~~~~~~~~~~~~~~~~~

This framework contains 4 built-in crawlers.

-  Google
-  Bing
-  Baidu
-  Flickr

Here is an example of how to use the built-in crawlers. The search
engine crawlers have similar interfaces.

.. code:: python

    from image_crawler.examples import GoogleImageCrawler
    from image_crawler.examples import BingImageCrawler
    from image_crawler.examples import BaiduImageCrawler

    google_crawler = GoogleImageCrawler('your_image_dir')
    google_crawler.crawl(keyword='sunny', offset=0, max_num=1000,
                         date_min=None, date_max=None, feeder_thr_num=1,
                         parser_thr_num=1, downloader_thr_num=4)
    bing_crawler = BingImageCrawler('your_image_dir')
    bing_crawler.crawl(keyword='sunny', offset=0, max_num=1000,
                       feeder_thr_num=1, parser_thr_num=1, downloader_thr_num=4)
    baidu_crawler = BaiduImageCrawler('your_image_dir')
    baidu_crawler.crawl(keyword='sunny', offset=0, max_num=1000,
                        feeder_thr_num=1, parser_thr_num=1, downloader_thr_num=4)

**Note:** Only google image crawler supports date range parameters.

Flickr crawler is a little different.

.. code:: python

    from image_crawler.examples import FlickrImageCrawler
    from datetime import date

    flickr_crawler = FlickrImageCrawler('your_apikey', 'your_image_dir')
    flickr_crawler.crawl(max_num=1000, feeder_thr_num=1, parser_thr_num=1,
                         downloader_thr_num=1, tags='child,baby',
                         group_id='68012010@N00', min_upload_date=date(2015, 5, 1))

Supported optional searching auguments are

-  ``user_id`` -- The NSID of the user who's photo to search.
-  ``tags`` -- A comma-delimited list of tags.
-  ``tag_mode`` -- Either 'any' for an OR combination of tags, or 'all'
   for an AND combination.
-  ``text`` -- A free text search. Photos who's title, description or
   tags contain the text will be returned.
-  ``min_upload_date`` -- Minimum upload date. The date can be in the
   form of ``datetime.date`` object, a unix timestamp or a string.
-  ``max_upload_date`` -- Maximum upload date. Same form as
   ``min_upload_date``.
-  ``group_id`` -- The id of a group who's pool to search.
-  ``extras`` -- A comma-delimited list of extra information to fetch
   for each returned record. See
   https://www.flickr.com/services/api/flickr.photos.search.html for
   more details.
-  ``per_page`` -- Number of photos to return per page.

You can see the complete example in *test.py*, to run it

::

    python test.py [option]

``option`` can be ``google``, ``bing`` , ``baidu``, ``flickr`` or
``all``, using ``all`` by default.

Write your own crawler
~~~~~~~~~~~~~~~~~~~~~~

The simplest way is to override some methods of Feeder, Parser and
Downloader class.

1. **Feeder**

   The method you need to override is

   .. code:: python

       feeder.feed(**kwargs)

   If you want to offer the start urls at one time, for example from
   'http://example.com/page\_url/1' up to
   'http://example.com/page\_url/10'

   .. code:: python

       from image_crawler import Feeder

       class MyFeeder(Feeder):
           def feed(self):
               for i in range(10):
                   url = 'http://example.com/page_url/{}'.format(i + 1)
                   self.url_queue.put(url)

2. **Parser**

   The method you need to override is

   .. code:: python

       parser.parse(response, **kwargs)

   ``response`` is the page content of the url from ``url_queue``, what
   you need to do is to parse the page and extract image urls, and then
   put them into ``task_queue``. Beautiful Soup package is recommended
   for parsing html pages. Taking ``GoogleParser`` for example,

   .. code:: python

       class GoogleParser(Parser):

           def parse(self, response):
               soup = BeautifulSoup(response.content, 'lxml')
               image_divs = soup.find_all('div', class_='rg_di rg_el ivg-i')
               pattern = re.compile(r'imgurl=(.*?)\.jpg')
               for div in image_divs:
                   href_str = div.a['href']
                   match = pattern.search(href_str)
                   if match:
                       img_url = '{}.jpg'.format(match.group(1))
                       self.put_task_into_queue(dict(img_url=img_url))

3. **Downloader**

   If you just want to change the filename of downloaded images, you can
   override the method

   .. code:: python

       downloader.set_file_path(img_task)

   The default names of downloaded images are counting numbers, from
   000001 to 999999.

   If you want to process meta data, for example save some annotations
   of the images, you can override the method

   .. code:: python

       downloader.process_meta(img_task):

   Note that your parser need to put meta data as well as image urls
   into ``task_queue``.

   If you want to do more with the downloader, you can also override the
   method

   .. code:: python

       downloader.download(self, img_task, request_timeout, **kwargs)

   You can retrive tasks from ``task_queue`` and then do what you want
   to do.

4. **Crawler**

   You can either use the base class ``ImageCrawler`` or inherit from
   it. Two main apis are

   .. code:: python

       crawler.__init__(self, img_dir='images', feeder_cls=Feeder, parser_cls=Parser,
                        downloader_cls=Downloader, log_level=logging.INFO)

   and

   .. code:: python

       crawler.crawl(self, feeder_thread_num=1, parser_thread_num=1,
                     downloader_thread_num=1, feeder_kwargs={},
                     parser_kwargs={}, downloader_kwargs={})

   So you can use your crawler like this

   .. code:: python

       crawler = Crawler(feeder_cls=SimpleSEFeeder, parser_cls=MyParser)
       crawler.crawl(feeder_thr_num=1, parser_thr_num=1, downloader_thr_num=4,
                     feeder_kwargs=dict(
                         url_template='https://www.some_search_engine.com/search?keyword={}&start={}',
                         keyword='cat',
                         offset=0,
                         max_num=1000,
                         page_step=50
                     ),
                     downloader_kwargs=dict(max_num=1000))

   Or define a class to avoid using complex and ugly dictionaries as
   arguments.

   .. code:: python

       class MyCrawler(Crawler):

           def __init__(self, img_dir='images', log_level=logging.INFO):
               ImageCrawler.__init__(self, img_dir, feeder_cls=SimpleSEFeeder,
                                     parser_cls=MyParser, log_level=log_level)

           def crawl(self, keyword, max_num, feeder_thr_num=1, parser_thr_num=1,
                     downloader_thr_num=1, offset=0):
               feeder_kwargs = dict(
                   url_template='https://www.some_search_engine.com/search?keyword={}&start={}',
                   keyword=keyword,
                   offset=offset,
                   max_num=max_num,
                   page_step=50
               )
               downloader_kwargs = dict(max_num=max_num)
               super(MyCrawler, self).crawl(
                   feeder_thr_num, parser_thr_num, downloader_thr_num,
                   feeder_kwargs=feeder_kwargs,
                   downloader_kwargs=downloader_kwargs)

       crawler = MyCrawler()
       crawler.crawl(keyword='cat', offset=0, max_num=1000, feeder_thr_num=1,
                     parser_thr_num=1, downloader_thr_num=4)

API reference
-------------

To be continued.