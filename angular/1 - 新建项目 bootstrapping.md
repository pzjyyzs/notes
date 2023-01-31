使用```npm install -g @angular/cli``` 安装angular-cli
使用```ng new <project-name> ``` 新建项目 可以添加 ```--package-manager	pnpm``` 指定包管理器

由main.ts的文件开始引入AppModule。
Module 模块 把一些关系比较紧密的组件，管道，指令等组织到一起，它们组织在一起实现了某个功能。例如内置模块BrowserModule，AppRoutingModule等。在同一个模块内的组件，指令，管道可以相互引用。不同模块的需要导入和导出。


```
@NgModule({
    declarations: [AppComponent],
    imports: [BrowserModule],
    bootstrap: [AppComponent]
})
export class AppModule {}
```

AppModule 启动的顶层module。@NgModule 是一个类装饰器，装饰器是为添加元数据或增加进一步的信息。使用了@NgModule才会被认为是一个Angular的模块，没有则是一个普通的class。
declarations 表示这个模块中定义了哪些组件，指令，管道
imports 我们如果需要使用不在这个模块的组件 需要引入定义了这个组件的模块
exports 其他模块需要使用这个模块的组件，需要把这个组件给导出去
providers 为模块提供服务，服务包括外部请求，一些可复用的逻辑。这里注入了服务，允许我们在组件内使用Angular的依赖注入系统去注册类
bootstrap 会根据这个数组的组件创建组件树的根组件，一般位于 根模块中并且只有一个。

在index.html 页面中使用了一个 标签 app-root, 在AppComponent中selector 指定的是 如果想渲染这个组件，该怎么使用它。详细的内容后面写。
```
@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.less']
})
export class AppComponent {}
```
