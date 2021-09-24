---
title: "Implementing Polyglot Persistence – Part 1"
date: 2013-05-22 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

_**NOTE: This post was recovered from archives and unfortunately some of the images were lost.  I've restored as much as I can.  However, due to the age of this post I won't be re-producing the missing images.**_


In my previous post I talked about what polyglot persistence could look like.  In this post I’ll start digging into some of the implementation details.  I’ve decided to make this a series of posts, hence the “Part 1” suffix in the title.

In this post, I’m going to build out the web service layer for the **product catalog service** using a repository pattern.  There’s plenty written about the repository pattern so I won’t go into those details here.  This first repository implementation will be a simple **in-memory repository** to define the model classes, service interface, and test out the web service layer,   In Part 2 I’ll build a repository against a document database.

> Disclaimer: This is not a feature rich implementation of a product catalog service.  It’s an example of some operations you might expect from such a service that provide some interesting use cases for kicking the tires of a document database.

## The Model

For product catalog service, I’ve got two classes comprising the model; **Product** and **Review**.

**(missing image #1 here)**

The Product class defines some common properties that you might expect to see for any product of any category. Note that I’m not making any effort here to further define models for various product categories.

**(missing image #2 here)**

The Review class is used to represent a review of a particular product.

**(missing image #3 here)**

## The Product Service Interface

For the web service API, I defined the following operations to interact with the model above. These operations will cover the various CRUD operations, plus give some interesting use cases for querying the repository and supporting basic pagination.

**(missing image #4 here)**

## The ASP.NET WebApi Controller

The  ASP.NET WebApi Controller supports these operations that rely on a repository implementation to carry out the work.

```http
GET    api/product/categories
GET    api/product/{prodId}
GET    api/product/{prodId}/details
POST   api/product/{prodId}/review/
GET    api/product/{prodId}/review
GET    api/product/{prodId}/review/{reviewId}
PUT    api/product/{prodId}/review/{reviewId}
DELETE api/product/{prodId}/review/{reviewId}
GET    api/product/search/category?value=clothing
```

The Http Routes to support this line-up of operations is shown here.

**(missing image #5 here)**

## Testing Things Out

The `ASP.NET Web Api Controller` is currently using an in-memory implementation of the repository described above. It’s quick, easy, and doesn’t require any database configuration to run it. It’s got a few products and reviews prepopulated too so I can begin testing my service layer.

**(missing image #6 here)**

Using Fiddler, I can interact easily with the service to make sure things are working as expected.

![Fiddler Composer Tab](/assets/img/pp-part1-07.png)

![Fiddler JSON output](/assets/img/pp-part1-08.png)

**(missing image #9 here)**

![Fiddler JSON output](/assets/img/pp-part1-10.png)

I won’t go through all the operations here.  If you want to explore this solution further you can download it from [here](https://github.com/rickrain/ImplementingPolyglotPersistence).

Now that this is working I can focus on adding a new repository implementation using a document database, which I’ll do in [Part 2](https://rickrainey.com/2013/06/14/implementing-polyglot-persistence-part-2/).

{% if page.comments %}
<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
/*
var disqus_config = function () {
this.page.url = "{{ site.baseurl }}";  // Replace PAGE_URL with your page's canonical URL variable
this.page.identifier = "{{ page.url }}"; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
};
*/
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://rickrainey.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
                            
{% endif %}