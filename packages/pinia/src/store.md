```js
// 每次调用一次 defineStore 函数就返回一个新的函数, 这个函数捕获传入的 id
const useFooStore = defineStore('foo', setup, setupOptions) {
  const isSetupStore = typeof setupFn === 'function'
  // the option store setup will contain the actual options in this case
  const options = isSetupStore ? setupOptions : setup;

  // 闭包捕获传入的 id = 'foo'
  function useStore(pinia?: Pinia | null,  hot?: StoreGeneric) {
    // 首先确保当前的函数是在用户的组件 setup 函数中执行
    // 组件 options api 内部也会转成 在 setup 函数中执行
    const hasContext = hasInjectionContext()
    // 若是没有出入参数 pinia 就从 app.provide(piniaSymbol, pinia) 中获取
    // 1. 当传入 pinia 参数时, 此时 函数可以不在组件的 setup 函数中执行
    // 2. 当不传 pinia 参数时, 此时 默认从 app.provide() -> appContext.provides[piniaSymbol] 获取
    pinia = pinia || (hasContext ? inject(piniaSymbol, null) : null)
    if (pinia) setActivePinia(pinia);

    // 当没有全局的 pinia 时, 这里直接中断执行,并抛出错误
    if (__DEV__ && !activePinia) {
      throw new Error(
        `[🍍]: "getActivePinia()" was called but there was no active Pinia. Are you trying to use a store before calling "app.use(pinia)"?\n` +
          `See https://pinia.vuejs.org/core-concepts/outside-component-usage.html for help.\n` +
          `This will fail in production.`
      )
    }
    pinia = activePinia!

    // 若是首次使用这个 id 对应的 store, 则进行创建 id 对应的 store
    if (!pinia._s.has(id)) {
      // creating the store registers it in `pinia._s`
      if (isSetupStore) {
        // setup 为函数
        createSetupStore(id, setup, options, pinia) {
          let $id = id
          let scope!: EffectScope
          const optionsForPlugin = assign({ actions: {}}, options);

          // 这个 pinia 执行了 pinia.$destroy() -> pinia._e.stop()
          if (__DEV__ && !pinia._e.active) {
            throw new Error('Pinia destroyed')
          }

          // watcher options for $subscribe
          // $subscribe 其实就是一个对 当前 store.$state 的一个深度 watch
          const $subscribeOptions: WatchOptions = { deep: true }

          // internal state
          let isListening: boolean // set to true at the end
          let isSyncListening: boolean // set to true at the end
          let subscriptions: SubscriptionCallback<S>[] = []
          // 每一个 store 对应自己的 actionSubscriptions
          let actionSubscriptions: StoreOnActionListener<Id, S, G, A>[] = []

          // 客户端第一次调用肯定是没有值的,
          // 但是在服务端可能已经设置了值, 那么再次在客户端调用可能存在 SSR 水合的值
          const initialState = pinia.state.value[$id]

          if (!isOptionsStore && !initialState) {
            pinia.state.value[$id] = {}
          }

          /**
           * Helper that wraps function so it can be tracked with $onAction
           * @param fn - action to wrap
           * @param name - name of the action
           * 回传给用户 setup 函数的 action 函数 - 一个包装函数, 返回的是一个包装函数
           * 确保 $onAction 钩子函数的执行顺序
           */
          const ACTION_MARKER = Symbol() // 全局的
          const action = function(fn, name) {
            if (ACTION_MARKER in fn) {
              // we ensure the name is set from the returned function
              ;(fn as unknown as MarkedAction<Fn>)[ACTION_NAME] = name
              // 表示这个函数已经被处理的,无需再次处理
              return fn
            }
            const wrappedAction = function (this: any) {
              // 在包装函数中, 确保执行的 action 函数上下文中始终有 activePinia
              setActivePinia(pinia)
              const args = Array.from(arguments)

              const afterCallbackList: Array<(resolvedReturn: any) => any> = []
              const onErrorCallbackList: Array<(error: unknown) => unknown> = []
              function after(callback: _ArrayType<typeof afterCallbackList>) {
                afterCallbackList.push(callback)
              }
              function onError(callback: _ArrayType<typeof onErrorCallbackList>) {
                onErrorCallbackList.push(callback)
              }

              function triggerSubscriptions(subscriptions, ...args) {
                subscriptions.slice().forEach((callback) => callback(...args))
              }

              // 在执行用户的 action 函数之前, 先执行这里的 triggerSubscriptions 函数
              // 就是通过 store.$onAction(callback) 注册的函数
              // @ts-expect-error
              triggerSubscriptions(actionSubscriptions, {
                args, // 执行 action 函数传入的 参数
                name: wrappedAction[ACTION_NAME], // action 函数名称(key)
                store, // 当前 action 执行时对应的 store
                after, // 注册 action 执行完后的回调
                onError, // 注册 action 执行异常的回调
              }) => {
                //
                store.$onAction(({args, name, store, after,  onError }) => {
                  // 这里是钩子函数执行前执行回调
                  console.log('action 函数执行前...')
                  // 这调用 after 注册 action 函数执行完后执行的回调函数
                  after((value) => {
                    // action 函数执行完后执行
                    // 这里的 value 就是 action 函数执行完后的返回值, 若是 promise 则是 resolve 后的值
                    console.log('action 函数执行后...', value)
                  })
                  onError((err) => {
                    // 是 action 函数执行出错后的回调
                    console.log('action 函数执行异常...', err)
                  })
                })
              }

              let ret: unknown
              try {
                // 给 用户 action 函数绑定 this (箭头函数无 this)
                ret = fn.apply(this && this.$id === $id ? this : store, args)
                // handle sync errors
              } catch (error) {
                triggerSubscriptions(onErrorCallbackList, error)
                throw error
              }

              if (ret instanceof Promise) {
                // 注意这里是 promise 直接返回, 不执行下面的语句了,
                // 不会导致 triggerSubscriptions(afterCallbackList) 重复执行, 提前 return
                return ret
                  .then((value) => {
                    triggerSubscriptions(afterCallbackList, value)
                    return value
                  })
                  .catch((error) => {
                    triggerSubscriptions(onErrorCallbackList, error)
                    return Promise.reject(error)
                  })
              }
              // 执行到这里表示不是 promsie
              // trigger after callbacks
              triggerSubscriptions(afterCallbackList, ret)
              return ret
            } as MarkedAction<Fn>

            wrappedAction[ACTION_MARKER] = true
            wrappedAction[ACTION_NAME] = name // will be set later

            // @ts-expect-error: we are intentionally limiting the returned type to just Fn
            // because all the added properties are internals that are exposed through `$onAction()` only
            return wrappedAction
          }

          // 定义每个 store 的操作方法
          const $reset = function() {
            const { state } = options
            // NOTE: 这个方法只能在 options api 中才有效, 在 setupStore 中无效
            const newState = state ? state() : {}
            // we use a patch to group all changes into one single subscription
            this.$patch(($state) => assign($state, newState))
          }
          // 提交对状态数据的修改
          // 通过 $patch 修改, 这里可以对修改进行劫持, 可以结合开发者工具(chrome插件进行修改的状态查看))
          // 1. $patch(stateMutationFn) 直接出入 $patch 一个状态修改函数
          // $patch(($state) => {
          //   $state.xxx = 1
          // })
          // 2. 或者直接传入一个对象进行修改, 会进行状态合并
          // $patch({a: 1})
          // $patch 既接受一个状态修改函数,也可以传入一个状态对象进行修改
          const $patch = function(partialStateOrMutator) {
            let subscriptionMutation: SubscriptionCallbackMutation<S>
            isListening = isSyncListening = false
            if (typeof partialStateOrMutator === 'function') {
              partialStateOrMutator(pinia.state.value[$id] as UnwrapRef<S>)
              subscriptionMutation = {
                type: MutationType.patchFunction,
                storeId: $id,
                events: debuggerEvents as DebuggerEvent[],
              }
            } else {
              // 对象
              mergeReactiveObjects(pinia.state.value[$id], partialStateOrMutator)
              subscriptionMutation = {
                type: MutationType.patchObject,
                payload: partialStateOrMutator,
                storeId: $id,
                events: debuggerEvents as DebuggerEvent[],
              }
            }
            const myListenerId = (activeListener = Symbol())
            nextTick().then(() => {
              if (activeListener === myListenerId) {
                isListening = true
              }
            })
            isSyncListening = true
            // because we paused the watcher, we need to manually call the subscriptions
            // 调用通过 store.$subscribe(fn) 注册的回调
            // store.$subscribe(({type, payload, storeId, events}, store) => {
            //   //
            // }, options)
            // 修改后会触发这里的 $subscribe() 注册的回调函数
            triggerSubscriptions(
              subscriptions,
              subscriptionMutation,
              pinia.state.value[$id] as UnwrapRef<S>
            )
          }
          // 订阅当前 store.$state 的变化,
          // 1. 没有调用 store.$patch(),
          //     那么若是直接通过修改响应式的值引起的变化,
          //     就会执行触发这里的注册回调执行, 其实此种情况的回调是 watch 的回调
          // 2. 直接通过调用 store.$patch() 进行修改 会设置 isListening = isSyncListening = false
          //    那么此时就会执行这里的回调, 而不是触发 watch 的回调, 因为 watch 的回调执行后, 因为
          //    这里的  isListening 为 false 所以不会执行注册的 callback
          const $subscribe = function(callback, options = {}) {
            // subscriptions 是这里的 局部变量, 对应当前的 store 的, 每一个 store 都有其对应的 subscriptions
            const removeSubscription = addSubscription(
              subscriptions,
              callback,
              options.detached,
              // 执行 removeSubscription() 也会执行这里的传入的 stopWatcher()
              () => stopWatcher()
            )
            const stopWatcher = scope.run(() =>
              // 原来一个 $subscribe 其实就是一个深度 watch, 对当前 store.$state 的深度 watch
              watch(
                () => pinia.state.value[$id] as UnwrapRef<S>,
                (state) => {
                  // options 其实就是 watch options
                  if (options.flush === 'sync' ? isSyncListening : isListening) {
                    callback(
                      {
                        storeId: $id,
                        type: MutationType.direct,
                        events: debuggerEvents as DebuggerEvent,
                      },
                      state
                    )
                  }
                },
                assign({}, $subscribeOptions, options)
              )
            )!

            return removeSubscription
          },

          const $dispose = function() {
            scope.stop()
            subscriptions = []
            actionSubscriptions = []
            pinia._s.delete($id)
          }
          // actionSubscriptions 这里是一个局部变量
          // 当我们调用 store.$onAction() - 注册当前 store 中 actionSubscriptions
          // 不会注册到其他的 store actionSubscriptions, 每个 store 都有自己的 actionSubscriptions
          // 每一个 store 对应自己的 actionSubscriptions
          // $onAction 其实就是 addSubscription 函数的封装, 默认封装了参数 actionSubscriptions
          // 调用 $onAction 函数总是会调用 addSubscription(actionSubscriptions, ...args)
          // 保证了 actionSubscriptions 总是这个
          // 当调用 $onAction(callback, detached, onCleanup = noop)
          // 会将 callback 放入 当前函数的局部变量 actionSubscriptions 中
          // 在执行 store 中的 action 函数时, 先执行这里注册的 actionSubscriptions 中函数
          // 执行 action() 时
          const $onAction = addSubscription.bind(null, actionSubscriptions) {
            actionSubscriptions.push(callback)
            const removeSubscription = () => {
              const idx = subscriptions.indexOf(callback)
              if (idx > -1) {
                subscriptions.splice(idx, 1)
                onCleanup()
              }
            }

            if (!detached && getCurrentScope()) {
              onScopeDispose(removeSubscription)
            }

            return removeSubscription
          }

          const partialStore = {
            _p: pinia,
            $id,
            $reset,
            $patch,
            $dispose,
            $onAction,
            $subscribe,
          }

          // 创建一个 store
          // NOTE: 整个 store 就是一个 reactive
          const store = reactive(partialStore)

          // store the partial store now so the setup of stores can instantiate each other
          // before they are finished without creating infinite loops.
          pinia._s.set($id, store)

          // app.runWithContext
          const runWithContext = pinia._a && pinia._a.runWithContext)

          // NOTE: 这里才开始执行用户传入的 setup 函数, 放在 effectScope 中执行
          // 以便让 pinia._e 进行追踪管理
          const setupStore = runWithContext(() =>
            // scope.parent -> pinia._e
            // pinia._e.scopes.push(scope)
            // scope.effects.push(setup())
            // scope.effects.push(setup())
            // setup 函数中执行过的 effect (watchEffect) 将被放入这个
            // 最终 pinia._e.scopes 里面装的就是所有通过 defineStore() 定义的状态对应的 scope
            // pinia._e.scopes[scope1[effect1, effect2, effect3, ...], scope2, scope3, ...]
            // 每一个 scope 对应一个 store, 或者说每一个 store 对应一个 scope
            pinia._e.run(() => (scope = effectScope()).run(() => setup({ action }))!)
            // NOTE: 注意这里的 setup({action}) 回调参数, 会将上面定义的 action 函数传入给用户调用
            // 模拟 setup 函数执行 -> 比如在组件的 setup 函数中执行
            setup({ action })){
              // 这里回调参数 action 是一个函数, 这里面是可以进行调用的
              const foo = {}
              const bar = reactive()
              const todo = ref('-')
              const todos = ref([])
              // NOTE: vue3.5 以上版本 计算属性现在已经不再基于 effect 了, 不受 effect.stop() 的影响了
              const comp = computed(() => {
                return todos.value.map((item, index) => {
                  return { item, index} })
              })
              // 这里面的 watch effect 将会被 pinia_e 进行搜集管理,
              // 可以通过 store.$dispose() -> 内部调用 pinia._e.stop() 让其失活, watch 则不会再次运行
              watch(() => todo.value, () => {
                console.log('watch')
              })
              function addTodo() {
                todos.value.push(todo.value)
              }
              function delTodo(index) {
                todos.value.splice(index, 1)
              }
              return {
                foo, // 返回的不是 isRef 也不是 isRective 不会放入 pinia.state[id] 中
                bar, // 是响应式数据 会放入 pinia.state[id] = { bar, }
                todo, // 是响应式数据 会放入 pinia.state[id] = { bar, todo }
                todos, // 是响应式数据 会放入 pinia.state[id] = { bar, todo, todos }
                comp, // 是计算属性 不会放入 pinia.state[id] = { bar, todo, todos }
                addTodo, // 是函数 不会放入 pinia.state[id]
                delTodo, // 是函数 不会放入 pinia.state[id]
              }
            }
          );

          // 处理 setup 函数返回的值:
          // 这里要注意 setup 函数的返回值被分类到几个不同的部分:
          // 响应式事状态的数据(ref,reactive) 放入到
          // pinia.state.value[id] = {}
          // overwrite existing actions to support $onAction
          for (const key in setupStore) {
            const prop = setupStore[key]
            // 这里之所以排除计算属性, 是因为计算属性是 lazy 的, 只有在重新渲染创建新的 vnode 时才会被读取值
            if ((isRef(prop) && !isComputed(prop)) || isReactive(prop)) {
              // 属性是响应式状态: 这里其实排除了函数
              // 比如上面的 setup 函数返回的 bar, todo, todos
              if (!isOptionsStore) {
                pinia.state.value[$id][key] = prop
              }
            }  else if (typeof prop === 'function') {
              // action: 返回的函数, 在 pinia 中将作为 action 对待,
              // 等价于 otptionsStore 中的 actions 中的 函数
              // 函数,比如上面返回的 addTodo, delTodo
              const actionValue = action(prop, key)
              setupStore[key] = actionValue
              // list actions so they can be used in plugins
              // 将从 setup 函数中返回的 函数收集起来放入到 optionsForPlugin.actions 中
              optionsForPlugin.actions[key] = prop
            } else if (__DEV__) {
              // 其他的属性 比如 foo 不是响应式数据(排除计算属性)也不是函数直接放入
              // store 中, 即下面执行的 assign(store, setupStore)
              // 可以直接通过 store.foo 进行访问
              if (isComputed(prop)) {
                // ...
              })
            }
          }

          // add the state, getters, and action properties
          assign(store, setupStore)
          // 将上面收集的用户 setup 函数返回的响应式状态数据映射到
          // store.$state, 便于直接通过 store.$state 属性进行访问设置,
          // 因为 $ 开头的, 默认规则表示公有属性/方法
          Object.defineProperty(store, '$state', {
            get: () => pinia.state.value[$id],
            // store.$state = {} -> 触发的是 assign($state, {}) 操作
            set: (state) => $patch(($state) => assign($state, state)),
          })
          // NOTE:
          // 以上务必需要区分 store 与 state 的区别:
          // state -> 仅仅是数据的对象, 就是专门放入用户传入的响应式数据的
          //          用户 setup 函数中返回的对象,可以包含任意类型(函数,普通对象,字符串,响应式对象)
          //          将用户定义的响应式对象数据专门放入到 pinia.state[id] = {} 中
          // store -> 则是包含 state 的, 已经用户返回的所有数据,
          //          同时内置了常用的方法($patch,$reset等)进行对 state(effect), scope 进行设置(触发更新),
          //          以及用户自定义的方法
          // store    其实就是封装了一组数据(state),一组方法(action), 同时可以在其方法中直接进行状态数据的操作
          //          一个 store 可以用在多个组件中, 进行状态(state)与方法的共享. 在任意的组件中, 只要操作了
          //          引入这个 store 的方法, 那么就会触发所有引入这个 store 组件的共同更新. 这样就解决了组件之间的
          //          状态共享了, 而且还是操作方法的共享.
          // 故可以理解: 一个 store 就是由 state 与 actions 组成. 其中 state 为响应式变量, action 为函数
          // store.$state,
          // store.fn1(), 执行 action
          // store.fn2(), 执行 action
          // store.$onAction(callback) // 注册 store.fn1() 执行时的事件,只要在执行 store.fn1() 时, 就会
          // 先执行 store.$onAction(callback) 中触发的 callback 函数

          // 在首次创建 store 时, 会让每个插件函数执行, 把创建的 store 回传给插件函数, 这样在插件函数中就可以
          // 获取到每个创建的 store
          // apply all plugins
          pinia._p.forEach((extender) => {
            // 执行使用 pinia.use(fn) 注册的插件函数
            assign(
              store,
              scope.run(() =>
                extender({
                  store: store as Store,
                  app: pinia._a,
                  pinia,
                  options: optionsForPlugin,
                })
              )!
            )
          })

          return store
        }
      } else {
        // setup 为选项, 内部本质也是转成 setup 函数的
        createOptionsStore(id, options as any, pinia) {
          const { state, actions, getters } = options
          const initialState = pinia.state.value[id];
          let store
          function setup() {
            if (!initialState) {
              pinia.state.value[id] = state ? state() : {}
            }
            // 将 options 中的 state 返回的对象都转成 ref
            // {a: 1, b: {} } => {a: ref(1), b: ref({})}
            const localState = toRefs(pinia.state.value[id])

            // setup 返回对象
            return assign(
              localState,
              actions, // 将 options actions 中转成 setup 中的普通函数
              // options 中的 getters 函数转成计算属性的 getter, 并以计算属性返回
              Object.keys(getters || {}).reduce(
                (computedGetters, name) => {
                  if (__DEV__ && name in localState) {
                    console.warn(
                      `[🍍]: A getter cannot have the same name as another state property.
                      Rename one of them. Found with "${name}" in store "${id}".`
                    )
                  }
                  computedGetters[name] = markRaw(
                    computed(() => {
                      setActivePinia(pinia)
                      // it was created just before
                      const store = pinia._s.get(id)!

                      // allow cross using stores

                      // @ts-expect-error
                      // return getters![name].call(context, context)
                      // TODO: avoid reading the getter while assigning with a global variable
                      return getters![name].call(store, store)
                    })
                  )
                  return computedGetters
                },
                {} as Record<string, ComputedRef>
              )
            )
          }
          store = createSetupStore(id, setup, options, pinia, hot, true /* isOptionsStore */)
          return store as any
        }
      }
    }

    // 通过 id 查找对应的 store, 最后返回的就是一个 store
    const store = pinia._s.get(id)!

    isListening = true
    isSyncListening = true
    return store
  }

  useStore.$id = id
  // 返回的 useStore 可以被执行多次(会有缓存,多次读取缓存: pinia._s.get(id))
  return useStore
}

// 这里导出的 useFooStore() 其实执行的就是 内部的 useStore
const fooStore = useFooStore() {
}
```
