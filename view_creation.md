<h3>View Creation</h3>

View Creation in Angular operates in a similar order to that in which change detection occurs:  From the <b><i>root</i></b> of the app.

Angular stores the information about the node's identity in the nodeDefinitions themselves using the `.flags` property of the node definition

>The `flags` are integers that, when converted to binary representation, guide Angular's logic for view creation and a whole lot more

Example:

nodeDef.flag = 49152  ---- to binary ----> 11000000 00000000

AKA 1<<14 and 1<<15 

Performing a lookup in Angular's nodeFlags provides the following defintions:

> 1<<14 = TypeDirective <br>
> 1<<15 = TypeComponent <br>

With the understanding that the .flags are the node definition's blueprints, the remainder of this document will hopefully make more sense.  

```createRootView(root: RootData, def: ViewDefinition, context?: any): ViewData```

This function truly kicks off the whole process of node creation in Angular


```createViewNodes(view:viewData)```
- checks whether the view it is provided is a componentView

- if yes, defines the view's parent node (parentDef)

- after, it iterates through every nodeDef associated with the view definition and delegates actions based on the type of the nodeDef's flags

```JavaScript
switch (nodeDef.flags & NodeFlags.Types) {
      case NodeFlags.TypeElement:
        const el = createElement(view, renderHost, nodeDef) as any;
        let componentView: ViewData = undefined !;
        if (nodeDef.flags & NodeFlags.ComponentView) {
          const compViewDef = resolveDefinition(nodeDef.element !.componentView !);
          componentView = Services.createComponentView(view, nodeDef, compViewDef, el);
        }
        listenToElementOutputs(view, componentView, nodeDef, el);
        nodeData = <ElementData>{
          renderElement: el,
          componentView,
          viewContainer: null,
          template: nodeDef.element !.template ? createTemplateData(view, nodeDef) : undefined
        };
        if (nodeDef.flags & NodeFlags.EmbeddedViews) {
          nodeData.viewContainer = createViewContainerData(view, nodeDef, nodeData);
        }
        break;
```

walking through what's happening in the codeblock above:

1)  The host element for the component is appended to the DOM, and any attribute key/binding information updated, before being returned as `el`
2)  The compiler attempts to resolve the element's definition
    - It checks against the internal Definition Cache to see if the component already has an associated factory
    - If a factory is already in the cache, it is returned; if it does not exist, the component is resolved into a factory, added to the cache, and returned as `compViewDef`
3)  The component view is created from the `compViewDef` factory function
4)  nodeData object is populated
5)  check performed regarding whether there are embedded views within (that is, views created by a directive within the component ViewContainerRef)

The `nodeData` is appended to the view's nodes (not its definition nodes)

So now we have a view's nodes created and appended, but what if that view has child nodes?  At this point, we've only rendered vanilla DOM nodes, component host elements, and instances of components/directives.  What about the views associated with a component (the child views)?  

enter the execComponentViewsAction function

```JavaScript 
export function createElement(view: ViewData, renderHost: any, def: NodeDef): ElementData {
  const elDef = def.element !;
  const rootSelectorOrNode = view.root.selectorOrNode;
  const renderer = view.renderer;
  let el: any;
  if (view.parent || !rootSelectorOrNode) {
    if (elDef.name) {
      el = renderer.createElement(elDef.name, elDef.ns);
    } else {
      el = renderer.createComment('');
    }
    const parentEl = getParentRenderElement(view, renderHost, def);
    if (parentEl) {
      renderer.appendChild(parentEl, el);
    }
  } else {
    // when using native Shadow DOM, do not clear the root element contents to allow slot projection
    const preserveContent =
        (!!elDef.componentRendererType &&
         elDef.componentRendererType.encapsulation === ViewEncapsulation.ShadowDom);
    el = renderer.selectRootElement(rootSelectorOrNode, preserveContent);
  }
  if (elDef.attrs) {
    for (let i = 0; i < elDef.attrs.length; i++) {
      const [ns, name, value] = elDef.attrs[i];
      renderer.setAttribute(el, name, value, ns);
    }
  }
  return el;
}
```

`execComponentViewsAction()`

1)  if there is no component view present in the definition return; else
2)  iterate over the node definitions until a component view is found then
3)  Return the component view from the actual view's current nodes

- the codeblock above exposes the mechanism by which <i><b>component</b></i> node definitions are handled


