---
id: original
title: Extracting the original state from a draft
sidebar_label: Original
---

<center>
<div data-ea-publisher="immerjs" data-ea-type="image" className="horizontal bordered"></div>
</center>

Immer exposes a named export `original` that will get the original object from the proxied instance inside `produce` (or return `undefined` for unproxied values). A good example of when this can be useful is when searching for nodes in a tree-like state using strict equality.

```js
import {original, produce} from "immer"

const baseState = {users: [{name: "Richie"}]}
const nextState = produce(baseState, draftState => {
	original(draftState.users) // is === baseState.users
})
```

Just want to know if a value is a proxied instance? Use the `isDraft` function! Note that `original` cannot be invoked on objects that aren't drafts.

```js
import {isDraft, produce} from "immer"

const baseState = {users: [{name: "Bobby"}]}
const nextState = produce(baseState, draft => {
	isDraft(draft) // => true
	isDraft(draft.users) // => true
	isDraft(draft.users[0]) // => true
})
isDraft(nextState) // => false
```
