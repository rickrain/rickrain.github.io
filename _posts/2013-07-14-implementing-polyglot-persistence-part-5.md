---
title: "Implementing Polyglot Persistence – Part 5"
date: 2013-07-14 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

In this post I will complete the implementation of my MongoDB repository by implementing the Review operations previously defined in my **IProductRepository** interface. This will also be the post where I visit some of the Create, Update, and Delete API’s of the MongoDB C# Drivers. Everything up to this point has been variations of Read API’s.

```csharp
// Review Operations
Review GetReview(string reviewId);
List<Review> GetReviews(string prodId, int pageIndex, int pageSize);
string AddReview(string prodId, Review review);
bool UpdateReview(string reviewId, Review review);
void DeleteReview(string reviewId);
```

However, before I get into the implementation of these methods, I need to revisit my **MongoHelper** class to support a new collection in MongoDB called “Reviews”, which is the name of the collection where I’ll be storing product reviews.

## Update MongoHelper to Support my “Reviews” Collection in MongoDB

In [part 3](https://rickrainey.com/2013/06/25/implementing-polyglot-persistence-part-3/) I first introduced my **MongoHelper** class and talked about class maps. For the reviews collection, I’ll need a similar class map, but this time only to indicate that the “Id” field in my **Review** class will be represented as an ObjectId. To achieve this, I added this small piece of code to my **RegisterClassMaps** method.

```csharp
BsonClassMap.RegisterClassMap<Review>(cm =>
{
    cm.AutoMap();
    cm.IdMemberMap.SetRepresentation(BsonType.ObjectId);
});
```

While I was in this file, I also added a new read-only property to return a reference to the Reviews collection. The reviewsCollectionName is a string set from an AppSetting in web.config (just like I did for the products collections).

```csharp
public static MongoCollection<Review> ReviewsCollection {
    get { return mongoDatabase.GetCollection<Review>(reviewsCollectionName); }
}
```

With those few changes in place, I can now proceed to the implementation of the Review operations.

## The GetReview Implementation

This method is among the simplest in all the methods I’ll cover in this post and the use case is very straight forward – you just want to retrieve a single review for a product. The way I’ve setup my controller, a review is accessed by including the “Id” of the review in the URL. For example,

![HTTP GET Review](/assets/img/pp-part5-01.png)

Recall that my class map defines the Id property as an ObjectId. So, to retrieve this Id, I just need to look for it in the collection using the **FindOneById** method.

```csharp
public ProductService.Model.Review GetReview(string reviewId)
{
    ObjectId objectId;
    if (ObjectId.TryParse(reviewId, out objectId))
        return MongoHelper.ReviewsCollection.FindOneById(objectId);
    else
        return null;
}
```

## The GetReviews Implementation

This method returns a list of reviews for a given product Id. Since there could be a large number of reviews, this method also supports some basic pagination. But, to simply return all reviews for a product Product Id, this URL would work.

![HTTP GET Reviews](/assets/img/pp-part5-02.png)

In this implementation, I’m building a query that will run on the server, searching the Reviews collection for all reviews with a **ProdId** matching the product Id passed in on the URL. Then, I simply pass that query into the **Find** method with the necessary pagination (skip, take).

```csharp
public List<ProductService.Model.Review> GetReviews(string prodId, int pageIndex, int pageSize)
{
    ObjectId objectId;
    if (ObjectId.TryParse(prodId, out objectId))
    {
        var query = Query<Review>.EQ(r => r.ProdId, objectId.ToString());
        return MongoHelper.ReviewsCollection.Find(query).
            Skip(pageIndex * pageSize).Take(pageSize).ToList<Review>();
    }
    else
        return null;
}
```



## The AddReview Implementation

This is the first method where we actually add something to a collection. The use case here is a user has purchased a product and wants to write a review for it. For this, you need to POST data to the server using this URL and JSON in the Request Body of the message.

![HTTP POST Add Review](/assets/img/pp-part5-03.png)

You may notice in the JSON that I didn’t specify things like **ProdId**, **Id**, and **ReviewDate**. All of which are properties of the Review class. I chose to set these explicitly in the method. For **ProdId**, I use the product Id from the URL. For **ReviewDate**, I just timestamp the document with the current time. And, for **Id**, I’m generating an new ObjectId before inserting.

```csharp
public string AddReview(string prodId, ProductService.Model.Review review)
{
    review.Id = ObjectId.GenerateNewId().ToString();
    review.ProdId = prodId;
    review.ReviewDate = DateTime.UtcNow;
    MongoHelper.ReviewsCollection.Insert(review);
    return review.Id;
}
```

## The UpdateReview Implementation

To update an existing review, this translates to a PUT verb on my controller with similar JSON in the Request Body for the data I want to update.

![HTTP PUT Update Review](/assets/img/pp-part5-04.png)

In the implementation, I’m applying updates to the properties of the existing review. I’m creating a query to find the review Id passed in. Next, I’m telling MongoDB to update the **ReviewDate**, **Rating**, and **Comments** fields in the review. The Combine method is particularly handy when you need to make updates to multiple fields in the document.

```csharp
public bool UpdateReview(string reviewId, ProductService.Model.Review review)
{
    ObjectId objectId;
    if (ObjectId.TryParse(reviewId, out objectId))
    {
        var query = Query<Review>.EQ(r => r.Id, objectId.ToString());
        var updates = Update<Review>.Combine(
            Update<Review>.Set(r => r.ReviewDate, DateTime.UtcNow),
            Update<Review>.Set(r => r.Rating, review.Rating),
            Update<Review>.Set(r => r.Comments, review.Comments));
        var result = MongoHelper.ReviewsCollection.Update(query, updates);
        return result.Ok;
    }
    else
        return false;
}
```

## The DeleteReview Implementation

Finally, to remove a review document from the reviews collection, the DELETE verb is set with the URL indicating the id of the review to delete.

![HTTP DELETE Delete Review](/assets/img/pp-part5-05.png)

Once the reviewId is parsed into an ObjectId, it’s a simple matter of finding the document and calling Remove.

```csharp
public void DeleteReview(string reviewId)
{
    ObjectId objectId;
    if (ObjectId.TryParse(reviewId, out objectId))
    {
        var query = Query<Review>.EQ(r => r.Id, objectId.ToString());
        MongoHelper.ReviewsCollection.Remove(query);
    }
}
```

## Conclusion

This wraps up my journey through the document database category and MongoDB-as-a-servicefrom Mongolab. The service from Mongolab is very powerful and easy to use. The C# Drivers are a gem and the more I use them the more I appreciate everything they do.

My solution is available [here](https://github.com/rickrain/ImplementingPolyglotPersistence) for anyone interested in reviewing the full end-to-end implementation.


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