# Sitemap Generator Module

This is a [sitemap](http://www.sitemaps.org/) module generator for [Play Framework](http://www.playframework.org/) 2.1. It uses [SitemapGen4j](https://code.google.com/p/sitemapgen4j/) to generate the sitemap files.

## How it works

### Configuring

The first step is include the sitemapper in your dependencies list, in `Build.scala` file:

```
import sbt._
import Keys._
import play.Project._

object ApplicationBuild extends Build {

  val appName         = "sitemapper-sample"
  val appVersion      = "1.0-SNAPSHOT"

  val appDependencies = Seq(
    // Add your project dependencies here,
    javaCore,
    javaJdbc,
    javaEbean,
    "com.edulify" % "sitemapper_2.10" % "1.1"
  )

  val main = play.Project(appName, appVersion, appDependencies).settings(
    // Add your own project settings here
    resolvers += Resolver.url("sitemapper repository", url("http://blabluble.github.com/modules/releases/"))(Resolver.ivyStylePatterns)
  )

}

```

Don't forget to add the resolver to your list of resolvers, or it won't work!

Ok, now you need to configure the module in your `application.conf` (the configuration syntax is defined by [Typesafe Config](https://github.com/typesafehub/config)):

```
sitemap.dispatcher.name = "Sitemapper"
sitemap.baseUrl="http://example.com"
sitemap.baseDir="/complete/path/to/directory/where/sitemap/files/will/be/saved"
sitemap.providers=sitemap.providers.ArticlesUrlProvider,sitemap.providers.AuthorsUrlProvider
```

- dispatcher.name: name of the dispatcher used in the Akka system.
- baseUrl: the base URL for links provided in your sitemap.
- baseDir: as it says, the complete path to directory where you want your sitemap xml files be saved. If you don't provide this value, the sitemap files will be saved under your `public` directory.
- providers: a comma separated list of *provider classes* that will generate URLs for your sitemap (see below).

You must also configure an execution context that will execute the sitemap job:

```
sitemap.dispatcher.name="sitemap-generator"
akka {

  // see play's thread pools configuration for more details

  actor {

    sitemap-generator = {
      fork-join-executor {
        parallelism-factor = 2.0
        parallelism-max = 24
      }
    }

  }
}
```

Since the Sitemap Generator runs as a job, you must start `com.edulify.modules.sitemap.SitemapJob` in your Global class. We provide an helper method that uses the execution context configured above. Here is an example:

```java
import play.GlobalSettings;
import com.edulify.modules.sitemap.SitemapJob;

public class Global extends GlobalSettings {

  @Override
  public void onStart(Application app) {
    SitemapJob.startSitemapGenerator();
  }
}
```

### Using

With everything configured, now its time to add some URLs to your sitemap(s). The simplest way is add the `@SitemapItem` annotation to your controllers methods which doesn't expect arguments:

```java
package controllers;

import com.edulify.modules.sitemap.SitemapItem;
import com.redfin.sitemapgenerator.ChangeFreq;

public class Articles extends Controller {

  @SitemapItem
  public Result listArticles() { ... }

  @SitemapItem(changefreq = ChangeFreq.MONTHLY, priority = 0.8)
  public Result specialArticles() { ... }
}
```

This will add the route for such method in the sitemap. The default values for the *changefreq* and *priority* attributes are **daily** and **0.5** repectevely.

For methods that needs arguments, you must implement a URL provider that will add the generated URL to sitemap. Here is an example:

```java
package sitemap.providers;

import java.net.MalformedURLException;
import java.util.List;

import models.Article;

import controllers.routes;
import play.Play;

import com.edulify.modules.sitemap.UrlProvider;

import com.redfin.sitemapgenerator.ChangeFreq;
import com.redfin.sitemapgenerator.WebSitemapUrl;
import com.redfin.sitemapgenerator.WebSitemapGenerator;

public class ArticlesUrlProvider implements UrlProvider {

  @Override
  public void addUrlsTo(WebSitemapGenerator generator) {
    String baseUrl = Play.application().configuration().getString("sitemap.baseUrl");

    List<Article> articles = Article.find.all();
    for(Article article : articles) {
      String articleUrl = routes.Application.showArticle(article.id).url();
      try {
        WebSitemapUrl url = new WebSitemapUrl.Options(String.format("%s%s", baseUrl, articleUrl))
                                             .changeFreq(ChangeFreq.DAILY)
                                             .lastMod(article.createdAt)
                                             .priority(0.5)
                                             .build();
        generator.addUrl(url);
      } catch(MalformedURLException ex) {
        play.Logger.error("wat? " + articleUrl, ex);
      }
    }
  }

}
```

Remember that you'll need to add the absolute path of the added URLs.

In order to deliver your sitemaps files, you can use the `SitemapController` provided by this module. For this, you can simply add the following route to your `routes` files:

<pre>
GET     /sitemap$suffix<[^/]*>.xml   com.edulify.modules.sitemap.SitemapController.sitemap(suffix: String)
</pre>

Or you can write your own file deliver. Just remember that the `sitemap_index.xml`, when generated, links to sitemap#.xml on the defined *baseUrl*, i.e., the `sitemap_index.xml` will like look this:
```xml
<?xml version="1.0" encoding="UTF-8"?>

<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">

    <sitemap>
      <loc>http://www.example.com/sitemap1.xml</loc>
    </sitemap>

    <sitemap>
      <loc>http://www.example.com/sitemap2.xml</loc>
    </sitemap>

</sitemapindex>
```

for a `baseUrl = http://www.example.com`.

## Issues

Report issues at https://github.com/blabluble/play-sitemap-module/issues

## Licence

This software is licensed under the Apache 2 license, quoted below.

Copyright 2013 Edulify.com (http://www.edulify.com).

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this project except in compliance with the License. You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0.

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.