# Reply to the `adoptedstylesheets` attribute in the template tag

The current standard proposal is very promising and even enough I have some remarks to make, I love the direction this is going. In this
reply I will not go into CSS modules, importmaps or how to register a stylesheet using a specifier at all. I will go into how this is going
to be consumed.

## Will we get a `template` soup, especially when using a ui-framework?

Let's take an example of this custom element I have recently been working on: a discussion feed with messages and the possibility to reply. It
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
            <p></p>
            <p></p>
          </template>
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
        <fw-button commandfor="message-1234" command="like">
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
elements require a `<template>` tag with a required `shadowrootmode="open"` and a `shadowrootadoptedstylesheets` to make sure it is styled.

## Complexity at the server side

When rendering a document mentioned, a discussion for backend team might be: should we encapsulate the usage of our custom elements framework? Why would we want to 
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

So, what will inevitibly happens is the creation of a custom elements registry at the server side level. Not only for elements created by the development teams,
but also for the elements that are being consumed.

