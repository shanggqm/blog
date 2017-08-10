![](https://github.com/shanggqm/blog/blob/master/asset/image/banner.jpg)
## 源码分析
在阅读一个库的源码之前，搞清楚这个库的存在是为了解决了什么问题至关重要。

[reselect](https://github.com/reactjs/reselect)的设计初衷是为了解决react-redux架构中，在把全局store里的state树的部分状态映射到对应组件的props中（mapStateToProps），每次组件更新的时候都需要执行计算的问题。即便组件依赖的状态并未发生改变。

因此可以知道，reselect提供的createSelector方法主要就是替代mapStateToProps函数的，并提了供缓存和比较机制，来减少组件更新时的无谓计算，从而提高性能。

实际使用示例如下：

```javascript
import { createSelector } from 'reselect'
import { connect } from 'react-redux'
import { toggleTodo } from '../actions'
import TodoList from '../components/TodoList'

const getVisibilityFilter = (state) => state.visibilityFilter
const getTodos = (state) => state.todos

const getVisibleTodos = createSelector(
  [ getVisibilityFilter, getTodos ],
  (visibilityFilter, todos) => {
    switch (visibilityFilter) {
      case 'SHOW_ALL':
        return todos
      case 'SHOW_COMPLETED':
        return todos.filter(t => t.completed)
      case 'SHOW_ACTIVE':
        return todos.filter(t => !t.completed)
    }
  }
)

const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state)
  }
}

const mapDispatchToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id))
    }
  }
}

const VisibleTodoList = connect(
  mapStateToProps,
  mapDispatchToProps
)(TodoList)

export default VisibleTodoList
```

```javascript
export const createSelector = createSelectorCreator(defaultMemoize)

export function createSelectorCreator(memoize, ...memoizeOptions) {/*...*/}
```

createSelector函数是由一个高阶函数，通过传入存储策略函数，以及缓存过期的比较函数来生成。reselect使用了默认提供的defaultMemoize和引用比较器`defaultEqualityCheck`来生成createSelector函数。

```javascript
export function defaultMemoize(func, equalityCheck = defaultEqualityCheck) {
  let lastArgs = null
  let lastResult = null
  // we reference arguments instead of spreading them for performance reasons
  return function () {
    if (!areArgumentsShallowlyEqual(equalityCheck, lastArgs, arguments)) {
      // apply arguments instead of spreading for performance.
      lastResult = func.apply(null, arguments)
    }

    lastArgs = arguments
    return lastResult
  }
}

function defaultEqualityCheck(a, b) {
  return a === b
}
```

`defaultMemoize`函数有两个参数，一个是计算函数func，一个是比较器函数，返回一个带缓存的结果计算函数。

通过闭包定义了两个私有变量：lastArgs和lastResult，分别代表上一次执行计算所用的参数集合和计算的结果。并在返回的结果计算函数中对最新的参数和上一次的参数进行equalityCheck，如果相同则使用缓存；如果不同则重新计算结果。

`defaultMemoize`是一个通用结果计算和缓存模型,提供了一个容量为1的缓存，计算缓存过期的逻辑就是根据传入的比较器函数判断前后两次参数是否相同。

搞清楚defaultMemoize，理解reselect就非常简单了，我们接着来看createSelectorCreator函数：
```javascript
export function createSelectorCreator(memoize, ...memoizeOptions) {
  return (...funcs) => {}
}
```
我们常用的createSelector方法就是由这个函数的返回值，支持传入多个函数作为参数，和我们熟知的createSelector写法一致：
```javascript
createSelector(funA, funcB, (a, b) => {});
createSelector([funcA, funcB], (a, b) => {});
```

最后一个参数是结果计算函数，前面的funA和funB为中间结果计算函数，称之为结果计算函数的依赖函数。我们来看看createSelector函数内部的实现：
```javascript
return (...funcs) => {
    //重新计算的次数统计
    let recomputations = 0
    //弹出结果计算函数
    const resultFunc = funcs.pop()
    //提取依赖函数数组，并对funcs数组中元素进行校验，默认只支持function
    const dependencies = getDependencies(funcs)

    //对结果计算函数使用用户传入的缓存策略（默认为上面说的defaultMemoize）。并且每执行一次结果计算函数，计数就加1
    const memoizedResultFunc = memoize(
      function () {
        recomputations++
        // apply arguments instead of spreading for performance.
        return resultFunc.apply(null, arguments)
      },
      ...memoizeOptions
    )

    //selector就是createSelector的返回值函数。
    //selector函数要做的事情就是把依赖的每一个中间结果计算函数依次执行，并组装成一个结果数组。
    //交给结果计算函数处理。在selector函数的参数值引用未发生变化时，中间计算函数不需要重复进行计算。
    const selector = defaultMemoize(function () {
      const params = []
      const length = dependencies.length
     
      for (let i = 0; i < length; i++) {
       
        // 循环执行所有依赖的计算函数，并把执行的结果保存在数组缓存中
        params.push(dependencies[i].apply(null, arguments))
      }

      //执行结果计算函数，返回最终计算结果
      return memoizedResultFunc.apply(null, params)
    })

    //绑定结果处理函数
    selector.resultFunc = resultFunc
    selector.recomputations = () => recomputations
    selector.resetRecomputations = () => recomputations = 0
    return selector
  }
```
到这一步基本上就吧reselect的核心代码分析完了，还剩下一个`createStructuredSelector`函数，是reselect提供的一种简化特殊场景的代码书写的方法。基本意思就是可以把下面这种写法：
```javascript
const mySelectorA = state => state.a
const mySelectorB = state => state.b

const structuredSelector = createSelector(
   mySelectorA,
   mySelectorB,
   mySelectorC,
   (a, b, c) => ({
     a,
     b,
     c
   })
)
```
简写成
```javascript
const mySelectorA = state => state.a
const mySelectorB = state => state.b

const structuredSelector = createStructuredSelector({
  x: mySelectorA,
  y: mySelectorB
})

const result = structuredSelector({ a: 1, b: 2 }) // will produce { x: 1, y: 2 }
```

源码不难，就不解读了。

## 最佳实践和思考
通过上面源码分析，可以得到如下的一些结论和思考：

1. createSelector(state)中的state如果发生变化（引用发生变化），则会让所有的子selector缓存失效，但如果所有中间计算结果没有发生变化的话，最终resultFunc的缓存仍然可以正常使用。

2. reselect的比较机制是简单的参数引用比较，如果参数引用发生了变化，则需要重新执行计算；否则走缓存。有两个地方有缓存：依赖的函数计算结果和最终计算结果
3. selector.recomputations可以用来判断重复计算的次数
4. reselect的设计目标是为了避免在redux流程中引发的mapStateToProp函数中的重复计算，而如果并不存在比较多的state复杂计算场景，则reselect的作用就显得微乎其微了，甚至由于闭包的存在，还会影响执行性能
5. 当触发组件的更新流程时，往往会把对要传递给子组件的props的组装和计算流程写在render函数内部，更好的做法是在reselect中提前做好，render里直接拿就可以了，这样可以充分利用reselect的缓存。还可以更清晰地定义组件的state结构。
6. reselect官方推荐的用法是和react-redux的connect方法配合使用，以便减少在mapStateToProps的时候重复计算性能损耗。然而在一个复杂的SPA系统里，当组件层级较深时，connect并不推荐在所有层级的组件里使用。因为每一个组件在connect的时候，可以将state tree上任意的状态对象映射到当前组件的props里，在团队协同开发，缺乏规范的状态小，这会导致组件对state tree的依赖变的异常复杂和不可控。一个比较靠谱的做法是在state tree的设计上尽量保证上层页面或者模块之间，在数据上尽量独立，互不依赖，模块内部的组件细分则通过至上而下的props进行流转便可。如此，connect操作也就仅限于在数量有限的上层组件上使用。这一定程度上限制了reselect的使用频次，因此在性能和可维护性上，到底如何宣召平衡点，则是一个需要深入思考的问题了。

    通常来说，移动应用的每一个页面的功能和数据相对简单，和其他页面之间的数据关系也比较独立，很少会有模块联动的需求。因此在不同层级频繁使用connect进行state到props的映射计算是一种可行且可控的实践方式。而对于复杂的PC Web而言，情况就有所不同了，同一个页面上，多个模块之间经常会有复杂的数据联动和依赖关系，而且代码量也会比较大。放开connect操作的使用场景，可以带来数据获取的便利和灵活性，但会造成模块之间数据耦合度高的可维护性问题。
    
    因此从这点上讲，reselect可能在移动端应用上更有用武之地。但另一个问题是：redux又通常被推荐在复杂的PC SPA项目中使用，真是让人纠结的很啊。

7. 除了在connect时使用，reselect可以在其他场景中使用吗？

    reselect虽然设计用来解决redux的问题，但其本质就是一个数据映射的缓存方案，也就是说任何需要将一个数据对象经过计算映射成另外一个数据对象的场景都可以使用reselect来提高性能。
    
    譬如：在一个一次性加载所有数据的表格组件中，存在一个按照状态字段进行过滤的功能。
    
    ```javascript
    const STATUS_NORMAL = 0;
    const STATUS_PAUSE = 1;
    const STATUS_FAIL = 2;
    
    const getNormal = (data)=> data.filter(item=> item.status === STATUS_NORMAL);
    const getPause = (data) => data.filter(item => item.status === STATUS_PAUSE);
    const getFail = (data) => data.filter(item => item.status === STATUS_FAIL);
    
    const filterSelector = createSelector(
        getNormal, 
        getPause, 
        getFail, 
        (normal, pause, fail) => {
            return {
                [STATUS_NORMAL]: normal, 
                [STATUS_PAUSE]: pause, 
                [STATUS_FAIL]: fail
            };
        }
    );
    
    function onFilter(status){
        render(myData[status]);
    }
    
    function load(data){
        return filterSelector(data);
    }
    
    const myData = load(data);
    ```
    
    看起来很完美，对吗？每次切换状态过滤的时候，都不用再次去进行分类过滤计算了。当重新执行load(data)时，如果data并没有发生变化，也不会再次进行计算。
    
    实际情况是这样吗？其实不是，reselect默认使用的比较器是引用比较，也就是说，判断data有没有发生变化的方式是 
    ```javascript
    oldData === data
    ```
    显然大多数场景下新的data往往来自于ajax请求的响应，其引用和oldData并不相同，所以reselect的缓存机制并未生效。
    
    使用引用比较器的原因是reselect的主要设计目标是为react-redux架构提供性能优化服务的，在这个体系中state是一个全局的Immutable对象，对象只要引用没有发生变化，就没必要再次执行映射计算，而一旦发生变化，其引用也会发生变化。因此使用引用比较省时省力，也更适配redux的理念。
    
    因此上面例子中只有在onFilter过滤的时候可以使用缓存，而在更新数据data的时候，reselect的缓存就会失效。这样看来，使用reselect做如上的数据映射处理的优势并不明显，和自己手动去做缓存区别不大：
    
    ```javascript
    const getNormal = (data)=> data.filter(item=> item.status === STATUS_NORMAL);
    const getPause = (data) => data.filter(item => item.status === STATUS_PAUSE);
    const getFail = (data) => data.filter(item => item.status === STATUS_FAIL);
    
    const myData = {
        [STATUS_NORMAL]: getNormal(data), 
        [STATUS_PAUSE]: getPause(data), 
        [STATUS_FAIL]: getFail(data)
    };
    
    function onFilter(status){
        render(myData[status]);
    }
    ```
    
    当然，reselect还提供了更高级的用法，允许开发者自定义存储函数和比较器函数，来实现更个性化的需求。
    
    ```javascript
    const memorize = (func, equalityCheck){
        //...
    }
    
    const deepCheck(a, b){
        //deep check for a and b
    }
    
    const createSelector = createSelectorCreator(memorize, deepCheck);
    
    ```
    具体使用场景则有待大家自己去发掘了。
    

## 结论   
归纳一下最佳实践就是：
- reselect适合用在state到props有很多计算的场景，计算量少的场景可以不用
- 尽量把在render里对props的计算提取出来，放到reselect里去做，充分利用缓存
- connect时使用reselect次数越多，越能体现reselect的缓存优势；但connect太多会影响可维护性，需要找个平衡点
