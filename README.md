#### 1. 请简述 React 16 版本中初始渲染的流程
* React元素初始渲染的过程：render阶段和commit阶段
  - render阶段：就是协调层处理阶段，在这个阶段要为每个React元素构建fiber对象，在构建fiber对象的过程中还要创建其DOM对象，并且对fiber对象创建tag属性，就是标注当前fiber对象要对应的DON对象要进行什么操作：是要插入、更新还是删除，这个新构建的fiber对象成为workInProgress fiber树，也理解为待提交的fiber树，当render阶段结束后会保存在fiberRoot对象中，接下来进入commit阶段。
  - commit阶段：要先获取render阶段的成果，就是获取到保存在fiberRoot对象中的workInProgress fiber树，根据fiber对象中的tag属性进行相应的DOM操作。



#### 2. 为什么 React 16 版本中 render 阶段放弃了使用递归
* 解析：在 React 15 的版本中，采用了循环加递归的方式进行了 virtualDOM 的比对，由于递归使用 JavaScript 自身的执行栈，一旦开始就无法停止，直到任务执行完成。如果 VirtualDOM 树的层级比较深，virtualDOM 的比对就会长期占用 JavaScript 主线程，由于 JavaScript 又是单线程的无法同时执行其他任务，所以在比对的过程中无法响应用户操作，无法即时执行元素动画，造成了页面卡顿的现象。
* 在 React 16 的版本中，放弃了 JavaScript 递归的方式进行 virtualDOM 的比对，而是采用循环模拟递归。而且比对的过程是利用浏览器的空闲时间完成的，不会长期占用主线程，这就解决了 virtualDOM 比对造成页面卡顿的问题。


#### 3. 请简述 React 16 版本中 commit 阶段的三个子阶段分别做了什么事情
* before mutation 阶段（执行 DOM 操作前）
  - 调用类组件的getSnapshotBeforeUpdate生命周期函数

* mutation 阶段（执行 DOM 操作）
  - 根据effectTag进行DOM操作

* layout 阶段（执行 DOM 操作后）
  - 调用类组件的生命周期函数和函数组件处理钩子函数

`文件位置: packages/react-reconciler/src/ReactFiberWorkLoop.js`

```react
function commitRootImpl(root, renderPriorityLevel) {
  // 获取待提交 Fiber 对象 rootFiber
  const finishedWork = root.finishedWork;
  // 如果没有任务要执行
  if (finishedWork === null) {
    // 阻止程序继续向下执行
    return null;
  }
  // 重置为默认值
  root.finishedWork = null;
  root.callbackNode = null;
  root.callbackExpirationTime = NoWork;
  root.callbackPriority = NoPriority;
  root.nextKnownPendingLevel = NoWork;
  
  // 获取要执行 DOM 操作的副作用列表
  let firstEffect = finishedWork.firstEffect;

  // true
  if (firstEffect !== null) {
    // commit 第一个子阶段
    nextEffect = firstEffect;
    // 处理类组件的 getSnapShotBeforeUpdate 生命周期函数
    do {
      invokeGuardedCallback(null, commitBeforeMutationEffects, null);
    } while (nextEffect !== null);
    
		// commit 第二个子阶段
    nextEffect = firstEffect;
    do {
      invokeGuardedCallback(null, commitMutationEffects, null, root, renderPriorityLevel);
    } while (nextEffect !== null);
    // 将 workInProgress Fiber 树变成 current Fiber 树
    root.current = finishedWork;
    
		// commit 第三个子阶段
    nextEffect = firstEffect;
    do {
      invokeGuardedCallback(null, commitLayoutEffects, null, root,expirationTime);
    } while (nextEffect !== null);
		
    // 重置 nextEffect
    nextEffect = null;
  }
}
```

###### 第一子阶段

* commitBeforeMutationEffects

`文件位置: packages/react-reconciler/src/ReactFiberWorkLoop.js`

```react
// commit 阶段的第一个子阶段
// 调用类组件的 getSnapshotBeforeUpdate 生命周期函数
function commitBeforeMutationEffects() {
  // 循环 effect 链
  while (nextEffect !== null) {
    // nextEffect 是 effect 链上从 firstEffect 到 lastEffec 的每一个需要commit的 fiber 对象

    // 初始化渲染第一个 nextEffect 为 App 组件
    // effectTag => 3
    const effectTag = nextEffect.effectTag;
    // console.log(effectTag);
    // nextEffect = null;
    // return;

    // 如果 fiber 对象中里有 Snapshot 这个 effectTag 的话
    // Snapshot 和更新有关系 初始化渲染 不执行
    // false
    if ((effectTag & Snapshot) !== NoEffect) {
      // 获取当前 fiber 节点
      const current = nextEffect.alternate;
      // 当 nextEffect 上有 Snapshot 这个 effectTag 时
      // 执行以下方法, 主要是类组件调用 getSnapshotBeforeUpdate 生命周期函数
      commitBeforeMutationEffectOnFiber(current, nextEffect);
    }
    // 更新循环条件
    nextEffect = nextEffect.nextEffect;
  }
}
```

* 5.2.4.2 commitBeforeMutationLifeCycles

`文件位置: packages/react-reconciler/src/ReactFiberCommitWork.js`

```react
function commitBeforeMutationLifeCycles(
  current: Fiber | null,
  finishedWork: Fiber,
): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block: {
      return;
    }
    // 如果该 fiber 类型是 ClassComponent
    case ClassComponent: {
      if (finishedWork.effectTag & Snapshot) {
        if (current !== null) {
          // 旧的 props
          const prevProps = current.memoizedProps;
          // 旧的 state
          const prevState = current.memoizedState;
          // 获取 classComponent 组件的实例对象
          const instance = finishedWork.stateNode;
          // 执行 getSnapshotBeforeUpdate 生命周期函数
          // 在组件更新前捕获一些 DOM 信息
          // 返回自定义的值或 null, 统称为 snapshot
          const snapshot = instance.getSnapshotBeforeUpdate(
            finishedWork.elementType === finishedWork.type
              ? prevProps
              : resolveDefaultProps(finishedWork.type, prevProps),
            prevState,
          );
        }
      }
      return;
    }
    case HostRoot:
    case HostComponent:
    case HostText:
    case HostPortal:
    case IncompleteClassComponent:
      // Nothing to do for these component types
      return;
  }
}
```

###### 第二子阶段

* commitMutationEffects

`文件位置: packages/react-reconciler/src/ReactFiberWorkLoop.js`

```react
// commit 阶段的第二个子阶段
// 根据 effectTag 执行 DOM 操作
function commitMutationEffects(root: FiberRoot, renderPriorityLevel) {
  // 循环 effect 链
  while (nextEffect !== null) {
    // 获取 effectTag
    // 初始渲染第一次循环为 App 组件
    // 即将根组件及内部所有内容一次性添加到页面中
    const effectTag = nextEffect.effectTag;

    // 根据 effectTag 分别处理
    let primaryEffectTag =
      effectTag & (Placement | Update | Deletion | Hydrating);
    // 匹配 effectTag
    // 初始渲染 primaryEffectTag 为 2 匹配到 Placement
    switch (primaryEffectTag) {
      // 针对该节点及子节点进行插入操作
      case Placement: {
        commitPlacement(nextEffect);
        // effectTag 从 3 变为 1
        // 从 effect 标签中清除 "placement" 重置 effectTag 值
        // 以便我们知道在调用诸如componentDidMount之类的任何生命周期之前已将其插入。
        nextEffect.effectTag &= ~Placement;
        break;
      }
      // 插入并更新 DOM
      case PlacementAndUpdate: {
        // 插入
        commitPlacement(nextEffect);
        nextEffect.effectTag &= ~Placement;

        // 更新
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 服务器端渲染
      case Hydrating: {
        nextEffect.effectTag &= ~Hydrating;
        break;
      }
      // 服务器端渲染
      case HydratingAndUpdate: {
        nextEffect.effectTag &= ~Hydrating;

        // Update
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 更新 DOM
      case Update: {
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 删除 DOM
      case Deletion: {
        commitDeletion(root, nextEffect, renderPriorityLevel);
        break;
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```

* 5.2.5.2 commitPlacement

`文件位置: packages/react-reconciler/src/ReactFiberCommitWork.js`

```react
// 挂载 DOM 元素
function commitPlacement(finishedWork: Fiber): void {
  // finishedWork 初始化渲染时为根组件 Fiber 对象
  // 获取非组件父级 Fiber 对象
  // 初始渲染时为 <div id="root"></div>
  const parentFiber = getHostParentFiber(finishedWork);

  // 存储真正的父级 DOM 节点对象
  let parent;
  // 是否为渲染容器
  // 渲染容器和普通react元素的主要区别在于是否需要特殊处理注释节点
  let isContainer;
  // 获取父级 DOM 节点对象
  // 但是初始渲染时 rootFiber 对象中的 stateNode 存储的是 FiberRoot
  const parentStateNode = parentFiber.stateNode;
  // 判断父节点的类型
  // 初始渲染时是 hostRoot 3
  switch (parentFiber.tag) {
    case HostComponent:
      parent = parentStateNode;
      isContainer = false;
      break;
    case HostRoot:
      // 获取真正的 DOM 节点对象
      // <div id="root"></div>
      parent = parentStateNode.containerInfo;
      // 是 container 容器
      isContainer = true;
      break;
    case HostPortal:
      parent = parentStateNode.containerInfo;
      isContainer = true;
      break;
    case FundamentalComponent:
      if (enableFundamentalAPI) {
        parent = parentStateNode.instance;
        isContainer = false;
      }
  }
  // 查看当前节点是否有下一个兄弟节点
  // 有, 执行 insertBefore
  // 没有, 执行 appendChild
  const before = getHostSibling(finishedWork);
	// 渲染容器
  if (isContainer) {
    // 向父节点中追加节点 或者 将子节点插入到 before 节点的前面
    insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
  } else {
    // 非渲染容器
    // 向父节点中追加节点 或者 将子节点插入到 before 节点的前面
    insertOrAppendPlacementNode(finishedWork, before, parent);
  }
}
```

* getHostParentFiber

`文件位置: packages/react-reconciler/src/ReactFiberCommitWork.js`

```react
// 获取 HostRootFiber 对象
function getHostParentFiber(fiber: Fiber): Fiber {
  // 获取当前 Fiber 父级
  let parent = fiber.return;
  // 查看父级是否为 null
  while (parent !== null) {
    // 查看父级是否为 hostRoot
    if (isHostParent(parent)) {
      // 返回
      return parent;
    }
    // 继续向上查找
    parent = parent.return;
  }
}
```
* insertOrAppendPlacementNodeIntoContainer

`文件位置: packages/react-reconciler/src/ReactFiberCommitWork.js`

```react
// 向容器中追加 | 插入到某一个节点的前面
function insertOrAppendPlacementNodeIntoContainer(
  node: Fiber,
  before: ?Instance,
  parent: Container,
): void {
  const {tag} = node;
  // 如果待插入的节点是一个 DOM 元素或者文本的话
  // 比如 组件fiber => false div => true
  const isHost = tag === HostComponent || tag === HostText;

  if (isHost || (enableFundamentalAPI && tag === FundamentalComponent)) {
    // 获取 DOM 节点
    const stateNode = isHost ? node.stateNode : node.stateNode.instance;
    // 如果 before 存在
    if (before) {
      // 插入到 before 前面
      insertInContainerBefore(parent, stateNode, before);
    } else {
      // 追加到父容器中
      appendChildToContainer(parent, stateNode);
    }
  } else {
    // 如果是组件节点, 比如 ClassComponent, 则找它的第一个子节点(DOM 元素)
    // 进行插入操作
    const child = node.child;
    if (child !== null) {
      // 向父级中追加子节点或者将子节点插入到 before 的前面
      insertOrAppendPlacementNodeIntoContainer(child, before, parent);
      // 获取下一个兄弟节点
      let sibling = child.sibling;
      // 如果兄弟节点存在
      while (sibling !== null) {
        // 向父级中追加子节点或者将子节点插入到 before 的前面
        insertOrAppendPlacementNodeIntoContainer(sibling, before, parent);
        // 同步兄弟节点
        sibling = sibling.sibling;
      }
    }
  }
}
```

* insertInContainerBefore

`文件位置: packages/react-dom/src/client/ReactDOMHostConfig.js`

```react
export function insertInContainerBefore(
  container: Container,
  child: Instance | TextInstance,
  beforeChild: Instance | TextInstance | SuspenseInstance,
): void {
  // 如果父容器是注释节点
  if (container.nodeType === COMMENT_NODE) {
    // 找到注释节点的父级节点 因为注释节点没法调用 insertBefore
    (container.parentNode: any).insertBefore(child, beforeChild);
  } else {
    // 将 child 插入到 beforeChild 的前面
    container.insertBefore(child, beforeChild);
  }
}
```

* appendChildToContainer

`文件位置: packages/react-dom/src/client/ReactDOMHostConfig.js`


```react
export function appendChildToContainer(
  container: Container,
  child: Instance | TextInstance,
): void {
  // 监测 container 是否注释节点
  if (container.nodeType === COMMENT_NODE) {
    // 获取父级的父级
    parentNode = (container.parentNode: any);
    // 将子级节点插入到注释节点的前面
    parentNode.insertBefore(child, container);
  } else {
    // 直接将 child 插入到父级中
    parentNode = container;
    parentNode.appendChild(child);
  }
}
```

###### 第三子阶段

* commitLayoutEffects

`文件位置: packages/react-reconciler/src/ReactFiberWorkLoop.js`

```react
// commit 阶段的第三个子阶段
function commitLayoutEffects(
  root: FiberRoot,
  committedExpirationTime: ExpirationTime,
) {
  while (nextEffect !== null) {
    // 此时 effectTag 已经被重置为 1, 表示 DOM 操作已经完成
    const effectTag = nextEffect.effectTag;
    // 调用生命周期函数和钩子函数
    // 前提是类组件中调用了生命周期函数 或者函数组件中调用了 useEffect
    if (effectTag & (Update | Callback)) {
      // 类组件处理生命周期函数
      // 函数组件处理钩子函数
      commitLayoutEffectOnFiber(root, current,nextEffect, committedExpirationTime);
    }
    // 更新循环条件
    nextEffect = nextEffect.nextEffect;
  }
}
```

* commitLifeCycles

`文件位置: packages/react-reconciler/src/ReactFiberCommitWork.js`

```react
function commitLifeCycles(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedExpirationTime: ExpirationTime,
): void {
  switch (finishedWork.tag) {
    case FunctionComponent: {
      // 处理钩子函数
      commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
      return;
    }
    case ClassComponent: {
      // 获取类组件实例对象
      const instance = finishedWork.stateNode;
      // 如果在类组件中存在生命周期函数判断条件就会成立
      if (finishedWork.effectTag & Update) {
        // 初始渲染阶段
        if (current === null) {
          // 调用 componentDidMount 生命周期函数
          instance.componentDidMount();
        } else {
          // 更新阶段
          // 获取旧的 props
          const prevProps = finishedWork.elementType === finishedWork.type
              ? current.memoizedProps
              : resolveDefaultProps(finishedWork.type, current.memoizedProps);
          // 获取旧的 state
          const prevState = current.memoizedState;
          // 调用 componentDidUpdate 生命周期函数
          // instance.__reactInternalSnapshotBeforeUpdate 快照 getSnapShotBeforeUpdate 方法的返回值
          instance.componentDidUpdate(prevProps, prevState, instance.__reactInternalSnapshotBeforeUpdate);
        }
      }
      // 获取任务队列
      const updateQueue = finishedWork.updateQueue;
      // 如果任务队列存在
      if (updateQueue !== null) {
        /**
         * 调用 ReactElement 渲染完成之后的回调函数
         * 即 render 方法的第三个参数
         */
        commitUpdateQueue(finishedWork, updateQueue, instance);
      }
      return;
    }
}
```

* commitUpdateQueue

`文件位置: packages/react-reconciler/src/ReactUpdateQueue.js`

```react
/**
 * 执行渲染完成之后的回调函数
 */
export function commitUpdateQueue<State>(
  finishedWork: Fiber,
  finishedQueue: UpdateQueue<State>,
  instance: any,
): void {
  // effects 为数组, 存储任务对象 (Update 对象)
  // 但前提是在调用 render 方法时传递了回调函数, 就是 render 方法的第三个参数
  const effects = finishedQueue.effects;
  // 重置 finishedQueue.effects 数组
  finishedQueue.effects = null;
  // 如果传递了 render 方法的第三个参数, effect 数组就不会为 null
  if (effects !== null) {
    // 遍历 effect 数组
    for (let i = 0; i < effects.length; i++) {
      // 获取数组中的第 i 个需要执行的 effect
      const effect = effects[i];
      // 获取 callback 回调函数
      const callback = effect.callback;
      // 如果回调函数不为 null
      if (callback !== null) {
        // 清空 effect 中的 callback
        effect.callback = null;
        // 执行回调函数
        callCallback(callback, instance);
      }
    }
  }
}
```

* commitHookEffectListMount

`文件位置: packages/react-reconciler/src/ReactFiberCommitWork.js`

```react
/**
 * useEffect 回调函数调用
 */
function commitHookEffectListMount(tag: number, finishedWork: Fiber) {
  // 获取任务队列
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  // 获取 lastEffect
  let lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  // 如果 lastEffect 不为 null
  if (lastEffect !== null) {
    // 获取要执行的副作用
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    // 通过遍历的方式调用 useEffect 中的回调函数
    // 在组件中定义了调用了几次 useEffect 遍历就会执行几次
    do {
      if ((effect.tag & tag) === tag) {
        // Mount
        const create = effect.create;
        // create 就是 useEffect 方法的第一个参数
        // 返回值就是清理函数
        effect.destroy = create();
      }
      // 更新循环条件
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```




#### 4. 请简述 workInProgress Fiber 树存在的意义是什么
* React 使用双缓存技术完成 Fiber 树的构建与替换，实现DOM对象的快速更新。
* 在 React 中最多会同时存在两棵 Fiber 树，当前在屏幕中显示的内容对应的 Fiber 树叫做 current Fiber 树，当发生更新时，React 会在内存中重新构建一颗新的 Fiber 树，这颗正在构建的 Fiber 树叫做 workInProgress Fiber 树。在双缓存技术中，workInProgress Fiber 树就是即将要显示在页面中的 Fiber 树，当这颗 Fiber 树构建完成后，React 会使用它直接替换 current Fiber 树达到快速更新 DOM 的目的，因为 workInProgress Fiber 树是在内存中构建的所以构建它的速度是非常快的。