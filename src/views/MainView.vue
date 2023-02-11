<template>
  <VirtualList
    v-slot="slotProps"
    :estimatedItemSize="100"
    :listData="listData"
  >
    <ListItem :item="slotProps.item" />
  </VirtualList>
</template>

<script>
import VirtualList from '@/components/VirtualList'
import ListItem from '@/components/ListItem'

import { faker } from '@faker-js/faker'

export default {
  name: 'VirtualScroll',
  components: {
    VirtualList,
    ListItem
  },
  data() {
    return {
      listData: [],
    }
  },
  created() {
    this.generateDataList()
  },
  methods: {
    generateDataList() {
      for (let i = 0; i < 1000; i++) {
        this.listData.push({
          id: i,
          value: `${faker.lorem.sentences()}`
        })
      }
    },
  },
}
</script>
