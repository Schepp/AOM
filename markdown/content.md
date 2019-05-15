# The Accessibility Object Model (AOM)
---
# Goals
---
## Goal: Better programming interface for ARIA
---
### Reflecting ARIA attributes

```js
el.role = "button";
el.ariaPressed = "true";  // aria-pressed is a tristate attribute
el.ariaDisabled = true;   // aria-disabled is a true/false attribute
el.ariaDescribedBy = "id1";
```
---
### Non-Reflected Element References

```js
el.ariaDescribedByElements = [labelElement1, labelElement2];
el.ariaActiveDescendantElement = ownedElement1;
```

<p class="fragment">Helpful in cases where one had to reference countless element ids</p>

<p class="fragment">Helpful in cases where one's CMS has trouble generating ids</p>

<p class="fragment">Enables authors using open ShadowRoots to specify relationships which cross over Shadow DOM boundaries.</p>
---
This does not work as ids are scoped:
```html
<custom-combobox>
  #shadow-root (open)
  |  <input aria-activedescendant="opt1"></input>
  |  <slot></slot>
  <custom-optionlist>
    <x-option id="opt1">Option 1</x-option>
    <x-option id="opt2">Option 2</x-option>
    <x-option id='opt3'>Option 3</x-option>
 </custom-optionlist>
</custom-combobox>
```
---
Solution via AOM:

```html
<custom-combobox>
  #shadow-root (open)
  |  <input>
  |  <slot></slot>
  <custom-optionlist>
    <x-option id="opt1">Option 1</x-option>
    <x-option id="opt2">Option 2</x-option>
    <x-option id='opt3'>Option 3</x-option>
 </custom-optionlist>
</custom-combobox>
```

```js
const input = comboBox.shadowRoot.querySelector("input");
const optionList = comboBox.querySelector("custom-optionlist");
input.activeDescendantElement = optionList.firstChild;
```
---
## Goal: Being able to configure default accessibility properties for Web Components

<p class="fragment">This capability need not, but may be limited to Web Components.</p>
---
Today people using Web Components need to set aria attributes, as they are not baked into the component's definition:

```html
<custom-tablist role="tablist">
  <custom-tab selected role="tab" aria-selected="true" aria-controls="tabpanel-1">Tab 1</custom-tab>
  <custom-tab role="tab" aria-controls="tabpanel-2">Tab 2</custom-tab>
  <custom-tab role="tab" aria-controle="tabpanel-3">Tab 3</custom-tab>
</custom-tablist>
```
---
Instead authors should be able to provide static, default semantics via `customElements.define()`:

```js
class TabListElement extends HTMLElement { ... }

customElements.define(
    "custom-tablist", 
    TabListElement,
    { 
        role: "tablist", 
        ariaOrientation: "horizontal" 
    }
);
```
---
Authors using the component could still override the default in the standard manner:

```html
<custom-tablist aria-orientation="vertical">
  <custom-tab selected>Tab 1</custom-tab>
  <custom-tab>Tab 2</custom-tab>
  <custom-tab>Tab 3</custom-tab>
</div>
```
---
Also, a custom element author may use the `ElementInternals` object to modify the semantic state of a group of elements in response to user interaction via a callback, to keep related UI in sync.

`custom-tab` would inform/adjust `custom-tablist`, which would then again inform all other tabs:

```html
<custom-tablist>
  <custom-tab selected>Tab 1</custom-tab>
  <custom-tab>Tab 2</custom-tab>
  <custom-tab>Tab 3</custom-tab>
</div>
```
---
## Listening for events from Assistive Technology

<p class="fragment">Potentially the bad part</p>
---
Right now, AT events get mapped to DOM events, like so:

<table>
<thead>
<tr>
<th><strong>AT event</strong></th>
<th><strong>Targets</strong></th>
<th><strong>DOM event</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td><code>click</code></td>
<td><em>all</em></td>
<td><code>click</code></td>
</tr>
<tr>
<td><code>focus</code></td>
<td><em>all elements</em></td>
<td><code>focus</code></td>
</tr>
<tr>
<td><code>select</code></td>
<td><code>cell</code><br><code>option</code></td>
<td><code>click</code></td>
</tr>
<tr>
<td><code>scrollIntoView</code></td>
<td>(n/a)</td>
<td>No event</td>
</tr>
<tr>
<td><code>dismiss</code></td>
<td><em>all elements</em></td>
<td>Keypress for <code>Escape</code> key</td>
</tr>
<tr>
<td><code>contextMenu</code></td>
<td><em>all elements</em></td>
<td><code>contextmenu</code></td>
</tr>
<tr>
<td><code>scrollByPage</code></td>
<td><em>all elements</em></td>
<td>Keypress for <code>PageUp</code><br><code>PageDown</code> key</td>
</tr>
<tr>
<td><code>increment</code></td>
<td><code>progressbar</code><br><code>scrollbar</code><br><code>slider</code><br><code>spinbutton</code></td>
<td>Keypress for <code>Up</code> key</td>
</tr>
<tr>
<td><code>decrement</code></td>
<td><code>progressbar</code><br><code>scrollbar</code><br><code>slider</code><br><code>spinbutton</code></td>
<td>Keypress for <code>Down</code> key</td>
</tr>
<tr>
<td><code>setValue</code></td>
<td><code>combobox</code>,<code>scrollbar</code>,<code>slider</code><br><code>textbox</code></td>
<td>TBD</td>
</tr>
</tbody>
</table>
---
There was once a plan to add these events:

* accessibleclick
* accessiblecontextmenu
* accessiblefocus
* accessiblescrollintoview
* accessibleincrement
* accessibledecrement
* accessibledismiss
* accessiblesetvalue
---
<!-- .slide: data-background="images/digital-apartheid.png" data-state="inverted" -->
---
New events \*really\* being added:

* increment
* decrement
* dismiss
* scrollPageUp
* scrollPageDown

<p class="fragment">Heuristic will be added to non-AT user's events to have them also trigger these, not just AT-users</p>
---
## Adding virtual nodes to the Accessibility tree

For use cases like interacting with HTML5 Canvas or when you stream a remote user interfaces via HTML5 Video element.
---
Possibility to create a virtual accessibility tree via `attachAccessibleRoot()`.

```js
// Implementing a canvas-based spreadsheet's semantics
canvas.attachAccessibleRoot();
let table = canvas.accessibleRoot
                .appendChild(new AccessibleNode());
table.role = 'table';
table.colCount = 10;
table.rowcount = 100;
let headerRow = table.appendChild(new AccessibleNode());
headerRow.role = 'row';
headerRow.rowindex = 0;
// etc. etc.
```
---
## 5. In(-tro-)specting the computed accessibility tree

Being able to check back the resulting accesibility tree via `ComputedAccessibleNode`
---
### Problems here:

Currently, the accessibility tree is not standardized between browsers: Each implements accessibility tree computation slightly differently. In order for this API to be useful, it needs to work consistently across browsers, so that developers don't need to write special case code for each.

Computing the value of many accessible properties requires layout. Allowing web authors to query the computed value of an accessible property synchronously via a simple property access would introduce confusing performance bottlenecks.
---
<!-- .slide: data-background="images/thankyoupage.jpg" data-state="inverted" -->

<br><br><br><br><br><br>
# That's about it! Thank you :)

* Slides: [http://schepp.github.io/AOM](http://schepp.github.io/AOM)
* Twitter: [@derSchepp](https://twitter.com/derSchepp)
* Podcast: [Working Draft](http://workingdraft.de)
