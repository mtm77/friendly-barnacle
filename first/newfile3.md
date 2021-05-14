---
title: Using Context with React classes
date: 2019-10-09
type: code
---

In my opinion, [Context](https://reactjs.org/docs/context.html) is the cleanest way to handle state in small React applications. It's simpler than [Redux](https://redux.js.org/) and [MobX](https://mobx.js.org/README.html); there are no node modules to install, and it doesn't require you to eject your Create React App.

Put simply, Context gives you an easy way to manage state across your entire application _without_ passing props through long chains of components (or setting up another state management solution). In this post, I'll share a simple and clean way to set it up. Let's get started.

1. Create a _contexts_ folder (I put mine at a high level, like inside _src_).

2. Inside that folder, create two files called _DataContext.js_ and _DataProvider.jsx_; if this context is going to handle a very specific part of your application (say, themes) you could use a different name such as _ThemeContext_. But to keep this tutorial simple, we'll just use _DataContext_. Add the following to each of these files:

	```js
	// DataContext.js

	import React from 'react'

	const DataContext = React.createContext()

	export default DataContext
	```

	<aside>

	Context comes out-of-the-box with React, so we don't need to install any special node modules to use it. We just need this one file so we can reference our context in other components.

	</aside>

	```js
	// DataProvider.jsx

	import React, { Component } from 'react'
	import DataContext from './DataContext'

	class DataProvider extends Component {
		// We'll use this component's state to hold our context. We'll
		// also add an `update` function to the state that will allow
		// us to update the state from anywhere in our application by
		// calling this.context.update().
		constructor(props) {
			super(props)
			this.updateState = this.updateState.bind(this)
			this.state = {
				x: 0,
				y: 0,
				update: this.updateState
			}
		}

		// Call `this.context.update({ key: value })` from a consumer
		// to update this state.
		updateState(values) {
			this.setState(values)
		}

		// Wrap the children in DataContext.Provider and pass in the
		// state as a value.
		render() {
			return (
				<DataContext.Provider value={this.state}>
					{this.props.children}
				</DataContext.Provider>
			)
		}
	}

	export default DataProvider
	```

	<aside>

	This component's state will act as a "data store". This is where we can put values to access via context. We'll also pass in an `update` function that lets us update the context from grandchild components. (This'll all make sense soon – keep going.)

	</aside>

3. Find a high level component, such as App.js, and wrap its contents in the DataProvider.

	```js
	// App.js (or any high-level component)

	import React from 'react'
	import DataProvider from '../contexts/DataProvider'

	const App = () => <DataProvider>{/* App Contents */}</DataProvider>

	export default App
	```

4. Now, any children (or grandchildren, or great-grandchildren, and so on) of this component can access Context. Let's say that a few levels further, we have a `Grandchild` component that needs to access `x` and `y` from our context. We can tell the component to access context like this (look at the comments):

	```js
	// Grandchild.jsx

	import React, { Component } from 'react'
	import DataContext from '../../contexts/DataContext' //  1. Import DataContext

	class Grandchild extends Component {
		render() {
			return (
				<div>
					{this.context.x}, {this.context.y}
				</div>
			)
		}
	}

	Grandchild.contextType = DataContext // 2. Set the contextType before exporting the component

	export default Grandchild
	```

5. Now, what if we want to update `x` and `y` from Grandchild? Without Context, we would need to pass some kind of function down the tree of components... but thanks to our setup from Step 2, all we need to do is call `this.context.update()`. Let's keep it really simple and set `x` and `y` to 100 when the user clicks the component.

	```js
	// Grandchild.jsx, revisited

	import React, { Component } from 'react'
	import DataContext from '../../contexts/DataContext'

	class Grandchild extends Component {
		// Update the Context. If you recall from Step 2, this is
		// actually just calling this.setState() on our Provider!
		handleClick() {
			this.context.update({ x: 100, y: 100 })
		}

		// Call `handleClick` when the user clicks the component.
		render() {
			return (
				<div onClick={() => this.handleClick()}>
					{this.context.x}, {this.context.y}
				</div>
			)
		}
	}

	Grandchild.contextType = DataContext

	export default Grandchild
	```

	<aside>

	**Beginner tip:** Remember to import `DataContext` and set your component's `contextType` on any component that needs to use context. Otherwise, you'll get an error.

	</aside>

And there you have it: a simple introduction to using React's Context. It's really that simple.
