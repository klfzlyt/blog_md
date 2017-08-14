---
title: React内部机制
date: 2017-08-14 17:20:52
tags: React
---

下面的文章介绍了React的内部机制，注释并解释下。

http://www.mattgreer.org/articles/react-internals-part-three-basic-updating/

待提问分析


```javascript
const FeactInstanceMap = {
    set(key, value) {
        key.__feactInternalInstance = value;
    },

    get(key) {
        return key.__feactInternalInstance;
    }
};

class FeactDOMComponent {
    constructor(element) {
        this._currentElement = element;
    }

    mountComponent(container) {
        const domElement = document.createElement(this._currentElement.type);
        const textNode = document.createTextNode(this._currentElement.props.children);

        domElement.appendChild(textNode);
        container.appendChild(domElement);

        this._hostNode = domElement;
        return domElement;
    }

    receiveComponent(nextElement) {
        const prevElement = this._currentElement;
        this.updateComponent(prevElement, nextElement);
    }

    updateComponent(prevElement, nextElement) {
        const lastProps = prevElement.props;
        const nextProps = nextElement.props;

        this._updateDOMProperties(lastProps, nextProps);
        this._updateDOMChildren(lastProps, nextProps);
    }

    _updateDOMProperties(lastProps, nextProps) {
        // nothing to do! I'll explain why below
    }

    _updateDOMChildren(lastProps, nextProps) {
        const lastContent = lastProps.children;
        const nextContent = nextProps.children;

        if (!nextContent) {
            this.updateTextContent('');
        } else if (lastContent !== nextContent) {
            this.updateTextContent('' + nextContent);
        }
    }

    updateTextContent(content) {
        const node = this._hostNode;
        node.textContent = content;

        const firstChild = node.firstChild;

        if (firstChild && firstChild === node.lastChild &&
            firstChild.nodeType === 3) {
            firstChild.nodeValue = content;
            return;
        }

        node.textContent = content;
    }
}

class FeactCompositeComponentWrapper {
    //自定义组件virtual  node
    constructor(element) {
        this._currentElement = element;
    }

    mountComponent(container) {
        const Component = this._currentElement.type;
        //这个type拿到一个函数 这个函数有那些对应的 react 方法
        const componentInstance = new Component(this._currentElement.props);
        //componentInstance 这个实例不是react vnode 的实例 是React.component的实例
        this._instance = componentInstance;

        FeactInstanceMap.set(componentInstance, this);

        if (componentInstance.componentWillMount) {
            componentInstance.componentWillMount();
        }

        const markup = this.performInitialMount(container);

        if (componentInstance.componentDidMount) {
            componentInstance.componentDidMount();
        }

        return markup;
    }

    performInitialMount(container) {
        //调用自定义组件的render方法，其实就是调用creactElement 出来的
        const renderedElement = this._instance.render();

        //virtual node 工厂得到child 可能会多个不同的实例 公用一组接口
        const child = instantiateFeactComponent(renderedElement);
        this._renderedComponent = child;

        //在这里抹平了mountComponent的过程
        //child.mountComponent 可能是FeactDOMComponent.mountComponent
        //也可能是FeactCompositeComponentWrapper.mountComponent
        return FeactReconciler.mountComponent(child, container);
    }

    performUpdateIfNecessary() {
        //应该是为事务做准备的 perform
        //两个一样的强制不走componentWillReceiveProps
        this.updateComponent(this._currentElement, this._currentElement);
    }

    receiveComponent(nextElement) {
        const prevElement = this._currentElement;
        this.updateComponent(prevElement, nextElement);
    }

    updateComponent(prevElement, nextElement) {
        //更新组件
        this._updating = true;
        const nextProps = nextElement.props;
        const inst = this._instance;

        const willReceive = prevElement !== nextElement;

        if (willReceive && inst.componentWillReceiveProps) {
            inst.componentWillReceiveProps(nextProps);
        }

        let shouldUpdate = true;

        const nextState = this._processPendingState();

        if (inst.shouldComponentUpdate) {
            shouldUpdate = inst.shouldComponentUpdate(nextProps, nextState);
        }

        if (shouldUpdate) {
            this._performComponentUpdate(nextElement, nextProps, nextState);
        } else {
            inst.props = nextProps;
        }

        this._updating = false;
    }

    _processPendingState() {
        const inst = this._instance;
        if (!this._pendingPartialState) {
            return inst.state;
        }

        let nextState = inst.state;

        for (let i = 0; i < this._pendingPartialState.length; ++i) {
            nextState = Object.assign({}, nextState, this._pendingPartialState[i]);
        }

        this._pendingPartialState = null;
        return nextState;
    }

    _performComponentUpdate(nextElement, nextProps, nextState) {
        this._currentElement = nextElement;
        //Component为真正React.Component
        const inst = this._instance;
        //真正更新状态
        inst.props = nextProps;
        inst.state = nextState;

        this._updateRenderedComponent();
    }

    _updateRenderedComponent() {

        const prevComponentInstance = this._renderedComponent;
        const inst = this._instance;
        //接受新props 和 state的时候再次render
        const nextRenderedElement = inst.render();
        //渲染过的_renderedComponent接受新的render得到的element
        //这个地方也会出现分叉，可能是FeactDOMComponent.receiveComponent
        //也可能是FeactCompositeComponentWrapper.receiveComponent
        FeactReconciler.receiveComponent(prevComponentInstance, nextRenderedElement);
    }
}

const TopLevelWrapper = function(props) {
    this.props = props;
};

TopLevelWrapper.prototype.render = function() {
    return this.props;
};

function instantiateFeactComponent(element) {
    //virtuan node 工厂
    if (typeof element.type === 'string') {
        return new FeactDOMComponent(element);
    } else if (typeof element.type === 'function') {
        return new FeactCompositeComponentWrapper(element);
    }
}

const FeactReconciler = {
    mountComponent(internalInstance, container) {
        return internalInstance.mountComponent(container);
    },

    receiveComponent(internalInstance, nextElement) {
        internalInstance.receiveComponent(nextElement);
    },

    performUpdateIfNecessary(internalInstance) {
        internalInstance.performUpdateIfNecessary();
    }
};

function FeactComponent() {}

FeactComponent.prototype.setState = function(partialState) {
    //在全局map中根据React.Component实例取出对应的内部的virtul node Component 实例
    const internalInstance = FeactInstanceMap.get(this);
    internalInstance._pendingPartialState = internalInstance._pendingPartialState || [];
    internalInstance._pendingPartialState.push(partialState);

    if (!internalInstance._updating) {
        //防止每次setState都触发更新
        FeactReconciler.performUpdateIfNecessary(internalInstance);
    }
}

function mixSpecIntoComponent(Constructor, spec) {
    const proto = Constructor.prototype;

    for (const key in spec) {
        proto[key] = spec[key];
    }
}


//createElement 建立了一个什么？
const Feact = {
    createElement(type, props, children) {
        const element = {
            type,
            props: props || {}
        };

        if (children) {
            element.props.children = children;
        }
        //返回一个对象，但是真实react不是这样？
        return element;
    },
    //creactClass利用FeactComponent(Component)混入成 Component
    createClass(spec) {
        function Constructor(props) {
            this.props = props;
            const initialState = this.getInitialState ? this.getInitialState() : null;
            //this为FeactComponent（React.Component）实例
            this.state = initialState;
        }

        Constructor.prototype = new FeactComponent();

        mixSpecIntoComponent(Constructor, spec);

        return Constructor;
    },
    //React.render 
    render(element, container) {
        //获得保存过的实例
        const prevComponent = getTopLevelComponentInContainer(container);

        if (prevComponent) {
            //container挂载过
            return updateRootComponent(prevComponent, element);
        } else {
            //render 返回真实dom
            return renderNewRootComponent(element, container);
        }
    }
};

function getTopLevelComponentInContainer(container) {
    return container.__feactComponentInstance;
}

function renderNewRootComponent(element, container) {
    //新建dom 挂载在container
    const wrapperElement = Feact.createElement(TopLevelWrapper, element);
    //new了 一个 vCompositeNode
    const componentInstance = new FeactCompositeComponentWrapper(wrapperElement);

    const markUp = FeactReconciler.mountComponent(componentInstance, container);
    // container.__feactComponentInstance 挂了 vCompositeNode 的render返回的vCompositeNode
    container.__feactComponentInstance = componentInstance._renderedComponent;

    return markUp;
}

function updateRootComponent(prevComponent, nextElement) {
    FeactReconciler.receiveComponent(prevComponent, nextElement);
}

const MyComponent = Feact.createClass({
    componentWillMount() {
        this.renderCount = 0;
    },

    getInitialState() {
        return {
            message: 'state from getInitialState'
        };
    },

    componentWillReceiveProps(nextProps) {
        this.setState({ message: 'state from componentWillReceiveProps' });
    },

    render() {
        this.renderCount += 1;
        return Feact.createElement('h1', null, 'this is render ' + this.renderCount + ', with state: ' + this.state.message + ', and this prop: ' + this.props.prop);
    }
});

Feact.render(
    Feact.createElement(MyComponent, { prop: 'first prop' }),
    document.getElementById('root')
);

setTimeout(function() {
    Feact.render(
        Feact.createElement(MyComponent, { prop: 'second prop' }),
        document.getElementById('root')
    );
}, 2000);

```