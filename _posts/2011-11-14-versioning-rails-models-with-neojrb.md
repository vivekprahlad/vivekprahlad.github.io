---
layout: post
title: Versioning Rails models with Neo4j.rb
---
I recently helped add versioning support to [Neo4j.rb](https://github.com/andreasronge/neo4j). Versioning only works for Rails models at the moment. To add versioning to your model, you'll need include the Versioning module:

{% highlight ruby %}
class VersionableModel < Neo4j::Rails::Model
  include Neo4j::Rails::Versioning
end
{% endhighlight %}

### How it works
Versioning creates snapshots under a given rails model instance. Note that snapshots aren't Rails models, but vanilla nodes. The relationship from the instance to its snapshot versions captures information about the snapshot, such as the version number, the class that is being versioned, and the id of the versioned model. When asked to retrieve a particular version of an model, the versioning module internally does a lucene search using all of these parameters.

![Versioning](/assets/versioning_with_neo4jrb/versioning.png)

### Full snapshots
Snapshots currently capture all the properties of an instance. The snapshots create links to the same nodes that are linked to an instance, with one important difference. The snapshot's relationships all have a `'version_'` prefix - the prefix is added in order to distinguish between 'regular' relationships and those created for versioning. Both incoming as well as outgoing relationships are recorded.

Let's look at an example:
  
  
{% highlight ruby %}
class SportsCar < Neo4j::Rails::Model
  include Neo4j::Rails::Versioning
  property :brand
end
 
class Driver < Neo4j::Rails::Model
  include Neo4j::Rails::Versioning
  property :name
  has_n(:sports_cars)
end
{% endhighlight %}

### Version 1
With version 1 of the driver, there are no links to any cars:

{% highlight ruby %}
driver = Driver.create(:name => 'Walter Plinge')
{% endhighlight %}

![Version 1](/assets/versioning_with_neo4jrb/version1.png)

### Version 2
With version 2, we link the driver to a sports car - a Porsche.


{% highlight ruby %}
driver.sports_cars << Sportscar.create(:brand => 'Porsche')
driver.save!
{% endhighlight %}

![Version 2](/assets/versioning_with_neo4jrb/version2.png)

Now, both the driver and the current version are linked to the Porsche.

### Version 3
With version 3, we link the driver to a second car - a Ferrari. The graph now starts getting complicated.


![Version 3](/assets/versioning_with_neo4jrb/version3.png)

The driver, as well as the latest snapshot, are both linked to two cars - the Porsche and the Ferrarri.

### Working with a version

To retrieve a snapshot, you'll need to call the version API.

{% highlight ruby %}
instance.version(version_number)
{% endhighlight %}

The snapshot object will respond to the exact same properties as those on instance. It also allows relationships to be traversed using the incoming / outgoing API. The prefix stuff I'd mentioned above is handled transparently, so you can continue to use the same relationship names that you've used in your model.

{% highlight ruby %}
driver.version(3).outgoing(:sports_cars) #Returns two cars as expected
{% endhighlight %}

### Reverting to a snapshot

To revert to a particular version, simply call

{% highlight ruby %}
instance.revert_to(version_number)
{% endhighlight %}

Revert restores properties and relationships, and creates a new version. So in the driver / car example, let's see what the driver example looks like when we revert to version 2:

{% highlight ruby %}
driver.revert_to(2)
{% endhighlight %}

![Version 4](/assets/versioning_with_neo4jrb/version4.png) 
This creates a new version (version 4), and ensures that the driver is linked to one sports car, just like it was with version 2.

### Max versions
If you'd like to limit the maximum number of versions, use the max_versions declaration in your model class. This deletes the oldest version once the max versions threshold is exceeded.

{% highlight ruby %}
class MaxVersionModel < Neo4j::Rails::Model
  include Neo4j::Rails::Versioning
 
  max_versions 2
 
end
{% endhighlight %}

### Gotchas

In case any of the nodes linked to an instance are deleted, the versioning support makes no attempt to capture this. The relationship between the snapshot and such a node simply disappears.

In other words, when related nodes are deleted, reverting a node to it's previous version will not recreate deleted nodes and relationships.

### Future improvements
The versioning support doesn't currently capture properties on relationships, but that should be relatively straightforward to address.

Deleting a node does not always delete its snapshots (this can happen whenever snapshots have relationships to other nodes). On second thoughts, this sounds more like a bug than an improvement :-).

Finally, supporting delta updates would allow more efficient storage (like I said earlier, each snapshot currently stores all the properties of a model).

That's about it, feedback and comments welcome!

PS: In case you're wondering who Walter Plinge is, check out [http://en.wikipedia.org/wiki/Walter_Plinge](http://en.wikipedia.org/wiki/Walter_Plinge)
