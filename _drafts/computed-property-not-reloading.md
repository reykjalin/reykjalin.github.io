---
layout: post
title: Computed property does not reload in Vue.js
---

## The problem  
When using a computed property to access a [Vuex](#) data store in a [Vue.js](#) app, sometimes the property will seemingly not update.
This is because Vue [can not properly detect property addition or deletion](https://terryz.github.io/vue/#/suggest/demo) at the root level for already created object instances.
Basically, this means that if you have a [Vuex](#) state that's an object, and you update the state at the root level, it won't trigger Vue's reactivity, and your data won't update.

For example, the following code will not trigger any updates when the state is changed via `this.$store.dispatch('setData', key, value)`.
The reason being that `state.data[key]` is a root level property of the `state.data` array.
```js
const state = {
  data: []
}

const mutations = {
  setData(state, key, value) {
    // Not reactive
    state.data[key] = value;
  }
}
```

This applies to objects as well!
```js
const state = {
  data: {}
}

const mutations = {
  setData(state, value) {
    // Not reactive
    state.data.property = value;
  }
}
```

Vue does provide methods to mitigate this, but I could never get those to work.
Both `Vue.set(object, key, value)` and `Object.assign()` - the recommended way to handle this in the [Vue.js documentation](#) - never worked.
My [Vuex](#) state is managed in JS files separate from the Vue files, and I have no easy access to the main Vue object, and that's my best guess as to why this never worked.
I'm fairly certain this would've worked if my [Vuex](#) store was managed in the Vue file itself.

This was extremely frustrating, and took a lot of trial and error to figure out, especially because I found no other workarounds online.
Only what is mentioned in the Vue documentation.

I did _finally_ figure out a solution though! And a super simple one at that!

## The solution  
I found that the best way to ensure the reactivity for an array (like the former example above) is to simply call `splice` on the array, _without actually modifying it_.
By calling `splice(0, 0)` on the array, you don't modify any of the data in the array, _and_ it triggers the update in Vue, as is documented [here](https://vuejs.org/v2/guide/list.html#Mutation-Methods).
```js
const state = {
  data: []
}

const mutations = {
  setData(state, key, value) {
    state.data[key] = value;
    // Ensure reactivity
    state.data.splice(0, 0);
  }
}
```

In the case of objects, it was only slightly trickier.
Simply reset the object, and then assign it the updated data.
```js
const state = {
  data: {}
}

const actions = {
  loadData({ commit }) {
    // Ensure reactivity by resetting the object
    commit('setData', {});
    /* ... */
    /* Retrieve/process new data */
    /* ... */
    commit('setData', updated);
  }
}

const mutations = {
  setData(state, value) {
    state.data = value
  }
}
```

## The story