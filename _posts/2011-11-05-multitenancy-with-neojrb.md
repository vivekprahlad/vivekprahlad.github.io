---
layout: post
title: Multitenancy with Neo4j.rb
---
A little while ago, I'd helped add multitenancy support to [Neo4j.rb](https://github.com/andreasronge/neo4j). This allows the same Neo4j database instance to support multiple tenants/customers simultaneously. There are several ways of achieving multitenancy, but the main aim of the current solution was to minimize the number of required changes to an existing codebase.

So here's how it all works.

### Partitioning graphs
The Neo4j.rb multitenancy approach partitions the graph in such a way that all queries and traversals are scoped to a given tenant. This means that queries like Order.all, for example would return different (and correct) results depending on what the current tenant is.

Once Neo4j supports sharding, it should be possible to adapt this scheme to take advantage of it.

### Reference nodes
The reference node is a starting point in the graph space. All Neo4j graphs are connected by default, which basically means that nodes and relationships cannot exist in isolation. The multitenancy feature works by 'moving' the reference node to whatever the current tenant is.

### The Neo4j.rb metamodel
Neo4j.rb stores type metadata in the graph. Let's consider an example where we have two models: Tenant and Country.

In the screenshot below, the home icon represents the reference node, which the default starting point in the graph. From the reference node, each model type has an outgoing relationship, named after the model class. Neo4j.rb refers to the node at the end of this relationship as a Rule node. The 'all' rule node is connected to every instance, and its count property keeps track of the number of instances.

This particular database has a single tenant instance...

... and three countries. The `_classname` property stores the Rails model class name for a given node.
### Tenants
While creating new tenants, it's often necessary to set up data for each tenant. Here's one way of doing this:
{% highlight ruby %}
class Tenant < Neo4j::Rails::Model
  property :name
  ref_node { Neo4j.default_ref_node }

  after_create :create_default_data

  def create_default_data
    Neo4j.threadlocal_ref_node = self
    load("#{Rails.root}/db/tenant_default_data.rb")
  end
end
{% endhighlight %}
In case it takes a while to set up the data for a client, it's worth doing the data setup in the background.

### 'Movable' reference nodes
Changing the reference node to a tenant ends up effectively partioning the graph. All tenant specific entities end up getting attached under the tenant, like so for Tenant 1:

... and for Tenant 2.



### Setting the Threadlocal reference node
Neo4j.rb allows a reference node to be set for the current thread. A :before_filter method in your controller is a reasonable place to set the threadlocal reference node.

{% highlight ruby %}
class OrdersController < ApplicationController
 
before_filter :authenticate_user!, :ensure_tenant_setup!

  protected
  def ensure_tenant_setup!
    Neo4j.threadlocal_ref_node = current_user.tenant
  end
end
{% endhighlight %}
### Lucene Indexing

By default, for every Rails model, Neo4j.rb creates lucene indices using this format:

`<TopLevelModulename>_<NextlevelModuleName>_<ModelClassName>-exact/fulltext`

The multitenancy work adds the tenant name as an additional dimension to the index name. Each tenant node contributes an index prefix. The prefix makes it possible to partition the lucene index on a per tenant basis.

`<Tenant Name>_<TopLevelModulename>_<NextlevelModuleName>_<ModelClassName>`

The net effect is that lucene queries are scoped to the current tenant. (Except in the case of shared models, which are accessible across all tenants and use a single lucene index).
### Reference data / Shared models
What about data that's shared across tenants? Examples could be a list of valid currencies, or a list of countries. It doesn't make sense to replicate this data on a per tenant basis. To handle these kind of entities, you can declare the model's reference node to be the default reference node, which ensures that all instances of the model are accessible across all tenants.


{% highlight ruby %}
class Country < Neo4j::Rails::Model

  property :name
  ref_node { neo4j.default_ref_node }

end
{% endhighlight %}
### Gotchas

* *Clearing the reference node*<br/>
  Neo4j.rb has Rack middleware that resets the reference node after every request. In Rails' multithreaded mode, you could end up with stale / wrong reference nodes in case this were not done.
* *The 'too many open files' problem*<br/>
  Since sets of lucene indices are created for each tenant, you can expect approximately 7 * N * T open files, where N is the total number of models, and T is the total number of tenants. By default, Neo4j's Lucene IndexWriters are never closed (keeping the writers open improves performance). The default per process file limit is 1024 on Linux.
  I've submitted a pull request to the Neo4j database here to fix this issue. (See [https://github.com/neo4j/community/pull/51](https://github.com/neo4j/community/pull/51)). The fix involves using a configurable LRU cache for Lucene index writers/searchers.
