<template>
  <p><span style="color:red">{{ item.id }}</span>&nbsp;{{ item.value }}</p>
</template>

<script>
export default {
  name: 'ListItem',
  props: {
    item: {
      type: Object,
      default: () => {},
    },
  }
}
</script>
