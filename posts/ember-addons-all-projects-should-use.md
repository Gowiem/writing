# Ember Addons all projects should use

## Intro

I've jumped back into the Ember world after a long hiatus for a client project recently and I'm really enjoying being back. Ember draws many similarities with Rails, has a fantastic community of passionate people, and the folks behind the framework are the open source maintainers I've come to respect the most. One of the greatest things I've found since coming back to Ember is the staggering number of [Ember addons](https://www.emberaddons.com/) which boasts over 4k npm packages. Back in 2015 I [wrote my own small addon](https://github.com/Gowiem/ember-sliding-tab-bar) and while the number of addons was big back then it was almost a 1/4th of the size it is today. That is damn cool in my opinion and when I started getting back into the Ember dev environment I found myself finding a number of new awesome addons. This post does a short intro for a few of my personal favorites that I'll be sure to be using on any of my projects in the future.

tl;dr: I'm going to give a short intro to these Ember Addons and show why they're awesome:

1. [ember-decorators](https://github.com/ember-decorators/ember-decorators) - A collection of *awesome* Ember Specific ES7 Decorators.
2. [ember-cli-prop-types](https://github.com/crystal-ball/ember-cli-prop-types) - A library to add *awesome* runtime prop-type checks to Components.
3. [ember-awesome-macros](https://github.com/kellyselden/ember-awesome-macros) - A collection of *awesome* (literally in the name) computed macros.
4. [ember-truth-helpers](https://github.com/jmurphyau/ember-truth-helpers) - A collection of *awesome* HTMLBars template helpers to avoid simple computed properties only used in templates.
5. [ember-route-action-helper](https://github.com/DockYard/ember-route-action-helper) - A library to add a *awesome* `route-action` helper for bubbling actions to the route without a controller.


## [Ember Decorators]((https://github.com/ember-decorators/ember-decorators)

The [ES7 JavaScript Decorators feature](https://github.com/wycats/javascript-decorators) (still in proposal, but it's gonna happen) is sweeeeet. Decorators add a great way to bundle reusable property, function, and class changes into a simple python-style annotation. ember-decorators take that awesome idea and apply it to some of the tedious bits of Ember's usage. It's fairly easy to grasp when you look at an example:

```javascript
// Before ember-decorators
import Ember from 'ember';

export default Ember.Component.extend({
  foo: Ember.inject.service(),

  bar: Ember.computed('someKey', 'otherKey', function() {
    var someKey = this.get('someKey');
    var otherKey = this.get('otherKey');

    return `${someKey} - ${otherKey}`;
  }),

  actions: {
    handleClick() {
      // do stuff
    }
  }
})

// After ember-decorators
import Component from '@ember/component';
import { action, computed } from 'ember-decorators/object';
import { service } from 'ember-decorators/service';

export default class ExampleComponent extends Component {
  @service foo

  @computed('someKey', 'otherKey')
  bar(someKey, otherKey) {
    return `${someKey} - ${otherKey}`;
  }

  @action
  handleClick() {
    // do stuff
  }
}
```

How cool is it that you get passed the values for the properties that your writing a computed prop around? I love that!

## [Ember-CLI Prop Types](https://github.com/crystal-ball/ember-cli-prop-types)

If you've never written any React code, React includes a cool way to do runtime property type checking by declaring the property names and types in the component definition and then doing assertions against those props in development mode. This helps to catch bugs early in development and enforces a contract on component consumers. Though I never used React heavily, this is one the features of that framework that I thoroughly enjoyed. You can make my mess of JavaScript and HTML a little less dynamic? Yes, please. Sign me up.

Anyway, some smart folks took the [React prop-types library](https://github.com/facebook/prop-types) and used some glue and duct tape to make it usable on Ember Components, which is great thinking. Here is an example from one of my own projects:

```javaScript
// components/array-input.js
import Ember from 'ember';
import PropTypes from 'prop-types';
import Submission from '../models/submission';
const { instanceOf, number, string, func, bool } = PropTypes;

export default Ember.Component.extend({

  // Component logic removed for brevity

  propTypes: {
    model: instanceOf(Submission).isRequired,
    idx: number.isRequired,
    value: string,
    flushValue: func.isRequired,
    disabled: bool,
  }
});
```

This also serves as some great documentation as well because now our component has a written contract in the code, which is something that I always did via class comments. Less comments, more type checking. Good stuff.

## [Ember Awesome Macros](https://github.com/kellyselden/ember-awesome-macros)

You know all the good stuff in [`Ember.computed.*`](https://www.emberjs.com/api/ember/2.14.1/namespaces/Ember.computed)? If you don't, [you should](https://dockyard.com/blog/2014/06/27/ember-macros-for-DRY-and-testable-code). Think of ember-awesome-macros (EAM) as `Ember.computed`, but extended and composable. A ton of the trivial operations you would normally write a full computed property for can easily be handled by EAM. Here are some good examples:

```javascript
// models/submission.js
import DS from 'ember-data';
import Ember from 'ember';
import Enum from 'ember-enum/utils/enum';
import Timestamps from '../mixins/timestamps';
import array from 'ember-awesome-macros/array';
import or from 'ember-awesome-macros/or';

const { sort, filter } = array;
const { attr, Model, belongsTo, hasMany, PromiseArray } = DS;
const states = { 'draft': 'Draft', 'submitted': 'Submitted', 'in_review': 'In Review', 'review_required': 'Review Required', 'completed': 'Completed' };

export default Model.extend(Timestamps, {
  state: Enum({
    options: Object.keys(states),
    defaultValue: 'draft',
  }),
  // ...
  submissionReviews: hasMany('submission-review'),
  // ...

  // CPs
  isSubmittedOrCompleted: or('state.isSubmitted', 'state.isCompleted'),
  savedSubmissionReviews: sort(filter('submissionReviews.@each.createdAt', (review) => {
    return !review.get('isNew');
  }), ['createdAt:desc']),

  // ...
});
```

The above is a simple example, but it gives you a small idea. The brevity of these macros is super cool and it saves a good deal in not having to write these abstractions yourself.

## [Ember Truth Helpers](https://github.com/jmurphyau/ember-truth-helpers)

How many times have you written a `Ember.computed.or` or a `Ember.computed.and` because you needed some boolean logic in a Handlebars template? A bunch right? No more! ember-truth-helpers provides a bunch of simple template helpers that will let you skill over all that. Here is some of the logic you can do easily:

```Handlebars
{{if (eq a b)}}
{{if (not-eq a b)}}
{{if (not a)}}
{{if (and a b)}}
{{if (or a b)}}    	
{{if (xor a b)}}
{{if (gt a b)}}
{{if (gte a b)}}
{{if (lt a b)}}
{{if (lte a b)}}
{{if (is-array a)}}
{{if (is-equal a b)}}
```

Avoid one off computed properties, go home happy.

## [Ember Route Action Helper](https://github.com/DockYard/ember-route-action-helper)

A simple one, but a good one. Avoid creating a controller for your base template just to pass actions. Use the `route-action` helper to easily bubble your actions to the current route and keep all of your actions in your components and your routes. Here is an example:

```
// routes/index.js
import Ember from 'ember';
export default Ember.Route.extend({
  // ...

  actions: {
    submit(model) {
      // ...
    }
  }
});

// index.hbs
{{#bs-form model=model onSubmit=(route-action "submit" model) as |f|}}
  {{!-- ... --}}
{{/bs-form}}
```

## Wrapping Up

These are just a handful of some of the great Ember Addons that are out there. One other that I didn't include, but I hear people rave about for complex apps is [ember-concurrency](https://ember-concurrency.com/#/docs/introduction). Definitely worth checking out if you're managing a bunch of async tasks in your Ember app. Share your favoite Ember Addons in the comments below!
