FunnelWeb - Content conversion made easy
****************************************

Easily convert content from existing sites into Plone.

- Code repository: http://github.com/collective/funnelweb
- Questions and comments to http://github.com/collective/funnelweb/issues
- Report bugs at http://github.com/collective/funnelweb/issues

.. contents::

Introduction
------------

Funnelweb is very easy to get started with via a few settings in either buildout
or the commandline. The predefined options have been
well thought out and have proved useful on many site conversions over the years.
Funnelweb is also very powerful since if the included transformations aren't
enough for your needs, funnelweb is built on a modular
transformation architecture called transmogrifier. This allows you to insert
transformation steps from yourself or others to suit any site conversion need.


The simplest way to install is via a buildout recipe (see zc.buildout) ::

  [buildout]
  parts += funnelweb

  [funnelweb]
  recipe = funnelweb
  crawler-url=http://www.whitehouse.gov
  ploneupload-target=http://admin:admin@localhost:8080/Plone

  $> buildout init
  $> bin/buildout

The above example will create a script to import content from the whitehouse.gov and upload
it to a local Plone site via XML-RPC. This can be run by ::

 $> bin/funnelweb

The script will:

1. Crawl
2. Cache locally so subsequent crawls are quicker
3. Filter
4. Remove boilerplate (automatically or via rules)
5. Restructure
6. Determines title, hidden from navigation, etc.
7. Uploads to Plone, or saves HTML to local directory


History
-------

- 2008 Built to import large corporate intranet
- 2009 released pretaweb.funnelweb (deprecated). Built into Plone UI > Actions > Import
- 2010 Split blueprints into transmogrify.* release on pypi
- 2010 collective.developermanual sphinx to Plone uses funnelweb blueprints
- 2010 funnelweb Recipe + Script released



Options
-------

Funnelweb uses a transmogrifier pipeline to perform the needed transformations. All
commandline and recipe options refer to options in the pipeline. All the options below
can either be given as options to the buildout recipe or can be overridden via the commandline.
For instance ::

  [funnelweb]
  recipe = funnelweb
  crawler-url=http://www.whitehouse.gov

::

 $> bin/funnelweb 

and ::

 $> bin/funnelweb --crawler:url=http://www.whitehouse.gov

will do the same thing.

The options pertain to different stages in the pipeline. In the above, we are
setting the `url` option for the `crawler` transformation. This transformation
also takes the `checkext`, `verbose`, `maxsize` and `nonames` options.

.. NOTE:: See the transmogrifier packages for documentation of the options accepted by
          the various transformations.

	  Transform options in the ``funnelweb`` buildout part refer to the
          parts defined in ``pipeline.cfg`` (Dylan, correct me if I'm wrong).

Crawling - HTML to import
~~~~~~~~~~~~~~~~~~~~~~~~~

Funnelweb imports HTML either from a live website, from a folder on disk, or a folder
on disk with HTML which was retrieved from a live website and may still have absolute
links refering to that website.

Funnelweb can only import things it can crawl, i.e. content that is linked from
HTML. If your site contains javascript links or password protected content, then
you may have to perform some extra steps to get funnelweb to crawl your
content.

To crawl a live website, supply the crawler with a base HTTP URL to start crawling from.
This URL must be the URL which all the other URLs you want from the site start with.

For example ::

 $> bin/funnelweb --crawler:url=http://www.whitehouse.gov --crawler:maxsize=50  --ploneupload=http://admin:admin@localhost:8080/Plone

will restrict the crawler to the first 50 pages and then convert the content
into a local Plone site.

The site you crawl will be cached locally, so if you run funnelweb again it will run much quicker. If you'd like
to disable the local caching use ::

 $> bin/funnelweb --cache:output=

By default the cache is stored in ``var/funnelwebcache/{site url}/``. You can set this to another directory using::

 $> bin/funnelweb --cache:output=my_new_dir


You can also crawl a local directory of HTML with relative links by just using a ``file://`` style URL ::

 $> bin/funnelweb --crawler:url=file:///mydirectory

or if the local directory contains HTML saved from a website and might have absolute URLs in it,
the you can set this as the cache. The crawler will always look up the cache first ::

 $> bin/funnelweb --crawler:url=http://therealsite.com --crawler:cache=mydirectory

The following will not crawl anything larger than 4Mb ::

 $> bin/funnelweb --crawler:max=400000

To skip crawling links by regular expression ::

  [funnelweb]
  recipe = funnelweb
  crawler-url=http://www.whitehouse.gov
  crawler-ignore = \.mp3
                   \.mp4 

If funnelweb is having trouble parsing the HTML of some pages, you can preprocesses
the HTML before it is parsed. e.g. ::

  [funnelweb]
  recipe = funnelweb
  crawler-patterns = (<script>)[^<]*(</script>)
  crawler-subs = \1\2

If you'd like to skip processing links with certain mimetypes you can use the
``drop:condition`` option. This TALES expression determines what will be processed further ::

  [funnelweb]
  recipe = funnelweb
  drop-condition: python:item.get('_mimetype') not in ['application/x-javascript','text/css','text/plain','application/x-java-byte-code'] and item.get('_path','').split('.')[-1] not in ['class']

.. NOTE:: ``drop`` is a transmogrifier option, not specific to any transform.

Templates
~~~~~~~~~

Funnelweb has a built-in clustering algorithm that tries to automatically extract the content from the HTML template.
This is slow and not always effective. Often you will need to input your own template extraction rules.

Rules are in the form of ::

  (title|description|text|anything) = (text|html|optional) XPath

For example ::

  [funnelweb]
  recipe = funnelweb
  crawler-site_url=http://www.whitehouse.gov
  ploneupload-target=http://admin:admin@localhost:8080/Plone
  template1-title       = text //div[@class='body']//h1[1]
  template1-_delete1    = optional //div[@class='body']//a[@class='headerlink']
  template1-_delete2    = optional //div[contains(@class,'admonition-description')]
  template1-description = text //div[contains(@class,'admonition-description')]//p[@class='last']
  template1-text        = html //div[@class='body']

In the default pipeline there are four templates called `template1`, `template2`, `template3` and `template4`.

If all XPaths are not matched, then the next template will be tried.

When an XPath is applied within a single template, the HTML it matches will be removed from the page.
Another rule in that same template can't match the same HTML fragment.

If a content part is not useful to Plone (e.g. redundant text, title or description) it is a way to effectively remove that HTML
from the content.

For more information about XPath see
- http://www.w3schools.com/xpath/default.asp
- http://blog.browsermob.com/2009/04/test-your-selenium-xpath-easily-with-firebug/

Note that spaces in XPaths must be escaped as ``&#32;``

Site Analysis
~~~~~~~~~~~~~

In order to provide a cleaner-looking Plone site, there are several options to analyse
the entire crawled site and clean it up. These are turned off by default.

To determine if an item is a default page for a container (it has many links
to items in that container, even if not contained in that folder), and then move
it to that folder, use ::

 $> bin/funnelweb --indexguess:condition=python:True

You can automatically find better page titles by analysing backlink text ::

  [funnelweb]
  recipe = funnelweb
  titleguess-condition = python:True
  titleguess-ignore =
	click
	read more
	close
	Close
	http:
	https:
	file:
	img


The following will find items only referenced by one page and move them into
a new folder with the page as the default view. ::

 $> bin/funnelweb --attachmentguess:condition=python:True

or the following will only move attachments that are images and use ``index-html`` as the new
name for the default page of the newly created folder ::

  [funnelweb]
  recipe = funnelweb
  attachmentguess-condition = python: subitem.get('_type') in ['Image']
  attachmentguess-defaultpage = index-html

The following will tidy up the URLs based on a TALES expression ::

 $> bin/funnelweb --urltidy:link_expr="python:item['_path'].endswith('.html') and item['_path'][:-5] or item['_path']"

Plone Uploading
~~~~~~~~~~~~~~~

Uploading happens via remote XML-RPC calls so can be done to a live running site anywhere.

To set where a the site will be uploaded to use ::

 $> bin/funnelweb --ploneupload:target=http://username:password@myhost.com/myfolder

Currently only basic authentication via setting the username and password in the url is supported. If no target
is set then the site will be crawled but not uploaded.

If you'd like to change the type of what's uploaded ::

 $> bin/funnelweb --changetype:value=python:{'Folder':'HelpCenterReferenceManualSection','Document':HelpCenterLeafPage}.get(item['_type'],item['_type'])

By default, funnelweb will automatically create Plone aliases based on the original crawled URLs, so that any old links
will automatically be redirected to the new cleaned-up urls. You can disable this by ::

 $> bin/funnelweb --plonealias:target=

You can change what items get published to which state by setting the following ::

  [funnelweb]
  recipe = funnelweb
  publish-value = python:["publish"]
  publish-condition = python:item.get('_type') != 'Image' and not options.get('disabled')

Funnelweb will hide certain items from Plone's navigation if that item was only ever linked
to from within the content area. You can disable this behavior by ::

 $> bin/funnelweb --plonehide:target=

You can get a local file representation of what will be uploaded by using the following ::

 $> bin/funnelweb --localupload:output=var/mylocaldir


Working directly with transmogrifier (advanced)
-----------------------------------------------

You might need to insert further transformation steps for your particular
conversion usecase. To do this, you can extend funnelweb's underlying
transmogrifier pipeline. Funnelweb uses a transmogrifier pipeline to perform the needed transformations and all
commandline and recipe options refer to options in the pipeline.


You can view pipeline and all its options via the following command ::

 $> bin/funnelweb --pipeline

You can also save this pipeline and customise it for your own needs ::

 $> bin/funnelweb --pipeline > pipeline.cfg
 $> {edit} pipeline.cfg
 $> bin/funnelweb --pipeline=pipeline.cfg

Customising the pipeline allows you add your own personal transformations which
haven't been pre-considered by the standard funnelweb tool.

See transmogrifier documentation to see how to add your own blueprints or add blueprints that
already exist to your custom pipeline.

Using external blueprints
~~~~~~~~~~~~~~~~~~~~~~~~~

If you have decided you need to customise your pipeline and you want to install transformation
steps that use blueprints not already included in funnelweb or transmogrifier, you can include
them using the ``eggs`` option in a funnelweb buildout part ::

  [funnelweb]
  recipe = funnelweb
  eggs = myblueprintpackage
  pipeline = mypipeline.cfg

However, this only works if your blueprint package includes the following setuptools entrypoint
in its ``setup.py`` ::

      entry_points="""
            [z3c.autoinclude.plugin]
            target = transmogrify
            """,
            )

.. NOTE:: Some transmogrifier blueprints assume they are running inside a Plone
          process such as those in `plone.app.transmogrifier` (see
	  http://pypi.python.org/pypi/plone.app.transmogrifier).  Funnelweb
	  doesn't run inside a Plone process so these blueprints won't work. If
	  you want upload content into Plone, you can instead use
	  transmogrify.ploneremote which provides alternative implementations
	  which will upload content remotely via XML-RPC.
	  ``transmogrify.ploneremote`` is already included in funnelweb as it is
	  what funnelweb's default pipeline uses.

Attributes available in funnelweb pipeline
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When using the default blueprints in funnelweb the following are some of the attributes that
will become attached to the items that each blueprint has access to. These can be used in the various
condition statements etc. as well as your own blueprints.

``_site_url``
  The base of the url as passed into the webcrawler

``_path``
  The remainder of the URL. ``_site_url`` + ``_path`` = URL

``_mimetype``
  The mimetype as returned by the crawler

``_content``
  The content of the item crawled, include image, file or HTML data.

``_orig_path``
  The original path of the item that was crawled. This is useful for setting redirects so
  you don't get 404 errors after migrating content.

``_sort_order``
  An integer representing the order in which this item was crawled. Helps to determine
  what order items should be sorted in folders created on the server if your site
  has navigation which has links ordered top to bottom.

``_type``
  The type of object to be created as returned by the "typeguess" step

``title``, ``description``, ``text``, etc.
  The template steps will typically create fields with content in them taken from ``_content``

``_template``
  The template steps will leave the HTML that wasn't seperated out into different fields in this
  attribute.

``_defaultpage``
  Set on an Folder item where you want to tell the uploading steps to set the containing item
  mentioned in ``_defaultpage`` to be the default page shown on that folder instead of a content listing.

``_transitions``
  Specify the workflow action you'd like to make on an item after it's uploaded or updated.

``_origin``
  This is used internally with the `transmogrify.siteanalysis.relinker` blueprint as a way to
  tell it that you have changed the ``_path`` and you now want the relinker to find any links that
  refer to ``_origin`` to now point to ``_path``.

The Funnelweb Pipeline
~~~~~~~~~~~~~~~~~~~~~~

see ``funnelweb/runner/pipeline.cfg``
or type ::

 $> bin/funnelweb --pipeline


.. include:: funnelweb/runner/pipeline.cfg
   :literal:





