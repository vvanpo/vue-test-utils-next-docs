# A Crash Course

Let's jump write into it, and learn Vue Test Utils (VTU) by building a simple Todo app, and writing tests as we go. This guide will cover:

- mounting components
- finding elements
- filling out forms
- triggering events

A working repository with this example can be found [here](https://github.com/lmiller1990/vtu-next-demo). The component file is [here](https://github.com/lmiller1990/vtu-next-demo/blob/master/src/TodoApp.vue) and the the test is [here](https://github.com/lmiller1990/vtu-next-demo/blob/master/src/TodoApp.spec.js).

## Getting Started

We will start off with a simple `TodoApp` component with a single todo:

```vue
<template>
  <div>
  </div>
</template>

<script>
export default {
  name: 'TodoApp',

  data() {
    return {
      todos: [
        {
          id: 1,
          text: 'Learn Vue.js 3',
          completed: false
        }
      ]
    }
  }
}
</script>
```

## The first test - a todo is rendered

The first test we will write verifies a todo is rendered. Let's see the test first, then discuss each part:

```js
import { mount } from '@vue/test-utils-next'

import TodoApp from './TodoApp.vue'

test('renders a todo', () => {
  const wrapper = mount(TodoApp)

  const todo = wrapper.find('[data-test="todo"]')

  expect(todo.text()).toBe('Learn Vue.js 3')
})
```

We start off by importing `mount` - this is the main way to render a component in VTU. You declare a test by using the `test` function with a short description of the test. The `test` and `expect` functions are globally available in most test runners (this example uses [Jest](https://jestjs.io/en/)). If `test` and `expect` look confusing, the Jest documentation has a [more simple example](https://jestjs.io/docs/en/getting-started) of how to use them and how they work.

Next, we call `mount` and pass the component as the first argument - this is something almost every test you write will do. By convention, we assign the result to a variable called `wrapper`, since `mount` provides a simple "wrapper" around the app with some convinient methods for testing.

Finally, we use another global function common to many tests runner - Jest included - `expect`. The idea is we are asserting, or *expecting*, the actual output to match what we think it should be. In this case, we are finding an element with the selector `data-test="todo"` - in the DOM, this will look like `<div data-test="todo">...</div>`. We then call the `text` method to get the content, which we expect to be `'Learn Vue.js 3'`.

> Using `data-test` selectors is not required, but it can make your tests less brittle. classes and ids tend to change or move around as an application grows - by using `data-test`, it's clear to other developers which elements are used in tests, and should not be changed.

## Making the test pass

If we run this test now, it fails with the following error message: `Cannot call text on an empty wrapper`. That's because we aren't rendering any todos, so the `find` call is failing to return a wrapper (remember, VTU wraps all components, and DOM elements, in a "wrapper" with some convinient methods). Let's update `<template>` in `TodoApp.vue` to render the `todos` array:

```vue
<template>
  <div>
    <div 
      v-for="todo in todos" 
      :key="todo.id" 
      data-test="todo"
    >
      {{ todo.text }}
    </div>
  </div>
</template>
```

With this change, the test is passing. Congratulations! You wrote your first component test.

## Adding a new todo

The next feature we will be adding is for the user to be able to create a new todo. To do so, we need a form with an input for the user to type some text. When the user submits the form, we expect the new todo to be rendered. Let's take a look at the test:

```js
test('creates a todo', () => {
  const wrapper = mount(TodoApp)
  expect(wrapper.findAll('[data-test="todo"]')).toHaveLength(1)

  wrapper.find('[data-test="new-todo"]').element.value = 'New todo'
  wrapper.find('[data-test="form"]').trigger('submit')

  expect(wrapper.findAll('[data-test="todo"]')).toHaveLength(2)
})
```

As usual, we start of by using `mount` to render the element. We are also asseting that only 1 todo is rendered - this makes it clear that we are adding an additional todo, as the final line of the test suggests. 

To update the `<input>`, we use `element` - this accesses the original DOM element wrapper, which is returned by `find`. `element` is useful to manipulate a DOM element in a manner that VTU does not provide any methods for.

After updating the `<input>`, we use the `trigger` method to simulate the user submitting the form. Finally, we assert the number of todos has increased from 1 to 2. 

If we run this test, it will obviously fail. Let's update `TodoApp.vue` to have the `<form>` and `<input>` elements and make the test pass:

```vue
<template>
  <div>
    <div 
      v-for="todo in todos" 
      :key="todo.id" 
      data-test="todo"
    >
      {{ todo.text }}
    </div>

    <form data-test="form" @submit.prevent="createTodo">
      <input data-test="new-todo" v-model="newTodo" />
    </form>
  </div>
</template>

<script>
export default {
  name: 'TodoApp',

  data() {
    return {
      newTodo: '',
      todos: [
        {
          id: 1,
          text: 'Learn Vue.js 3',
          completed: false
        }
      ]
    }
  },

  methods: {
    createTodo() {
      this.todos.push({
        id: 2,
        text: this.newTodo,
        completed: false
      })
    }
  }
}
</script>
```

We are using `v-model` to bind to the `<input>` and `@submit` to listen for the form submission. When the form is submitted, `createTodo` is called and inserts a new todo into the `todos` array.

While this looks good, running the test shows an error:

```
expect(received).toHaveLength(expected)

    Expected length: 2
    Received length: 1
    Received array:  [{"element": <div data-test="todo">Learn Vue.js 3</div>}]
```

The number of todos has not increased. The problem is that Jest executes tests in a synchronous manner, ending the test as soon as the final function is called. Vue, however, updates the DOM asynchronously. We need to mark the test `async`, and call `await` on any methods that might cause the DOM to change. `trigger` is one such method - we can simply prepend `await` to the `trigger` call:

```js
test('creates a todo', async () => {
  const wrapper = mount(TodoApp)

  wrapper.find('[data-test="new-todo"]').element.value = 'New todo'
  await wrapper.find('[data-test="form"]').trigger('submit')

  expect(wrapper.findAll('[data-test="todo"]')).toHaveLength(2)
})
```

Now the test is finally passing!

## Completing a todo

Now that we can create todos, let's give the user the ability to mark a todo item as completed/uncompleted with a checkbox. As previously, let's start with the failing test:

```js
test('completes a todo', async () => {
  const wrapper = mount(TodoApp)

  await wrapper.find('[data-test="todo-checkbox"]').setChecked()

  expect(wrapper.find('[data-test="todo"]').classes()).toContain('completed')
})
```

This test is similar to the previous two; we find an element, interact with in in same way (in this test we use `setChecked`, since we are interacting with a `<input type="checkbox">`. Again, since the DOM will be changing (the checkbox `checked` value will be updated) we need to prepand `await` to the interaction. 

Lastly, we make an assertion. We will be applying a `completed` class to completed todos - we can then use this to add some styling to visually indicate the status of a todo.

We can get this test to pass by updating the `<template>` to include the `<input type="checkbox">` and a class binding on the todo element:

```vue
<template>
  <div>
    <div 
      v-for="todo in todos" 
      :key="todo.id" 
      data-test="todo"
      :class="[ todo.completed ? 'completed' : '' ]"  
    >
      {{ todo.text }}
      <input 
        type="checkbox" 
        v-model="todo.completed" 
        data-test="todo-checkbox" 
      />
    </div>

    <form data-test="form" @submit.prevent="createTodo">
      <input data-test="new-todo" v-model="newTodo" />
    </form>
  </div>
</template>
```

Congratulations! You wrote your first component tests.

## Arrange, Act, Assert

You may have noticed some new lines between the code in each of the tests. Let's look at the second test again, in detail:

```js
test('creates a todo', async () => {
  const wrapper = mount(TodoApp)

  wrapper.find('[data-test="new-todo"]').element.value = 'New todo'
  await wrapper.find('[data-test="form"]').trigger('submit')

  expect(wrapper.findAll('[data-test="todo"]')).toHaveLength(2)
})
```

The test is split into three distinct stages, separated by new lines. The three stages represent the three phases of a test: **arrange**, **act** and **assert**.

In the *arrange* phase, we are setting up the scenario for the test. A more complex example may require creating a Vuex store, or populating a database.

In the *act* phase, we act out the scenario, simulating how a user would interact with the component or application.

In the *assert* phase, we make assertions about how we expect the current state of the component to be. 

Almost all test will follow these three phases. You don't need to separate them with new lines like this guide does, but it is good to keep these three phases in mind as you write your tests.

## Conclusion

This guide demonstrates the basics of Vue Test Utils, including:

- `mount` to render a component
- `find` and `findAll` to query the DOM
- `trigger` and `setChecked` to simulate user input
- `async` and `await` to ensure the DOM is rerendered prior to an assertion
- the three phases of testing; act, arrange and assert