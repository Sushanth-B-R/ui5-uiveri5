# Locators

## What to Prefer
Always work on the highest level of abstraction that is possible in the specific case. 
* Prefer control locators instead of DOM-level locators. 
* Prefer ID selectors if you have manually assinged IDs. 
* Prefer hierarchical class selectors but avoid layout-specific classes and try to stick to semantical classes.

__Try to compose the selector as if you are explaining to a manual tester where to click.__

## What to Avoid

### Avoid ID Locators that Use Generated IDs
Selection of a DOM element by ID is the simplest and most widely used approach in classical website testing.
The classical web page is composed manually. This way the important elements are assigned nice and meaningful IDs so it is easy to identify those elements in automatic tests.

In highly-dynamic JS frameworks, such as SAPUI5, the DOM tree is generated out of the views. The views can
also be generated from the content meta information. Very often, IDs are not assigned by a developer during application
creation. In such cases, the ID is generated in runtime from the control name with a suffix that is the sequential number
of this control in the app. The generated ID can also contain prefix of the enclosing view, such as `__xmlview1`.
In this scheme, the leading "__" means "internal and generated, not to be relied on".

There are several problems with using such generated IDs in application tests:

1. IDs are stable between application runs but are generated and will definitely change when the application is modified.
Even minor unrelated change, such as adding one more button in some common area like the header can cause a change of
all IDs. This will require changes in all selectors used in all tests for this application.
2. A change of IDs is even more probable with metadata-driven UIs, such as SAP Fiori Elements. The SAP Fiori Elements template can change with a UI5 minor version upgrade and can introduce new visual elements that will also change the generated IDs of the rest.
3. IDs are execution-unique and are generated on the runtime. So repetitive ID's require repetitive navigation path
in the application. This makes it especially hard for a human to develop and support the test as manual reproduction requires to always start from the beginning and pass over the whole test. It is also impossible to execute only part of the whole scenario by using disabled or focused specs and suites.
4. The generated IDs can be different depending on the environment the application is running on.
5. Generated IDs are not self-documenting and this makes the test harder to understand and maintain.

### Avoid Non-visible Attributes
Have in mind the point of view of the users. Users do not see DOM nodes and their attributes but see rendered DOM.
So, write selectors that include only "visible" attributes.
This also makes the test more self-documenting and simplifies maintenance.

### Minimize Use of Attribute Locators
These locators are slow and are usually not closely related to the visual representation. Besides, attribute values may often change and may not be specific enough if used on their own.

## Control Locators
In order to use the `control` locator in your tests, the application being tested must use a certain version of UI5. All versions newer than 1.54 are acceptable, as well as all patches of version 1.52 and 1.54 after and including 1.52.12 and 1.54.4.

In the application testing approach we use hierarchical class locators composed of UI5 control main
(marker) class names (the class names of the control root DOM element). This hierarchical composition is important to guarantee the stability of locators. Usage of classes is problematic as DOM is not a UI5 API and it can change between UI5 minor releases. Only UI5 JS API is guaranteed to be backward-compatible. One approach to mitigate this issue is to use control locators.

The `control` locator is closely tied to the control level of abstraction and therefore should be much more intuitive for application developers. The `control` locator object can be written easily by inspecting the application using the [UI5 Inspector](https://chrome.google.com/webstore/detail/ui5-inspector/bebecogbafbighhaildooiibipcnbngo).

The locator is created using the `by` collection of factory functions:
```javascript
element(by.control({id: "testID"});
// or to find multiple elements:
element.all(by.control({id: /test/});
```

Using the `control` locator gives you an `ElementFinder` of the DOM element best representing the found control. Since there can be more than one representation of a control, you can choose which one fits best a desired interaction. This is a common pitfall and is described below in the Interactions section.

### Syntax
Under the hood, control locators rely on [OPA5](https://openui5.hana.ondemand.com/#/api/sap.ui.test.Opa5/overview) functionality. If you are familiar with OPA5's `waitFor` structure, then you will be able to immediately transition to control locators. The difference between a control selector and a typical OPA5 `waitFor` is that some values are not allowed. The unsupported property values are: `matchers` and `actions` object constructions and `check`, `success` and `actions` functions.

`by.control` accepts a plain object specifying the `viewName`, `viewNamespace`, ID, `controlType`, ID suffix, and other properties of the control to look for. The ID can be a string or a regular expression. Just like in OPA5, if a `viewName` is given, the ID is the view-relative ID, otherwise it is the global ID. All [OPA5 matchers](https://openui5.hana.ondemand.com/#/api/sap.ui.test.matchers/overview) are supported with exception of `AncestorMatcher` that is supported implicitly as documented below. In OPA5, you normally create a matcher instance and pass the expected parameters to the constructor as a plain object. In a control selector, you can set the same plain object parameters to the matcher property.

Matchers syntax:
```javascript
// find an object header with full ID matching "myViewID--foo.[0-9]+" and property data binding for model "JSONModel"
element(by.control({
  id: /^foo.[0-9]+/,
  viewName: "myViewName",
  controlType: "sap.m.ObjectHeader",
  bindingPath: {path: "/products/1", propertyPath: "Status", modelName: "JSONModel"}
});
// other examples of matcher properties:
  I18NText: {propertyName: "text", key: "buttonText"}
  labelFor: {key: "labelText", modelName: "i18n"}
  labelFor: {text: "myText"}
  properties: {text: "My Header Text"}
  aggregationContainsPropertyEqual: {aggregationName: "myAggregation", propertyName: "enabled", propertyValue: true}
  aggregationLengthEquals: {name: "myAggregation", value: 1}
  aggregationEmpty: {name: "myAggregation"}
  aggregationFilled: {name: "myAggregation"}
```

Multiple uses of one type of matcher in a single selector:
```javascript
element(by.control({
  viewName: "myViewName",
  controlType: "sap.m.ObjectHeader",
  aggregationFilled: [
    {name: "myAggregation"},
    {name: "myOtherAggregation"}
  ]
}))
```

### Interaction adapters
A control DOM tree may include multiple interactable DOM elements. If you need to interact with a specific DOM element of this tree, use an interaction adapter.

Interaction adapters are inspired by Opa5 [press](https://openui5.hana.ondemand.com/#/api/sap.ui.test.actions.Press) adapters. You specify an adapter by using the `interaction` property of the by.control object.

The interaction can be any one of: `root`, `focus`, `press`, `auto` (default), and `{idSuffix: "myIDsuffix"}`.

Located element for each case:
* root: the root DOM element of the control
* focus: the DOM element that typically gets the focus
* press: the DOM element that gets the `press` events, as determined by OPA5
* auto: the DOM element that receives events, as determined by OPA5. It searches for special elements with the following priority: press, focus, root.
* {idSuffix: "myIDsuffix"}: child of the control DOM reference with ID ending in "myIDsuffix"

One common use case for changing the adapter is locating search fields:
```javascript
var searchInput = element(by.control({
  controlType: "sap.m.SearchField",
  interaction: "focus"
}); // will locate the input field
var searchPress = element(by.control({
  controlType: "sap.m.SearchField",
  interaction: "press"
}); // will locate the search button (magnifier)
```

Another use case is for controls, such as `ObjectIdentifier` or `ObjectAttribute` that can have different appearance and OPA interaction adapters. The default `auto` interaction uses the interaction adapter to find the DOM.

If the control is not in the expected apearance and due to the hard-coded interaction adapter type order, it is possible that the search to fail with a message, such as: _INFO: Expectation FAILED: Failed: unknown error: Control Element sap.m.ObjectAttribute#\_\_attribute0 has no dom representation idSuffix was text_.

In this case, you need to override the intercation type and search for a focused element:
```javascript
var objectAttributeText = element(by.control({
  controlType: "sap.m.ObjectAttribute",
  interaction: "focus",
  properties: [{
    title: "Freight RFQ"
  }]
})); // will locate the text inside this ObjectAttribute
```

### Control Ancestors
When you use a control locator to find a child element of a specific parent, IUVeri5 applies the [ancestor matcher](https://openui5.hana.ondemand.com/#/api/sap.ui.test.matchers.Ancestor) to the child. The ID of the parent element is used to find child controls. For example, if the parent is a header bar, its control's root element is the header. If you then search for child elements, the other header bars might match as well.

Example:
```javascript
element(by.id("page1-intHeader-BarLeft")) // can be any locator
  .element(by.control({
    controlType: "sap.m.Button"
  })); // will look for buttons in the header
```

## DOM Locators
All standart `by.` locators from [WebDriverJs](https://seleniumhq.github.io/selenium/docs/api/javascript/module/selenium-webdriver/index_exports_By.html) are supported.

## JQuery Locators
SAPUI5 runtime includes and uses jQuery to bridge its power to application tests.

All [jQuery selectors](https://api.jquery.com/category/selectors/) are available, including the powerful pseudo-selectors.

Select an element by jQuery expression:
```javascript
element(by.jq('<jguery expression>'));
```

### Select an Element that Contains a Specific Child
Sometimes, it is useful to have a backward selectors, for example, to select the parent of an element with specific properties.

This is easily achieved with the [:has()](https://api.jquery.com/has-selector/) jQuery pseudo-selector.

Select a tile from SAP Fiori Launchpad:
```javascript
element(by.jq('.sapUshellTile:has(\'.sapMText:contains(\"Track Purchase Order\")\')'))
```

### Select an Element from a List
`ElementArrayFinder` that is returned from `element.all()` has a `.get(<index>)` method that returns
an element by its index. However, chaining several levels of `.get()` can slowdown the test execution as every
interaction requires a browser roundtrip. Additionally, the whole expression becomes cumbersome and hard to read.

It is more simple to use the [:eq()](https://api.jquery.com/eq-selector/) jQuery pseudo-selector.
```javascript
element(by.jq('.sapMList > ul > li:eq(1)')),
```
