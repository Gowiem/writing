# Pagination with Ember, Rails, and jsonapi-resources

## Intro

Adding pagination to a Rails JSONAPI backed Ember app is not a totally plug-and-play development task, which is what I would have expected. There is of course [ember-cli-pagination](LINK) which is a library that tries to make it as simple as possible. Unfortunately, I wasn't able to get that working out of the box and it didn't seem like the type of library I wanted to put the effort into banging on to make work. Luckily, I found [this awesome post on EmberIgniter.com](LINK) which covers pagination with a JSONAPI very nicely. This post and the code that follows is heavily based off of that post, so big thanks to [@thefrank06](https://twitter.com/thefrank06) for paving the way. This post is fairly similar to that one however it is made to be a bit more for the advanced Emberino, more reuse friendly, and focuses on using the [jsonapi-resources Rails gem](LINK). Keep reading if that sounds of interest!

## Setup

Before we begin, let's make sure we're on the same page in regards to our dependencies. Here is what you should have in place:
- A [`jsonapi-resources`](LINK) Rails API
- An Ember-CLI frontend using Ember Data
- [`ember-truth-helpers`](LINK) installed (cause it's a great lib and allows us to skip adding a handlebars helper).
- A model you're looking to paginate. Using the classic blog example, I'll refer to this as "Post" from now on.
- An understanding of [how query params work in Emberjs](LINK) Apps.

Got all that? Great, we're ready to go.

## `jsonapi-resources` Updates

A couple simple things need to get added to our backend app to enabled our pagination via `jsonapi-resources`. Let me throw some code at you:

```ruby
# backend/config/initializers/jsonapi-resources.rb
JSONAPI.configure do |config|
  config.top_level_meta_include_page_count = true
end
```

```ruby
# backend/app/resources/post_resource.rb
class PostResource < JSONAPI::Resource
  attributes :title, :content, :you_get_the_idea

  paginator :paged
end
```

So what are we doing here? First, we create an `jsonapi-resources` initializer (or append to our existing one) so we can let jsonapi know that we'd like to [include the page count in our meta object](https://github.com/cerebris/jsonapi-resources/commit/3db8260978119ac278d2db92b17a2611c36d754a) when requesting an index route for our models. Second, we're updating our `PostResource` (which is our JSON blueprint) to have a `paginator` so the API knows what we're talking about when we send it page number and page size query params later on. We're using the `:paged` paginator here as that is likely what you're looking to do, but if you'd like to do offset or your own custom pagination rules then [check out these docs](LINK).

It's that simple! Our backend is good to go, so let's jump over to the frontend side of things.

## Ember Updates

A few things need to happen to the frontend to make pagination come together:

1. We need to update our `ApplicationSerializer` so it pulls our pagination meta information and makes it available to us.
2. We need to properly request the page number and page size when fetching our model at the Route layer.
3. We need to add our query params to our Controller and make sure they have some defaults.
4. Finally, we need to provide a UI to our user's so they can page through the Posts.

Let's attack those one-by-one!

### Application Serializer Update

```javascript
// frontend/app/mixins/pagination-serializer.js
import Ember from 'ember';

export default Ember.Mixin.create({
  normalizeQueryResponse(store, clazz, payload) {
    const result = this._super(...arguments);
    result.meta = result.meta || {};

    if (payload.links && payload.meta && payload.meta['page-count']) {
      result.meta.pagination = this._createPagination(payload);
    }

    return result;
  },

  _createPagination(data) {
    let meta = { links: {}, pageCount: data.meta['page-count'] };
    Object.keys(data.links).forEach(type => {
      const link = data.links[type];
      meta.links[type] = {};
      let anchor = document.createElement('a');
      anchor.href = link;
      anchor.search.slice(1).split('&').forEach(pairs => {
        const [param, value] = pairs.split('=');
        if (param == 'page%5Bnumber%5D') {
          meta.links[type].number = parseInt(value);
        }
        if (param == 'page%5Bsize%5D') {
          meta.links[type].size = parseInt(value);
        }
      });
      anchor = null;
    });
    return meta;
  }
});
```

The above code (literally copy and pasted from our friend @INSERT_GUYS_NAMEHERE) is an `Ember.Mixin` that you should add to your `ApplicationSerilaizer`. I broke this out as a Mixin so that it gives this functionality a name so it is obvious what it is doing, but feel free to copy/paste those methods into your serializer if you don't want to add another file. If you don't already have an application serializer you can generate one via `ember g serializer application`.

The Mixin does some pulling apart of the `links` JSON returned by the API to create a `meta.pagination` object on our `DS.RecordArray`. We'll use that `pagination` object later to create the links needed to paginate via our routes.

### Update our Route to query properly

```javascript
// frontend/app/routes/posts.js
import Ember from 'ember';

export default Ember.Route.extend({
  model(params) {
    return this.store.query('post', { page: {
        number: params.page,
        size: params.size
      }
    });
  },

  queryParams: {
    page: {
      refreshModel: true
    },
    size: {
      refreshModel: true
    }
  }
});
```

Here we update our `PostsRoute` to query for our model using the params 'page' and 'size'. This creates requests like `/posts?page[number]=1&page[size]=20` to our backend, which properly returns the paginated result. The `queryParams` declaration on the bottom half makes it so our route knows to reload when those params change.

### Update our Controller to declare our query params

```javascript
// frontend/app/mixins/pagination-controller.js
import Ember from 'ember';
export default Ember.Mixin.create({
  queryParams: ['page', 'size'],
  page: 1,
  size: 20
});
```

```javascript
// frontend/app/controllers/posts.js
import Ember from 'ember';
import PaginationControllerMixin from '../mixins/pagination-controller';
export default Ember.Controller.extend(PaginationControllerMixin, {});
```

To make our pagination as re-useable as possible across different routes, we've got a `PaginationControllerMixin`. We add that to our `PostsController` and we've properly wired up Ember to insert our query params into the URL.

### Add our pagination links to the UI

Lastly, we've got to give the user some links to page through our Posts. We'll do that in the form of a simple component named `paginator-links`:


```javascript
// frontend/app/components/paginator-links.js
import Ember from 'ember';
const { Component, computed } = Ember;

// Usage: {{paginator-links pagination=model.meta.pagination routeName="admin.submissions"}}
export default Ember.Component.extend({
  pagination: null, // Object of form:  { links: { first: { number: 1, size: 20 }, next: {...}, ... }, pageCount: 3 }
  routeName: null, // The route to link to

  pageNumbers: computed('pagination.pageCount', function() {
    const result = Ember.A();
    const pageCount = this.get('pagination.pageCount');

    // Using for-loops in 2017 feels weak. Wish I had Ruby's `Integer#times` :(
    for (let i = 1; i <= pageCount; i++) {
      result.pushObject(i);
    }
    return result;
  })
});

```

```handlebars
{{!-- frontend/app/templates/components/paginator-links.hbs --}}
{{#if (gt pagination.pageCount '1')}}
  {{#each-in pagination.links as |key value|}}
    {{link-to key routeName (query-params page=value.number)}}
  {{/each-in}}

  <br>

  {{#each pageNumbers as |number|}}
    {{#if (eq number pagination.links.self.number) }}
      {{ number }}
    {{else}}
      {{link-to number routeName (query-params page=number)}}
    {{/if}}
  {{/each}}
{{/if}}
```

```hbs
{{!-- frontend/app/templates/posts.hbs --}}
{{your-component-to-render-posts model=model}}
{{paginator-links pagination=model.meta.pagination routeName="posts"}}
```

Here we've got our component getting passed the `meta.pagination` object that our `PaginationSerializer` created as well as the name of the route that we're paginating over. Our component takes these in and spits out links to allow the user to paginate over.

By the way, this is BYOCSS. The `paginator-links` doesn't look pretty by any means, but it's functional.

## Wrapping Up

Using the above, we've got some reusable pagination goodness going on. Our `jsonapi-resources` backend properly speaks pagination, our Ember App has a reusable set of Mixins to support pagination, and we've got a handy Component to render the needed links for paginating. This pattern gives us a pretty straight forward way to paginate and I know I walked away from it happy.

Have an improvement or noticed something I missed? Let me know in the comments!
