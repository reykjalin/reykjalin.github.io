---
layout: post
title: Removing a class from a 3rd party componenet in Vue.js
---

## The problem  
You have a specific style for all your own components and views, e.g. by using something like [Bootstrap](https://getbootstrap.com), and then you import another component.
All of a sudden you have inconsistent styling in you web app, because everything _except_ the third party component is using your desired styling.
So how do you change this?

## The solution  
1. Add a ref to your component
2. In the `mounted()` function of your view:
    1. Find that component using the ref
    2. Remove all classes from the component
3. Add your style to the third party component

```vue
<template>
  <div>
    <third-party-component class="mystyle" ref="thirdparty" />
  </div>
</template>

<script>
  mounted() {
    this.$refs.thirdparty.$el.classlist = [];
  }
</script>
```

## The story  
While working on adding auto-complete to a Vue.js application using [v-suggest](https://terryz.github.io/vue/#/suggest/demo) I encountered this issue and found it extremely frustrating that the third party component had a different style from everything else on the website.

When I tried to set the style using the `class` attribute of the component, it didn't work.
The reason that happens is because the `class` attribute is applied to the inner element of the component, while the outer element surrounding the element - still inside the component - has some custom styling.

After searching for a while for a workaround for this I found a way to simply get the html element for the component, and removing the class.
That in turn removes the styling.

If you only want to remove a specific class instead of resetting the whole list, I suppose you can also just remove specific elements.