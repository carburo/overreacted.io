---
title: La notación «Bug-O»
date: '2019-01-25'
spoiler: ¿Cuál es el 🐞(<i>n</i>) de tu API?
---

Cuando escribes código cuyo rendimiento es importante es una buena idea tener en cuenta su complejidad algorítmica. Esta se expresa a menudo con la [notación O grande](https://rob-bell.net/2009/06/a-beginners-guide-to-big-o-notation/) (*Big-O* en inglés).

O grande es una medida de **cuán lento puede ser tu código a medida que añades mas datos**. Por ejemplo, si un algoritmo de ordenamiento tiene una complejidad O(<i>n<sup>2</sup></i>), si se  ordenan 50 veces más elementos será aproximadamente 50<sup>2</sup> = 2,500 veces más lento. La notación O grande no te da un número exacto, pero te ayuda a entender como *escala* un aloritmo.

Algunos ejemplos: O(<i>n</i>), O(<i>n</i> log <i>n</i>), O(<i>n<sup>2</sup></i>), O(<i>n!</i>).


Sin embargo, **este artículo no es sobre algoritmos o rendimiento**. Se trata de las API y la depuración. Resulta que el diseño de una API involucra consideraciones muy similares.

---

Una parte significativa de nuestro tiempo se concentra en encontrar y resolver errores en nuestro código. La mayoría de los desarrolladores quisieran encontrar errores más rápido. Si bien puede ser muy satisfactorio al final, no es nada agradable pasarse el día detrás de un error cuando podrías estar concentrado en la implementación de alguna idea que tenías en tus planes.

La experiencia de la depuración influye en nuestra elección de abstracciones, bibliotecas y herramientas. Algunos diseños de API y lenguajes hacen que muchos tipos de errores no puedan ocurrir. Otros crean un sinfín de problemas. **Pero como distingues cuál es cuál.**

Muchas debates en la web acerca de las API se concentran sobre todo en la estética. Pero eso [no dice mucho](/optimized-for-change/) de cómo se siente usar una API en la práctica.

**Tengo una métrica que me ayuda a razonar sobre esto. La llamo la notación *Bug-O* (*bug* en inglés puede ser tanto un error informático como un bicho):**

<font size="40">🐞(<i>n</i>)</font>

La O grande describe cuánto se ralentiza un algoritmo a medida que crece su entrada. La *Bug-O* describe cuánto una API *te* ralentiza a medida que crece tu base de código.

---

Por ejemplo, considera este código que actualiza manualmente el DOM con operaciones imperativas como `node.appendChild()` y `node.removeChild()` y que no tiene una estructura clara:

```jsx
function trySubmit() {
  // Section 1
  let spinner = createSpinner();
  formStatus.appendChild(spinner);
  submitForm().then(() => {
  	// Section 2
    formStatus.removeChild(spinner);
    let successMessage = createSuccessMessage();
    formStatus.appendChild(successMessage);
  }).catch(error => {
  	// Section 3
    formStatus.removeChild(spinner);
    let errorMessage = createErrorMessage(error);
    let retryButton = createRetryButton();
    formStatus.appendChild(errorMessage);
    formStatus.appendChild(retryButton)
    retryButton.addEventListener('click', function() {
      // Section 4
      formStatus.removeChild(errorMessage);
      formStatus.removeChild(retryButton);
      trySubmit();
    });
  })
}
```

The problem with this code isn’t that it’s “ugly”. We’re not talking about aesthetics. **The problem is that if there is a bug in this code, I don’t know where to start looking.**

**Depending on the order in which the callbacks and events fire, there is a combinatorial explosion of the number of codepaths this program could take.** In some of them, I’ll see the right messages. In others, I’ll see multiple spinners, failure and error messages together, and possibly crashes.

This function has 4 different sections and no guarantees about their ordering. My very non-scientific calculation tells me there are 4×3×2×1 = 24 different orders in which they could run. If I add four more code segments, it’ll be 8×7×6×5×4×3×2×1 — *forty thousand* combinations. Good luck debugging that.

**In other words, the Bug-O of this approach is 🐞(<i>n!</i>)** where *n* is the number of code segments touching the DOM. Yeah, that’s a *factorial*. Of course, I’m not being very scientific here. Not all transitions are possible in practice. But on the other hand, each of these segments can run more than once. <span style="word-break: keep-all">🐞(*¯\\\_(ツ)\_/¯*)</span> might be more accurate but it’s still pretty bad. We can do better.

---

To improve the Bug-O of this code, we can limit the number of possible states and outcomes. We don't need any library to do this. It’s just a matter of enforcing some structure on our code. Here is one way we could do it:

```jsx
let currentState = {
  step: 'initial', // 'initial' | 'pending' | 'success' | 'error'
};

function trySubmit() {
  if (currentState.step === 'pending') {
    // Don't allow to submit twice
    return;
  }
  setState({ step: 'pending' });
  submitForm().then(() => {
    setState({ step: 'success' });
  }).catch(error => {
    setState({ step: 'error', error });
  });
}

function setState(nextState) {
  // Clear all existing children
  formStatus.innerHTML = '';

  currentState = nextState;
  switch (nextState.step) {
    case 'initial':
      break;
    case 'pending':
      formStatus.appendChild(spinner);
      break;
    case 'success':
      let successMessage = createSuccessMessage();
      formStatus.appendChild(successMessage);
      break;
    case 'error':
      let errorMessage = createErrorMessage(nextState.error);
      let retryButton = createRetryButton();
      formStatus.appendChild(errorMessage);
      formStatus.appendChild(retryButton);
      retryButton.addEventListener('click', trySubmit);
      break;
  }
}
```

This code might not look too different. It’s even a bit more verbose. But it is *dramatically* simpler to debug because of this line:

```jsx{3}
function setState(nextState) {
  // Clear all existing children
  formStatus.innerHTML = '';

  // ... the code adding stuff to formStatus ...
```

By clearing out the form status before doing any manipulations, we ensure that our DOM operations always start from scratch. This is how we can fight the inevitable [entropy](/the-elements-of-ui-engineering/) — by *not* letting the mistakes accumulate. This is the coding equivalent of “turning it off and on again”, and it works amazingly well.

**If there is a bug in the output, we only need to think *one* step back — to the previous `setState` call.** The Bug-O of debugging a rendering result is 🐞(*n*) where *n* is the number of rendering code paths. Here, it’s just four (because we have four cases in a `switch`).

We might still have race conditions in *setting* the state, but debugging those is easier because each intermediate state can be logged and inspected. We can also disallow any undesired transitions explicitly:

```jsx
function trySubmit() {
  if (currentState.step === 'pending') {
    // Don't allow to submit twice
    return;
  }
```

Of course, always resetting the DOM comes with a tradeoff. Naïvely removing and recreating the DOM every time would destroy its internal state, lose focus, and cause terrible performance problems in larger applications.

That’s why libraries like React can be helpful. They let you *think* in the paradigm of always recreating the UI from scratch without necessarily doing it:

```jsx
function FormStatus() {
  let [state, setState] = useState({
    step: 'initial'
  });

  function handleSubmit(e) {
    e.preventDefault();
    if (state.step === 'pending') {
      // Don't allow to submit twice
      return;
    }
    setState({ step: 'pending' });
    submitForm().then(() => {
      setState({ step: 'success' });
    }).catch(error => {
      setState({ step: 'error', error });
    });
  }

  let content;
  switch (state.step) {
    case 'pending':
      content = <Spinner />;
      break;
    case 'success':
      content = <SuccessMessage />;
      break;
    case 'error':
      content = (
        <>
          <ErrorMessage error={state.error} />
          <RetryButton onClick={handleSubmit} />
        </>
      );
      break;
  }

  return (
    <form onSubmit={handleSubmit}>
      {content}
    </form>
  );
}
```

The code may look different, but the principle is the same. The component abstraction enforces boundaries so that you know no *other* code on the page could mess with its DOM or state. Componentization helps reduce the Bug-O.

In fact, if *any* value looks wrong in the DOM of a React app, you can trace where it comes from by looking at the code of components above it in the React tree one by one. No matter the app size, tracing a rendered value is 🐞(*tree height*).

**Next time you see an API discussion, consider: what is the 🐞(*n*) of common debugging tasks in it?** What about existing APIs and principles you’re deeply familiar with? Redux, CSS, inheritance — they all have their own Bug-O.

---