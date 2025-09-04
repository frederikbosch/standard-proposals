# Reply to the proposal of the `shadowrootadoptedstylesheets` attribute in the `<template>` tag

The current standard proposal is very promising and even though I have some remarks, I love the direction this is going. In this
reply I will not go into CSS modules, importmaps or how to register a stylesheet using a specifier. I will go into how stylesheets
are adopted.

## Problems with consumption by the `<template>` tag

Considering my experience in designing many custom elements, ones that are created both declaratively inside a server side application and 
dynamically inside web applications, I am seeing at the least three problems with [the current proposal](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ShadowDOM/explainer.md).

1. [Rendering in both dynamic and static contexts](#problem-1)
1. [`<template>` soup](#problem-2)
1. Requirement of serverside custom elements registries for all elements

<a name="problem-1"></a> 
### Problem 1: declarative and dynamic contexts: is my element ready to be displayed?

This problem is only applicable to custom elements and not to Shadow DOMs linked to non-custom elements. Consider the following avatar custom 
element. Its shadow DOM is created declaratively.

```html
<style type="module" specifier="fw-avatar">...</style>

<fw-avatar>
  <template shadowrootmode="open" shadowrootadoptedstylesheets="fw-avatar">
    <img src="" alt="">
  </template>
</fw-avatar>
```

This element is also registered in the custom elements registry.

```js
import FwAvatar from 'fw/avatar.js';

customElements.define('fw-avatar', FwAvatar);
```

Suppose, the user clicks somewhere we do a `fetch()` call and using the response data we render a new list of people with their avatar.

```js
for (const person of people) {
  const avatar = document.createElement('fw-avatar');
}
```

Now, this creates a problem. When `fw-avatar` was parsed by the browser inside the response document, the `shadowrootadoptedstylesheets` attribute
created the `CSSStyleSheet` and adopted it into the [`host.shadowRoot`](https://developer.mozilla.org/en-US/docs/Web/API/ShadowRoot/adoptedStyleSheets).
But now that the element is created dynamically in Javascript we need a second solution, one the second solution needs to verify the first solution is not
applicable. This *could* look like this.

```js
// import a reference to the CSSStyleSheet available through <style specifier="fw-avatar">, which is already inside the document
import styles from 'fw-avatar' with { type: 'css' };

export class FwAvatar extends HTMLElement {
  connstructor() {
    if (!this.shadowRoot) {
      this.attachShadow({mode: 'open'});
    }

    // do not load the stylesheet when it has already been loaded declaratively
    if (!this.shadowRoot.adoptedStyleSheets.includes(styles)) {
      this.shadowRoot.adoptedStyleSheets.push(styles);
    }
  }
}
```

This means that, in case elements are created both declaratively and dynamically, there have to be, at least, two places with knowledge 
of how to style such an element. The first place is where HTTP response documents are generated and secondly, at the 
element itself. Is this inevitable when an element has to be rendered both declaratively and dynamically?

Moreover, the suggested solution is flawed. Suppose we want to change the adopted stylesheets into only `my-fw-avatar`? In a declarative
context this is easy, you simply change the `shadowrootadoptedstylesheets` attribute.

```html
<style type="module" specifier="my-fw-avatar">...</style>

<fw-avatar>
  <template shadowrootmode="open" shadowrootadoptedstylesheets="my-fw-avatar">
    <img src="" alt="">
  </template>
</fw-avatar>
```

The dynamic context is way harder. If the class `FwAvatar` is coming from an external library, and its contents cannot be touched inside the current context of the developer, 
the creator of the element - the one calling `document.createElement` - has to do an additional call.

```js
// import our own styles
import myStyles from 'my-fw-avatar' with { type: 'css' };

for (const person of people) {
  const avatar = document.createElement('fw-avatar');
  // overwrite with our own styles
  avatar.shadowRoot.adoptedStyleSheets = [myStyles];
}
```

This works, but only when the `FwAvatar` class does not add its own stylesheet. When it would add its own stylesheet, and it is likely that a ui framework comes with its own/default
stylesheet, this requires **both** the framework and the overwrite styles to be inside the original response document. Why? Because if the `fw-avatar` specifier is missing, the 
following line inside the `FwAvatar` class will throw an error.

```js
import styles from 'fw-avatar' with { type: 'css' };
```

So, if we want an optimal solution, we must change the `FwAvatar` class as such that the `fw-avatar` does not contain a stylesheet by default or this default stylesheet becomes 
conditional. However,  a conditional stylesheet generates another problem, the import has to become a dynamic `import()` call or the raw css should also be embedded inside the custom element.

<a name="problem-2"></a> 
### Problem 2: will we get a `<template>` soup, especially when using a ui-framework?

Let's take an example of a custom element I have recently been working on: a discussion feed with messages and the possibility to reply. It
might be a requirement that such a feed be rendered declaratively. Reasons for such a requirement are discussed in another Github thread and
therefore not given here. With the current proposals the response document might look like this.

```html
<style type="module" specifier="discussion-feed">...</style>
<style type="module" specifier="discussion-message">...</style>
<style type="module" specifier="discussion-reply">...</style>
<style type="module" specifier="discussion-content">...</style>
<style type="module" specifier="discussion-likes">...</style>
<style type="module" specifier="fw-avatar">...</style>
<style type="module" specifier="fw-button">...</style>
<style type="module" specifier="fw-dialog">...</style>

<discussion-feed id="feed">
  <template shadowrootmode="open" shadowrootadoptedstylesheets="discussion-feed">
    <discussion-message id="message-1234">
      <template shadowrootmode="open" shadowrootadoptedstylesheets="discussion-message">
        <h1>Declarative CSS Modules and Declarative Shadow DOM adoptedstylesheets attribute</h1>
        <header>
          <fw-avatar>
            <template shadowrootmode="open" shadowrootadoptedstylesheets="fw-avatar">
              <img src="" alt="KurtCattiSchmidt" />
            </template>
          </fw-avatar>
          <p class="author>KurtCattiSchmidt</p>
          <time timestamp="">Oct 2, 2024</time>
        </header>
        <discussion-content>
          <template shadowrootmode="open" shadowrootadoptedstylesheets="discussion-content">
            <slot></slot>
          </template>
          <p></p>
          <p></p>
        </discussion-content>
        <footer>
          <discussion-likes>
            <template shadowrootmode="open" shadowrootadoptedstylesheets="discussion-likes">
              <fw-button>
                <template shadowrootmode="open" shadowrootadoptedstylesheets="fw-button">
                  <button commandfor="message-1234" command="like">Create reply</button>
                </template>
              </fw-button>
              <div class="number">
                <slot>
                </slot>
              </div>
              5
            </template>
          </discussion-likes>
        </footer>
        <fw-button>
          <template shadowrootmode="open" shadowrootadoptedstylesheets="fw-button">
            <button commandfor="feed" command="reply">Create reply</button>
          </template>
        </fw-button>
        <fw-dialog>
          <template shadowrootmode="open" shadowrootadoptedstylesheets="fw-dialog">
            <dialog>
              <slot></slot>
            </dialog>
          </template>
          <discussion-reply>
            <form>
              ...
            </form>
          </discussion-reply>
        </fw-dialog>
      </template>
    </discussion-message>

    {{#each replies as |reply|}}
      <discussion-message id="message-1235">
        ...
      </discussion-message>
    {{/each}}
  </template>
</discussion-feed>
```

Such an example is becoming a very verbose. Especially when someone would like to use a custom-element framework like [shoelace](https://shoelace.style). All these 
elements would require a `<template>` tag with a required `shadowrootmode="open"` and a `shadowrootadoptedstylesheets` to make sure it is styled. The above also
applies to shadow roots attached to non-custom elements.

### Problem 3: complexity at the server side

When rendering the document mentioned above, a discussion for backend team might be: should we encapsulate the usage of our custom elements framework? Why would we want to 
write this in our server side templates?

```html
<fw-avatar>
  <template shadowrootmode="open" shadowrootadoptedstylesheets="fw-avatar">
    <img src="" alt="">
  </template>
</fw-avatar>
```

When we could write this?

```hbs
{{fw-avatar src="" name=""}}
```

So, what will inevitably happen is the creation of a custom elements registry at the server side level. Not only for elements created by the development teams,
but also for the elements that are being consumed from external libraries.

## Solution, consumption at the host

As I have already been suggesting in the reply to the [TAG design review request](https://github.com/w3ctag/design-reviews/issues/1000), I think modules should be consumed 
at the host element, not by the template tag. To take the example of the `fw-avatar`.

```html
<style type="module" specifier="fw-avatar">img { border-radius: 100%; }</style>
<fw-avatar host-for="fw-avatar">
  <template shadowrootmode="open">
    <img src="" alt="">
  </template>
</fw-avatar>
```

This would also work for shadow roots attached to other elements.

```html
<style type="module" specifier="my-avatar">img { border-radius: 100%; }</style>
<div host-for="my-avatar">
  <template shadowrootmode="open">
    <img src="" alt="">
  </template>
</fw-avatar>
```

Now, as a rule, we could say that the `host-for` attribute equals tag name, only in case of custom elements. This changes the first example into the following, and the image would
still have its `border-radius`.

```html
<style type="module" specifier="fw-avatar">img { border-radius: 100%; }</style>
<fw-avatar>
  <template shadowrootmode="open">
    <img src="" alt="">
  </template>
</fw-avatar>
```

The last suggestion also solves problem 1 and 2. When `fw-avatar` is created dynamically, it automatically adopts the `fw-avatar` stylesheet because its `host-for` attribute by 
default equals to its tag name. The framework designer can leave out the css in their component and supply default stylesheet with their library.

Now if we would want to change the stylesheet into `my-fw-avatar`, we could decide to change the `host-for` attribute both contexts.

```html
<style type="module" specifier="my-fw-avatar">img { border-radius: 100%; }</style>
<fw-avatar host-for="my-fw-avatar">
  <template shadowrootmode="open">
    <img src="" alt="">
  </template>
</fw-avatar>
```

```js
document.createElement('fw-avatar', {
    hostFor: 'my-fw-avatar'
});
```

Hence, there is no requirement to change the `FwAvatar` class. But don't we now have to change our complete codebase to add this `host-for` attribute? There is another solution, 
if we combine it with the `@sheet` proposal as is also done by the current proposal.

```html
<style type="module" specifier="framework">
  @sheet fw_avatar {
    img { border-radius: 100%; }
  }
</style>

<style type="module" specifier="fw-avatar">
  @import("framework/fw-avatar");
</style>

<fw-avatar>
  <template shadowrootmode="open">
    <img src="" alt="">
  </template>
</fw-avatar>
```

## Future Work

Just as is suggested in the current proposal, we could extend the `host-for` attribute to host more than just stylesheets. I saw some discussions on
declarative templating. I think we'd have to consider this possibility.

```html
<template type="module" specifier="fw-avatar">
  <img src="{{@src}}" alt="{{@name}}">
</template>

<fw-avatar src="me.jpg" name="My Name">
</fw-avatar>
```

```js
const avatar = document.createElement('fw-avatar', {
  attributes: {
    src: 'me.jpg',
    name: 'My Name',
  }
});
avatar.setAttribute('src', 'me.jpg');
```

Or, even better, combined with a `<definition>` suggestion I have seen somewhere (cannot recall where).

```html
<definition specifier="fw-avatar">
  <!-- definition can only contain modules, no module attribute required -->
  <template>
    <img src="{{@src}}" alt="{{@name}}">
  </template>
  <style>img { border-radius: 100%; }</style>
</definition>

<fw-avatar src="me.jpg" name="My Name"></fw-avatar>
```

Now let's regenerate the source code of our discussion feed.

```html
<definition specifier="discussion-feed">...</definition>
<definition specifier="discussion-message">...</definition>
<definition specifier="discussion-reply">...</definition>
<definition specifier="discussion-content">...</definition>
<definition specifier="discussion-likes">...</definition>
<definition specifier="fw-avatar">...</definition>
<definition specifier="fw-button">...</definition>
<definition specifier="fw-dialog">...</definition>
<!-- maybe packaged together? -->
<link rel="definition" href="definitions.html" blocking="render">

<discussion-feed id="feed">
  <discussion-message id="message-1234">
    <h1>Declarative CSS Modules and Declarative Shadow DOM adoptedstylesheets attribute</h1>
    <header>
      <fw-avatar src="me.jpg" alt="KurtCattiSchmidt"></fw-avatar>
      <p class="author>KurtCattiSchmidt</p>
      <time timestamp="">Oct 2, 2024</time>
    </header>
    <discussion-content>
      <p></p>
      <p></p>
    </discussion-content>
    <footer>
      <discussion-likes number="5">
        <fw-button commandfor="message-1234" command="like">Like</fw-button>
      </discussion-likes>
    </footer>
    <fw-button commandfor="message-1234" command="like">Create reply</fw-button>
    <fw-dialog>
      <discussion-reply>
        <form>
          ...
        </form>
      </discussion-reply>
    </fw-dialog>
  </discussion-message>

  {{#each replies as |reply|}}
    <discussion-message id="message-1235">
      ...
    </discussion-message>
  {{/each}}
</discussion-feed>
```

Because of the `host-for` attribute, and because a custom element automatically has a `host-for` attribute that equals its tag name and with 
a possible `<definition>` tag in the future, we can move all `<template>` declarations inside this `<definition>` tag. Now would a backend team prefer to write this
`{{fw-avatar src="" alt=""}}` over `<fw-avatar src="" alt=""></fw-avatar>`? I don think they won't have any preference at all and would therefore not even think 
to encapsulate this markup.

We could go much furher then. Why would we want to extend elements like `<button>` now? There is no need! We can use composition and pass the `commandfor` and `command` 
attributes to the native `<button>` element. 

```html
<definition specifier="fw-button">
  <template>
    <button ?commandfor="{{@commandfor}}" ?command="{{@command}}">
      <slot></slot>
    </template>
  </template>
  <style>button { }</style>
</definition>

<fw-button commandfor="message-1234" command="like"></fw-button>
```

If we would ever get there, this would, among many others, generate a question. Is every client, like a search-engine, that receives such a HTTP response able to read such a 
markup? Whatever the answer to that question is, using the `<template>` tag has become optional. They are not a requirement anymore for declarative custom elements
in the `<body>` of our page, and this starts with the `host-for` attribute.
