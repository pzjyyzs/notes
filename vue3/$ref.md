# Template refs
有时需要直接访问子组件和HTML Element
```
<template>
    <div>
        <Father ref="father" />
        <button @click="changeName"></button>  
    </div>
</template>
<script>
export default {
    name: 'App',
    components: {
        Father,
    },
    methods: {
        changeName() {
            this.$refs.father.changeName()
        }
    }
}
</script>

<template>
    <h1>{{ name }}</h1>
</template>
<sctipt>
export default {
    name: 'Father',
    components: {
        Child
    },
    data() {
        return {
            name: 'zhangsan'
        }
    },
    methods: {
        changeName() {
            this.name = 'lisi';
        }
    }
}
</script>
```
放在组件上,就是组件的实例；HTML就是HTML元素。
不能在 html和computed上直接访问 $refs

# componsition api
```
<template>
    <div>
        <div ref="child">张三</div>
        <button @click="changeName">改变名字</button>  
    </div>
</template>
<script>
import { ref } from 'vue';
 
export default {
    name: 'Child',
    setup() {
        const child = ref(null);

        const changeName = () => {
            console.log(child.value);
            child.value.innerText = '李四';
        }

        return {
            child,
            changeName
        }
    }
}
</script>
```