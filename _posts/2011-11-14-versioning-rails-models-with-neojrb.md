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

![Versioning](http://blog.vivekprahlad.com/assets/versioning_with_neo4jrb/versioning.png)

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

![Version 1](http://blog.vivekprahlad.com/assets/versioning_with_neo4jrb/version1.png)

### Version 2
With version 2, we link the driver to a sports car - a Porsche.


{% highlight ruby %}
driver.sports_cars << Sportscar.create(:brand => 'Porsche')
driver.save!
{% endhighlight %}

![Version 2](http://blog.vivekprahlad.com/assets/versioning_with_neo4jrb/version2.png)

Now, both the driver and the current version are linked to the Porsche.

### Version 3
With version 3, we link the driver to a second car - a Ferrari. The graph now starts getting complicated.


![Version 3](http://blog.vivekprahlad.com/assets/versioning_with_neo4jrb/version3.png)

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

![Version 4](http://blog.vivekprahlad.com/assets/versioning_with_neo4jrb/version4.png) 
