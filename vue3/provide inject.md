# options Provide / Inject
通常使用props从父组件传递数据到子组件，如果组件层级非常的深，props要一层一层的传递。
提供了provide和Inject,父级使用 provide,子集使用 inject  
```
export default {
    name: 'App',
    components: {
        Father,
    },
    provide() {
        return {
            name: 'zhangsan',
            age: this.age
        }
    }    
    data() {
        return {
            age: 11 // 如果要使用this.age 就要转换provide为函数
        }
    }
}

export default {
    name: 'Father',
    components: {
        Child
    }
}

export default {
    name: 'Child',
    inject: ['name', 'age']
}
```
中间的father就不用传递。但是这个age不是响应式的，修改了之后不会传递到子组件。需要修改

# composition
```
import { provide } from 'vue';

export default {
    name: 'App',
    components: {
        Father,
    },
    setup() {
        provide('name', 'zhangsan');
        provide('age', '26');
        provide('hobby', 'travel');
        provide('career', 'Programmer');
    }
}

export default {
    name: 'Father',
    components: {
        Child
    }
}

import { inject } from 'vue';
export default {
    name: 'Child',
    setup() {
        const name = inject('name', 'noname'),
              age = inject('age', 0),
              hobby = inject('hobby', 'loading...'),
              career = inject('career', 'loading...');

        return {
            name,
            age,
            hobby,
            career
        }
    }
}
```

为Provid增加响应式
```
import { provide, ref } from 'vue';

export default {
    name: 'App',
    components: {
        Father,
    },
    setup() {
        const name = ref('zhangesan');
        provide('name', name);

        const changeName = () => {
            name.value = 'lisi';
        }

        return {
            changeName
        }
    }
}

import { inject } from 'vue';
export default {
    name: 'Child',
    setup() {
        const name = inject('name', 'noname');

        return {
            name,
        }
    }
}
```
如果子组件要修改provide的值，可以在父组件Provide一个方法出来，子组件调用这个方法。(与Angular类似)
```
import { provide, ref } from 'vue';

export default {
    name: 'App',
    components: {
        Father,
    },
    setup() {
        const name = ref('zhangesan');
        

        const changeName = (name) => {
            name.value = name;
        }
        provide('name', name);
        provide('changeName', changeName);
    }
}

import { inject } from 'vue';
export default {
    name: 'Child',
    setup() {
        const name = inject('name', 'noname');
        const changeName = inject('changeName');
        return {
            name,
            changeName
        }
    }
}
```
