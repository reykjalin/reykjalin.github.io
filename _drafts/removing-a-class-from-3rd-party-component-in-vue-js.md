---
layout: post
title: Removing a class from a 3rd party componenet in Vue.js
---

## The problem  
You have a specific style for all your own components and views, e.g. by using Bootstrap, and then you import another component.
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