---
title: "Implementing Polyglot Persistence – Part 3"
date: 2013-06-25 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

In this post I will start creating a Mongolab repository to perform CRUD operations against the MongoDB database I created in [Part 2](https://rickrainey.com/2013/06/14/implementing-polyglot-persistence-part-2/). As a starting point, I will use the solution that I introduced in [Part 1](https://rickrainey.com/2013/05/22/implementing-polyglot-persistence-part-1/), adding a new repository for Mongolab.

## Get the MongoDB Client Library
To start, I’ll need some client libraries to program against my MongoDB. I’ll be using the C# drivers that 10gen supports, which can be found here. This is an outstanding resource for documentation and samples on how to use the drivers.

Using the “**Manage Nuget Packages…**” feature in Visual Studio 2012 and searching for “mongocsharpdriver” makes installing the drivers for my Mongolab project a one-click task.

![ProductService.Repository.Mongolab project](/assets/img/pp-part3-01.png)

## Handling Multiple Product Schemas

MongoDB is a “schemaless” database. While the database may not be enforcing schema, it’s still helpful to have some basic schema in place (in the application). Recall from Part 1 that I created a very basic model (schema) for products and reviews in **ProductService.Model**. However, in Part 2 where I created my database, I was not restricted to the model and MongoDB was perfectly happy to take whatever I gave it.

In MongoDB, you can reference the collection as a generic collection of BsonDocument types. Some examples of how to use this document type are in the [C# Driver Tutorial](http://mongodb.github.io/mongo-csharp-driver/2.2/getting_started/). This exhibits a great deal of flexibility when working with the collection. With that flexibility, however, comes challenges in how you’re able to query the collection. For example, if you’re not able to assume any model (or schema) at all, then field names for the entities obviously can be different or non-existent. Even the casing of field names starts to become an issue (“Price” does not equal “price”).

Having a basic model for products and insuring items in the collection adhere to that basic model I’ve found makes things much simpler. Yet, I still have the power of schema flexibility in the database. To handle the various properties that a product might have, I extended my **ProductService.Model.Product** to include a new property called “ExtraElements”. I extended this internal to my repository since this is a capability the client library supports and not relevant to other potential repository implementations using the model.

Now, anytime I refer to the “products” collection in my database, I do so using this new *ProductDocument** class as the type. I wrote a little helper class to return an instance of the collection for me using this type.

```csharp
public static MongoCollection<ProductDocument> ProductsCollection {
    get { return mongoDatabase.GetCollection<ProductDocument>(productsCollectionName); }
}
```

Because I have two classes representing products (**Product** and **ProductDocument**), I need to give some hints to the client library so it can properly serialize these. I also need to tell it how to handle Id’s (more on that later). This is accomplished using the BsonClassMap, as shown here.

```csharp
public static void RegisterClassMaps()
{
    BsonClassMap.RegisterClassMap<Product>(cm =>
    {
        cm.AutoMap();
        cm.SetIgnoreExtraElements(true);
        cm.IdMemberMap.SetRepresentation(BsonType.ObjectId);
    });

    BsonClassMap.RegisterClassMap<ProductDocument>(cm =>
    {
        cm.AutoMap();
        cm.SetExtraElementsMember(cm.GetMemberMap(c => c.ExtraElements));
    });
}
```

In the first class map, I’m telling it that when serializing a product as a **ProductService.Model.Product**, ignore any extra properties that may exist for the product. I’m also telling it that the “Id” field should be represented as an ObjectId.

In the second class map, I’m telling it that when serializing a product as a **ProductService.Repository.Mongolab.ProductDocument**, store any extra data it finds for the product in the “ExtraElements” property I added above.

## The Id Field

When I inserted the JSON documents into my products collection, a “_id” field was created automatically for each product. I could have generated the field myself and populated it differently, but I chose to just let the database do it for me when I added these documents in Part 2.

![JSON Document](/assets/img/pp-part3-02.png)

The “_id” field is actually an ObjectId, which is a structure of 12 bytes (no less and no more) and not to be confused for a Guid (although similar). Its value takes into account things like time, process id, machine and an incrementing counter.

My **ProductService.Model.Product** already has an “Id” field and the client library automatically maps any field named “Id” to the “_id” field in the database. So, my “Id” is already handled and queries for products by Id will just work.

## Implementing the Repository
With everything described above in place, the implementation of the repository methods is very simple. Here I’ll show a few of the methods and the output using Fiddler to generate requests.

### The GetProduct Implementation

This method returns the base **Product** representation of a product. Recall from above that my collection is actually a collection of **ProductDocument**. Using the **FindOneByIdAs<>** method, I can tell Mongo to return the document as a **Product** instance (ie: no product details).

```csharp
public ProductService.Model.Product GetProduct(string prodId)
{
    var id = ObjectId.Parse(prodId);
    if (id != null)
        return MongoHelper.ProductsCollection.FindOneByIdAs<Product>(id);
    else
        return null;
}
```

### The GetProductDetails Implementation

This will return the product and any additional fields stored for the product.

One side effect of this is the “Extra Elements” field wrapping the additional product details. Ideally, I’d like for the additional product details to appear as they do in my MongoDB without being wrapped by the “Extra Elements” field. Perhaps in a future post I’ll fix this with some custom serialization. For now, I’m happy to accept this little nuance

### The GetCategories Implementation

The use case for this method is one where you need a list of all available categories of products in the collection. For example, to populate a list of categories on a web page so users can browse products by category.

```csharp
public List<string> GetCategories()
{
    var categories = MongoHelper.ProductsCollection.Distinct<string>("Categories");
    if (categories != null)
        return categories.ToList();
    else
        return null;
}
```

## Conclusion

I’ll get to the rest of the methods in the repository in Part 4. Things get a little more interesting when implementing search. Although, as you can probably imagine, the client libraries simplify this considerably.

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