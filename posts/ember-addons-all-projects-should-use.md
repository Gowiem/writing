# Ember Addons all projects should use

## Intro

I've jumped back into the Ember world after a long hiatus for a client project recently and I'm really enjoying being back. Ember draws many similarities with Rails, has a fantastic community of passionate people, and the folks behind the framework are the open source maintainers I've come to respect a ton. One of the greatest things I've found since coming back to Ember is the staggering number of [Ember addon](https://www.emberaddons.com/) which boasts over 4k npm packages. Back in 2015 I [wrote my own small addon](https://github.com/Gowiem/ember-sliding-tab-bar) and while the number of addons was big back then it was almost a 1/4th of the size it is today. That is damn cool in my opinion and when I started getting back into the Ember dev environment I found myself finding a number of new awesome addons. This post does a short intro for a few of my personal favorites that I'll be sure to be using on any of my projects in the future.

tl;dr: I'm going to give a short intro to these Ember Addons and show why they're awesome:

1. [ember-decorators](https://github.com/ember-decorators/ember-decorators) - A collection of *awesome* Ember Specific ES7 Decorators.
2. [ember-cli-prop-types](https://github.com/crystal-ball/ember-cli-prop-types) - A library to add *awesome* runtime prop-type checks to Components.
3. [ember-awesome-macros](https://github.com/kellyselden/ember-awesome-macros) - A collection of *awesome* (literally in the name) computed macros.
4. [ember-truth-helpers](https://github.com/jmurphyau/ember-truth-helpers) - A collection of *awesome* HTMLBars template helpers to avoid simple computed properties only used in templates.


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

If you've never written any React code, it includes a cool way to do runtime property type checking by declaring the property names and types in the component definition and then doing assertions against those props in development mode. This helps to catch bugs early in development and enforces a contract on component consumers. Though I never used React heavily, this is one the features of that framework that I thoroughly enjoyed. You can make my mess of JavaScript and HTML a little less dynamic? Yes, please. Sign me up.

Anyway, some smart folks took the [React prop-types library](https://github.com/facebook/prop-types) and used some glue and duct tape to make it usable on Ember Components, which is great thinking. Here is an example from one of my own projects:

```javaScript
// array-input.js
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

This also serves as some great documentation as well because now our component has a written contract in the code, which is something that I always did via class comments. So less comments, more type checking. Good stuff.

## [Ember Awesome Macros](https://github.com/kellyselden/ember-awesome-macros)

TODO

## [Ember Truth Helpers](https://github.com/jmurphyau/ember-truth-helpers)

## Wrapping Up

TODO
