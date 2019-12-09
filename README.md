# system-allocation-solution
非正式的开发代码，在此先实验一种设计方式。

整个功能封装一个独立模块，在模块中提供 index 输出一个标准的组件。供使用场景调用，或者定义在 routers。

技术栈为：Vue + ElementUI。



## 设计思路

模块内主要构成：

- components
- controller
- declars
- model
- resolvers
- utils


### declare

整个功能的核心是以模型驱动（数据模型），将数据进行抽象。一个数据模型由特定的属性和方法组成，以及模型之间存在的关系。在实现模型前需要先定义 interface，它的主要好处：一是为了确定整体的设计思路，二是在编码时按照设计规则进行实现，三维护时快速确定模型关系。

如何模块有几个子模块组成，则可单独定义，然后在 declare/index 中进行 export * from '/xxx'。

下面简单先定义几个：

```ts
// 整个数据结构
export interface IParamCriteria {
  params: IParamItemCriteria[];
  monitorList: IMonitorCriteria[];
  subSystem?: IParamCriteria;
}

// 一个数据模型对应单个参数，在整个模块中所具备的所有属性与方法
export interface IParamItemCriteria {
  name: string;
  alias? string;
  paramType: number;
  unit: string;
  value: IParamCriteriaValue;
  relateMonitor: IMonitorCriteria;
  isDisplay: boolean;
  isEnabled: boolean;
  isAllowModifyAlias: boolean;
  // ...
}

export interface IParamCriteriaValue {
  values?: any[];
  value: any;
  type: string | number;
}

export interface IQueryModelItemResolverOptions<T = any> {
  // 请求前，对查询条件进行扩展
  beforeInitialize?: (
    item: IParamCriteria,
    options: IQueryCollect
  ) => Promise<void>;
  
  // 请求后对数据进行解析方法
  parser?: (item: IQueryCriteria<T>) => any;
  onModelCreated?: (item: QueryItem) => void;
  
  // 对每个参数的样式进行自定义
  formElementRender?: (item: QueryItem) => Element;
  componentRender?: (item: QueryItem) => Element;
  // ...
}

// ....
```

更多的 压缩机、库房等都按照同理定义。


### model

主要组成子模型的实现：ParamItem、ParamGroup，对上面定义的实现。并提供一个主输出 ParamModel，其中数据的请求和存储在此，以及示例所有的子数据模型。

例如，一个子模型：
```ts
export class ParamItem implements IParamCriteriaItem {
    private _disabled = false;
    private _visible = true;
    
    constructor(
        public readonly criteria: IParamCriteria,
        public readonly paramModel: ParamModel,
        private readonly resolverOptions: IParamModelItemResolverOptions = {}
    ) {
    
        // 为 resolvers 提供的自定义生命周期
        this.model.addDisposer(
            this.queryModel.on(LifecycleEvent.onFormCreated, this.onFormCreated),
            this.queryModel.on(LifecycleEvent.onModelCreated, this.onModelCreated)
        );
    }
    
    public get isEnabled() {
        return this.criteria.isEnabled;
    }
    public set isEnabled(isEnabled: boolean) {
        this.criteria.isEnabled = isEnabled;
    }
}
```

ParamModel 文件（Event 是在 utils 中定义个一个事件控制器，主要就是 dispatch 和 on 的实现）
```ts
export class ParamModel extends Event {
  
  // 请求数据，并将每个参数进行扩展绑定
  public async initialize() {
    await request();
    
    // ...
    
    // 扩展每个参数
    this.paramList.forEach(param => {
      this.applySpecialResolvers(param);
    })
  }

  private applySpecialResolvers(criteria: IParamCriteria): IParamModelItemResolverOptions {
    if (criteria.paramType === ParamType.Switch) {
      return createSwitchTypeParamResolver();
    }
    return;
  }
}
```

模型本身是纯数据的操作，不涉及任何UI。

> 在 model 中，数据的不同流程，就开放一些周期和钩子。供 resolvers 中进行定制化的业务扩展，而不会污染模型定义，再然后在 model 中进行引入。



### components

对 UI 组件的二次封装，在此引入基础 UI（如Element、AntDesign）。这里主要分三类：item、layout、index。在 index 中提供组件唯一出口，却引入和实例 controller ，为 component 提供数据和方法。

需要在 index 引入上下文功能（private/inject），确保任何深度的子组件都能够使用 controller 。



### constroller

这是为 component 和 model 的关联部分，针对不同的界面具体的交互所具体的数据模型操作的定义。例如保存，则定义一个 save 方法，并调用model中的一些列动作。



### constant

模块内常量定义此处。

下面是模块内的生命周期：
```
export enum LifecycleEvent {
    onFormCreated = 'onFormCreated',
    onModelCreated = 'onModelCreated'
}
```


### resolvers

对特定的业务场景进行扩展，例如：开关类型参数的设置按钮，在一些场景下会 disabled。

定义 switchParamResolver 文件：
```
export function createSwitchTypeParamResolver(): IQueryModelItemResolverOptions {
  return {
    formElementRender: (item: IParamCriteria, rowRef: ElementRowRef) => {
      // 判断参数并进行修改 ...
    }
  }
}
```
