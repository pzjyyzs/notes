**ORM 是 Object Relational Mapping，对象关系映射。也就是说把关系型数据库的表映射成面向对象的 class，表的字段映射成对象的属性映射，表与表的关联映射成属性的关联。**

TypeORM 就是一个流行的 ORM 框架

创建项目
```
npx typeorm@latest init --name typeorm-all-feature --database mysql
```
data-source.ts 然后改下用户名密码数据库，把连接 msyql 的驱动包改为 mysql2
```
import "reflect-metadata"

import { DataSource } from "typeorm"

import { User } from "./entity/User"

import { Aaa } from "./entity/Aaa"

  

export const AppDataSource = new DataSource({

type: "mysql",

host: "localhost",

port: 3306,

username: "root",

password: "test123",

database: "practice",

synchronize: true,

logging: false,

entities: [User, Aaa],

migrations: [],

subscribers: [],

connectorPackage: 'mysql2',

extra: {

authPlugin: 'sha256_password'

}

})
```
安装mysql2
```
npm install --save mysql2
```
type 是数据库的类型，因为 TypeORM 不只支持 MySQL 还支持 postgres、oracle、sqllite 等数据库。

host、port 是指定数据库服务器的主机和端口号。

user、password 是登录数据库的用户名和密码。

database 是要指定操作的 database，因为 mysql 是可以有多个 database 或者叫 schema 的。

entities 是指定有哪些和数据库的表对应的 Entity。也可以是``````
```
entities: ['./**/entity/*.ts']
```
migrations 是修改表结构之类的 sql，暂时用不到，就不展开了。
subscribers 是一些 Entity 生命周期的订阅者，比如 insert、update、remove 前后，可以加入一些逻辑。
poolSize 是指定数据库连接池中连接的最大数量。
connectorPackage 是指定用什么驱动包。
extra 是额外发送给驱动包的一些选项。
这些配置都保存在 DataSource 里。
DataSource 会根据你传入的连接配置、驱动包，来创建数据库连接，并且如果制定了 synchronize 的话，会同步创建表。
创建表的依据就是 Entity：
```
import { Column, Entity, PrimaryColumn, PrimaryGeneratedColumn } from "typeorm";

  

@Entity({

name: 't_aaa'

})

export class Aaa {

@PrimaryGeneratedColumn({

comment: 'this is id'

})

id: number;

  

@Column({

name: 'a_aa',

type: 'text',

comment: '这是 aaa'

})

aaa: string;

  

@Column({

unique: true,

nullable: false,

length: 10,

type: 'varchar',

default: 'bbb'

})

bbb: string;

  

@Column({

type: 'double'

})

ccc: number;

  

}
```
我们新增了一个 Entity Aaa。
@Entity 指定它是一个 Entity，name 指定表名为 t_aaa。
@PrimaryGeneratedColumn 指定它是一个自增的主键，通过 comment 指定注释。
@Column 映射属性和字段的对应关系。
通过 name 指定字段名，type 指定映射的类型，length 指定长度，default 指定默认值。
nullable 设置 NOT NULL 约束，unique 设置 UNIQUE 唯一索引。

执行npm run start 就可以生成

然后是增删改查
```
import { AppDataSource } from "./data-source"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {

    const user = new User()
    user.firstName = "aaa"
    user.lastName = "bbb"
    user.age = 25

    await AppDataSource.manager.save(user)

}).catch(error => console.log(error))

```
在User表里新增一个user,增加一个id,就是修改这个数据

批量插入
```
import { AppDataSource } from "./data-source"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {

    await AppDataSource.manager.save(User, [
        { firstName: 'ccc', lastName: 'ccc', age: 21},
        { firstName: 'ddd', lastName: 'ddd', age: 22},
        { firstName: 'eee', lastName: 'eee', age: 23}
    ]);


}).catch(error => console.log(error))

```
批量修改就是加一个id
```
import { AppDataSource } from "./data-source"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {

    await AppDataSource.manager.save(User, [
        { id: 2 ,firstName: 'ccc111', lastName: 'ccc', age: 21},
        { id: 3 ,firstName: 'ddd222', lastName: 'ddd', age: 22},
        { id: 4, firstName: 'eee333', lastName: 'eee', age: 23}
    ]);

}).catch(error => console.log(error))
```

删除和批量删除
```
import { AppDataSource } from "./data-source"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {

    await AppDataSource.manager.delete(User, 1);
    await AppDataSource.manager.delete(User, [2,3]);

}).catch(error => console.log(error))
```
也可以remove
```
import { AppDataSource } from "./data-source"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {

    const user = new User();
    user.id = 1;

    await AppDataSource.manager.remove(User, user);

}).catch(error => console.log(error))

```
delete和remove的区分，delete直接传入id，remove传入entity对象

查询 find
```
import { AppDataSource } from "./data-source"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {
    const users = await AppDataSource.manager.find(User);
    console.log(users);
    
}).catch(error => console.log(error))
```
findBy
```
import { In } from "typeorm";
import { AppDataSource } from "./data-source"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {
    const users = await AppDataSource.manager.findBy(User, {
        age: 23
    });
    console.log(users);
   
}).catch(error => console.log(error))
```

findAndCount
```
import { AppDataSource } from "./data-source"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {
    const [users, count] = await AppDataSource.manager.findAndCount(User);
    console.log(users, count);

}).catch(error => console.log(error))

```

count可以指定条件
```
import { AppDataSource } from "./data-source"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {
    const [users, count] = await AppDataSource.manager.findAndCountBy(User, {
        age: 23
    })
    console.log(users, count);

}).catch(error => console.log(error))
```

findOne
```
import { AppDataSource } from "./data-source"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {
    const user = await AppDataSource.manager.findOne(User, {
        select: {
            firstName: true,
            age: true
        },
        where: {
            id: 4
        },
        order: {
            age: 'ASC'
        }
    });
    console.log(user);

}).catch(error => console.log(error))
```
指定查询的 where 条件是 id 为 4 ，指定 select 的列为 firstName 和 age，然后 order 指定根据 age 升序排列。

你还可以用 query 方法直接执行 sql 语句：
```
import { AppDataSource } from "./data-source"

AppDataSource.initialize().then(async () => {

    const users = await AppDataSource.manager.query('select * from user where age in(?, ?)', [21, 22]);
    console.log(users);

}).catch(error => console.log(error))
```
但复杂 sql 语句不会直接写，而是会用 query builder：
```
const queryBuilder = await AppDataSource.manager.createQueryBuilder();

const user = await queryBuilder.select("user")
    .from(User, "user")
    .where("user.age = :age", { age: 21 })
    .getOne();

console.log(user);
```

 transaction 
 ```
 await AppDataSource.manager.transaction(async manager => {
    await manager.save(User, {
        id: 4,
        firstName: 'eee',
        lastName: 'eee',
        age: 20
    });
});

```

可以先调用 getRepository 传入 Entity，拿到专门处理这个 Entity 的增删改查的类，再调用这些方法：
```
AppDataSource.manager.getRepository(User).find
```

## 一对一映射

```
@Entity({
	name: 'id_card'
})
export class IdCard {
	@PrimaryGeneratedColumn()
	id: number

	@Column({
		length: 50,
		comment: '身份证号'
	})
	cardName: string

	@JoinColumn()
	@OneToOne(() => User)
	user: User
}
```
修改在级联关系
```
@JoinColumn()
@OneToOne(() => User, {
	onDelete: '',
	onUpdate: '' // CASCADE,DEFAULT,NO ACTION,RESTRICT,SET NULL
})
user: User
```

```
import { AppDataSource } from "./data-source"
import { IdCard } from "./entity/IdCard"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {

    const user = new User();
    user.firstName = 'guang';
    user.lastName = 'guang';
    user.age = 20;
    
    const idCard = new IdCard();
    idCard.cardName = '1111111';
    idCard.user = user;
    
    await AppDataSource.manager.save(user);
    await AppDataSource.manager.save(idCard);

}).catch(error => console.log(error))
```

还要分别保存 user 和 idCard，能不能自动按照关联关系来保存呢？
可以的，在 @OneToOne 那里指定 cascade 为 true：
```
@JoinColumn()
@OneToOne(() => User, {
	cascade: true,
	onDelete: '',
	onUpdate: '' // CASCADE,DEFAULT,NO ACTION,RESTRICT,SET NULL
})
user: User
```

```
import { AppDataSource } from "./data-source"
import { IdCard } from "./entity/IdCard"
import { User } from "./entity/User"

AppDataSource.initialize().then(async () => {

    const user = new User();
    user.firstName = 'guang';
    user.lastName = 'guang';
    user.age = 20;
    
    const idCard = new IdCard();
    idCard.cardName = '1111111';
    idCard.user = user;
    
    //await AppDataSource.manager.save(user);
    await AppDataSource.manager.save(idCard);

}).catch(error => console.log(error))
```

关联查询
```
const ics = await AppDataSource.manager.find(IdCard, {
    relations: {
        user: true
    }
});
console.log(ics);
```

## 一对多

```
@Entity()
export class Employee {
	@PrimaryGeneratedColumn()
	id: number;

	@Column({
		length: 50
	})
	name: string;

	@ManyToOne(() => Department, {
		cascade: true
	})
	department: Department;
}
```
```
export class Department {
	@PrimaryGeneratedColumn()
	id: number;

	@Column({
		length: 50
	})
	name: string;

	@OneToMany(() => Employee, (employee) => employee.department, {
		cascade: true // onetoMany 和 ManytoOne 设置一个cascade 就可以了
	})
	employees: Employee[];
}
```
一对一的时候我们还通过 @JoinColumn 来指定外键列，为什么一对多就不需要了呢？
因为一对多的关系只可能是在多的那一方保存外键呀！
所以并不需要 @JoinColumn。

## 多对多
```
@ManyToMany(() => Tag)
tags: Tag[]
```