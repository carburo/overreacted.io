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

El problema de este código no es que sea «feo». No se trata de una cuestión estética. **El problema es que si hay un error en este código, no sé dónde empezar a buscar.**

**En dependencia del orden en que los *callbacks** y los eventos se disparen, hay una explosión combinatoria del número de caminos en el código que este programa podría tomar.** En algunos, se podrán ver los mensajes correctos. En otros, lo que se mostrará será la mezcla de una multitud de indicadores de carga, mensajes de error y de fallas; y probablemente también ???crashes.

Esta función tiene 4 secciones diferentes sin garantías sobre su orden. Mi primer cálculo no científico me dice que hay 4×3×2×1 = 24 órdenes diferntes en los que podrían ocurrir. Si añado 4 segmentos más, será 8×7×6×5×4×3×2×1, igual a *cuarenta mil* combinaciones. Te deseo buena suerte si te atreves a depurarlo.

**En otras palabras, la notación Bug-O de este enfoque es 🐞(<i>n!</i>)**, donde *n* es el número de segmentos de código que tocan el DOM. Sí, ese es un *factorial*. Por supuesto, no estoy siendo muy científico. No todas las transiciones son posibles en la práctica. Pero por otra parte, cada uno de esos segmentos pueden ejecutarse más de una vez. <span style="word-break: keep-all">🐞(*¯\\\_(ツ)\_/¯*)</span> podría ser más preciso, pero aún es bastante malo. Podemos hacerlo mejor.

---

Para mejorar el Bug-O de este código, podemos limitar el número de estados posibles y de resultados. No necesitamos ninguna biblioteca para lograrlo. Es solo cuestión de promover cierta estructura en nuestro código. Esta es una forma de hacerlo:

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

Este código podría no parecer demasiado diferente. Incluso es un poco más verboso. Pero es *dramáticamente* más simple de depurar simplemente por esta línea:

```jsx{3}
function setState(nextState) {
  // Clear all existing children
  formStatus.innerHTML = '';

  // ... the code adding stuff to formStatus ...
```

Al limpiar el estado del formulario antes de hacer cualquier manipulación, nos aseguramos que nuestras operaciones en el DOM siempre se inician desde cero. Es así como podemos enfrentar a la inevitable [entropía](/the-elements-of-ui-engineering/), al *no* permitir que se acumulen los errores. Este es el equivalente en código de «apaga y vuelve a encender», y funciona increíblemente bien.

**Si hay solo un error en la salida, solo necesitamos pensar *un* paso hacia atrás (la llamada previa a `setState`.)** La Bug-O de la depuración de un resultado de renderizado es 🐞(*n*) donde *n* es el número de caminos de código de renderizado. Aquí hay solo cuatro (porque tenemos cuatro casos en un `switch`).

Aún así podríamos tener condiciones de carrera al asignar el estado, pero depurarlos es más fácil porque cada estado intermedio puede ser impreso e inspeccionado. También podemos rechazar explícitamente cualquier transición no deseada:

```jsx
function trySubmit() {
  if (currentState.step === 'pending') {
    // Don't allow to submit twice
    return;
  }
```

Por supuesto, el hecho de reiniciar siempre siempre el DOM tiene una desventaja. Si se elimina y recrea el DOM en cada ocasión se destruiría su estado interno, perdería el foco y causaría terribles problemas en el rendimiento en aplicaciones grandes.

Es por eso que bibliotecas como React pueden ser de gran ayuda. Ellas te permiten *pensar* bajo el paradigma de recrear la interfaz de usuario desde cero sin que en realidad ocurra:

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

El código puede parecer distinto, pero el principio es el mismo. La abstracción de un componente impone barreras de manera que sepas que *otro* código en la página no puede afectar su DOM o su estado. El diseño basado en componentes ayuda a reducir la Bug-O.

De hecho, si *cualquier* valor parece incorrecto en el DOM de una aplicación React, puedes rastrearlo al mirar el código de los componentes por encima de él en el árbol de React, uno por uno. Sin importar el tamaño de la aplicación, rastrear un valor renderizado es 🐞(*tamaño del árbol*).

**La próxima vez que veas una discusión sobre una API, considera: ¿cuál es el 🐞(*n*) de las tareas comunes de depuración en ella?**. ¿Y qué ocurre con las API y los principios con los que estás ampliamente familiarizado? Redux, CSS, herencia, todos tienen su propio Bug-O.

---