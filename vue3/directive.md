```
<template>
  <div class="tab" @click="handleTablClick($event)">
    <button data-index="0" :class="['tab-btn', { active: curIndex === 0 }]">
      选项一
    </button>
    <button data-index="1" :class="['tab-btn', { active: curIndex === 1 }]">
      选项二
    </button>
    <button data-index="2" :class="['tab-btn', { active: curIndex === 2 }]">
      选项三
    </button>
  </div>
</template>

<script>
import { ref } from 'vue';

export default {
  name: 'Directive',
  setup() {
    const curIndex = ref(0);
    const handleTablClick = (e) => {
      const tar = e.target;
      const className = tar.className;

      if (className === 'tab-btn') {
        const index = parseInt(tar.dataset.index);
        curIndex.value = index;
      }
    };

    return {
      handleTablClick,
      curIndex,
    };
  },
};
</script>

<style>
.tab-btn.active {
  background-color: green;
  color: white;
}
</style>

```
使用directive 改进

https://stackblitz.com/edit/vue-eywdhd?file=src%2Fcomponents%2FDirective.vue,src%2Fdirective%2Findex.js,src%2Fdirective%2Ftab.js,src%2FApp.vue
tab.js
```
export default {
  mounted(el, binding) {
    const { className, activeClass, curIndex } = binding.value;
    const items = el.getElementsByClassName(className);

    console.log('123', items);
    items[curIndex].className += ` ${activeClass}`;
  },
  updated(el, binding) {
    const oldCurIndex = binding.oldValue.curIndex;
    const { className, activeClass, curIndex } = binding.value;

    const items = el.getElementsByClassName(className);
    items[oldCurIndex].className = className;
    items[curIndex].className += ` ${activeClass}`;
  },
};

```

Directive.vue
```
<template>
  <div
    class="tab"
    @click="handleTablClick($event)"
    v-tab="{
      className: 'tab-btn',
      activeClass: 'active',
      curIndex,
    }"
  >
    <button data-index="0" class="tab-btn">选项一</button>
    <button data-index="1" class="tab-btn">选项二</button>
    <button data-index="2" class="tab-btn">选项三</button>
  </div>
</template>

<script>
import { ref } from 'vue';
import { tab } from '../directive';

export default {
  name: 'Directive',
  directives: {
    tab,
  },
  setup() {
    const curIndex = ref(0);
    const handleTablClick = (e) => {
      const tar = e.target;
      const className = tar.className;

      if (className === 'tab-btn') {
        const index = parseInt(tar.dataset.index);
        curIndex.value = index;
      }
    };

    return {
      handleTablClick,
      curIndex,
    };
  },
};
</script>

<style>
.tab-btn.active {
  background-color: green;
  color: white;
}
</style>

```