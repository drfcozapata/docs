# Transformación de la Reactividad {#reactivity-transform}

:::danger Característica Experimental Obsoleta
La Transformación de la Reactividad era una función experimental y ha quedado obsoleta. Por favor, lea sobre [el razonamiento aquí](https://github.com/vuejs/rfcs/discussions/369#discussioncomment-5059028).

Eventualmente será elimidada del núcleo de Vue en una futura versión menor. Si todavía quieres usarlo, ahora está disponible a través del plugin [Vue Macros](https://vue-macros.sxzz.moe/features/reactivity-transform.html).
:::

:::tip Específica de la Composition API
La Transformación de la Reactividad es una característica específica de la Composition-API y requiere un paso de compilación.
:::

## Refs vs. Variables Reactivas {#refs-vs-reactive-variables}

Desde la introducción de la Composition API, una de las principales cuestiones sin resolver es el uso de objetos refs vs. objetos reactive. Es fácil perder la reactividad cuando se desestructuran los objetos reactive, mientras que puede ser engorroso utilizar `.value` en todas partes cuando se utilizan refs. Además, `.value` es fácil de omitir si no se utiliza un sistema de tipos.

La [Transformación de la Reactividad de Vue](https://github.com/vuejs/core/tree/main/packages/reactivity-transform) es una transformación en tiempo de compilación que nos permite escribir código como este:

```vue
<script setup>
let count = $ref(0)

console.log(count)

function increment() {
  count++
}
</script>

<template>
  <button @click="increment">{{ count }}</button>
</template>
```

El método `$ref()` aquí es una **macro de tiempo de compilación**: no es un método real que será llamado en tiempo de ejecución. En su lugar, el compilador de Vue lo utiliza como una pista para tratar la variable resultante `count` como una **variable reactiva.**

Las variables reactivas pueden ser accedidas y reasignadas como las variables normales, pero estas operaciones se compilan en refs con `.value`. Por ejemplo, la parte `<script>` del componente anterior se compila en:

```js{5,8}
import { ref } from 'vue'

let count = ref(0)

console.log(count.value)

function increment() {
  count.value++
}
```

Cada API de reactividad que devuelva refs tendrá una macro equivalente con el prefijo `$`. Estas APIs incluyen:

- [`ref`](/api/reactivity-core#ref) -> `$ref`
- [`computed`](/api/reactivity-core#computed) -> `$computed`
- [`shallowRef`](/api/reactivity-advanced#shallowref) -> `$shallowRef`
- [`customRef`](/api/reactivity-advanced#customref) -> `$customRef`
- [`toRef`](/api/reactivity-utilities#toref) -> `$toRef`

Estas macros están disponibles globalmente y no necesitan ser importadas cuando está habilitada la Transformación de la Reactividad, pero puedes importarlas opcionalmente desde `vue/macros` si quieres ser más explícito:

```js
import { $ref } from 'vue/macros'

let count = $ref(0)
```

## Desestructuración con `$()` {#destructuring-with}

Es habitual que una función de composición devuelva un objeto de refs, y que utilice la desestructuración para recuperar estas refs. Para este propósito, la transformación de la reactividad proporciona la macro **`$()`**:

```js
import { useMouse } from '@vueuse/core'

const { x, y } = $(useMouse())

console.log(x, y)
```

Salida compilada:

```js
import { toRef } from 'vue'
import { useMouse } from '@vueuse/core'

const __temp = useMouse(),
  x = toRef(__temp, 'x'),
  y = toRef(__temp, 'y')

console.log(x.value, y.value)
```

Observa que si `x` ya es una ref, `toRef(__temp, 'x')` simplemente la devolverá tal cual y no se creará ninguna ref adicional. Si un valor desestructurado no es una ref (por ejemplo, una función), seguirá funcionando; el valor será envuelto en una ref para que el resto del código funcione como se espera.

La desestructuración de `$()` funciona tanto en objetos reactivos **como** en objetos simples que contienen refs.

## Convertir las Referencias Existentes en Variables Reactivas con `$()` {#convert-existing-refs-to-reactive-variables-with}

En algunos casos podemos tener funciones envueltas que también devuelven refs. Sin embargo, el compilador de Vue no podrá saber de antemano que una función va a devolver una ref. En estos casos, también se puede utilizar la macro `$()` para convertir las refs existentes en variables reactivas:

```js
function myCreateRef() {
  return ref(0)
}

let count = $(myCreateRef())
```

## Desestructuración de Props Reactivas {#reactive-props-destructure}

Hay dos puntos problemáticos con el uso actual de `defineProps()` en `<script setup>`:

1. Al igual que con `.value`, necesitas acceder siempre a los props como `props.x` para mantener la reactividad. Esto significa que no puedes desestructurar `defineProps` porque las variables desestructuradas resultantes no son reactivas y no se actualizarán.

2. Cuando se utiliza la [declaración de props de sólo tipo](/api/sfc-script-setup#type-only-props-emit-declarations), no hay una manera fácil de declarar valores por defecto para las props. Hemos introducido la API `withDefaults()` para este propósito, pero sigue siendo difícil de usar.

Podemos solucionar estos problemas aplicando una transformación en tiempo de compilación cuando se utiliza `defineProps` con desestructuración, similar a lo que vimos antes con `$()`:

```html
<script setup lang="ts">
  interface Props {
    msg: string
    count?: number
    foo?: string
  }

  const {
    msg,
    // el valor por defecto simplemente funciona
    count = 1,
    // El sistema de alias local también funciona
    // aquí estamos poniendo un alias de `props.foo` a `bar`
    foo: bar
  } = defineProps<Props>()

  watchEffect(() => {
    // registrará cada vez que las props cambien
    console.log(msg, count, bar)
  })
</script>
```

Lo anterior será compilado en la siguiente declaración equivalente en tiempo de ejecución:

```js
export default {
  props: {
    msg: { type: String, required: true },
    count: { type: Number, default: 1 },
    foo: String
  },
  setup(props) {
    watchEffect(() => {
      console.log(props.msg, props.count, props.foo)
    })
  }
}
```

## Mantener la Reactividad a través de los Límites de la Función {#retaining-reactivity-across-function-boundaries}

Mientras que las variables reactivas nos liberan de tener que usar `.value` en todas partes, se crea un problema de "pérdida de reactividad" cuando pasamos variables reactivas a través de los límites de la función. Esto puede ocurrir en dos casos:

### Introducir una Función como Argumento {#passing-into-function-as-argument}

Dada una función que espera una ref como argumento, por ejemplo:

```ts
function trackChange(x: Ref<number>) {
  watch(x, (x) => {
    console.log('¡x cambió!')
  })
}

let count = $ref(0)
trackChange(count) // ¡no funciona!
```

El caso anterior no funcionará como se espera porque compila a:

```ts
let count = ref(0)
trackChange(count.value)
```

Aquí `count.value` se pasa como un número, mientras que `trackChange` espera una referencia real. Esto se puede arreglar envolviendo `count` con `$$()` antes de pasarlo:

```diff
let count = $ref(0)
- trackChange(count)
+ trackChange($$(count))
```

Lo anterior se compila a:

```js
import { ref } from 'vue'

let count = ref(0)
trackChange(count)
```

Como podemos ver, `$$()` es una macro que sirve como **punto de escape**: a las variables reactivas dentro de `$$()` no se les añadirá `.value`.

### Retorno en el Ámbito de la Función {#returning-inside-function-scope}

La reactividad también puede perderse si las variables reactivas se utilizan directamente en una expresión retornada:

```ts
function useMouse() {
  let x = $ref(0)
  let y = $ref(0)

  // escucha a mousemove...

  // ¡no funciona!
  return {
    x,
    y
  }
}
```

La declaración de retorno anterior compila a:

```ts
return {
  x: x.value,
  y: y.value
}
```

Para mantener la reactividad, deberíamos retornar las refs actuales, no el valor vigente en el momento del retorno.

De nuevo, podemos utilizar `$$()` para solucionar esto. En este caso, `$$()` se puede utilizar directamente en el objeto retornado; cualquier referencia a las variables reactivas dentro de la llamada a `$$()` conservará la referencia a sus refs subyacentes:

```ts
function useMouse() {
  let x = $ref(0)
  let y = $ref(0)

  // escucha a mousemove...

  // corregido
  return $$({
    x,
    y
  })
}
```

### Uso de `$$()` en Props Destructuradas {#using-on-destructured-props}

`$$()` funciona en props desestructuradas ya que son variables reactivas también. El compilador lo convertirá con `toRef` por eficiencia:

```ts
const { count } = defineProps<{ count: number }>()

passAsRef($$(count))
```

compila a:

```js
setup(props) {
  const __props_count = toRef(props, 'count')
  passAsRef(__props_count)
}
```

## Integración con TypeScript <sup class="vt-badge ts" /> {#typescript-integration}

Vue proporciona tipificaciones para estas macros (disponibles globalmente) y todos los tipos funcionarán como se espera. No hay incompatibilidades con la semántica estándar de TypeScript, por lo que la sintaxis funcionará con todas las herramientas existentes.

Esto también significa que las macros pueden funcionar en cualquier archivo donde se permitan JS / TS válidos y no sólo dentro de los SFC de Vue.

Dado que las macros están disponibles globalmente, sus tipos necesitan ser referenciados explícitamente (por ejemplo, en un archivo `env.d.ts`):

```ts
/// <reference types="vue/macros-global" />
```

Si se importan explícitamente las macros desde `vue/macros`, el tipo funcionará sin declarar las globales.

## Opt-in Explícito {#explicit-opt-in}

:::warning
Lo siguiente sólo se aplica hasta la versión Vue 3.3 e inferiores. El soporte se eliminará del núcleo en la versión 3.4 y superiores. Si tiene la intención de seguir utilizando la transformación, por favor migre a [Vue Macros](https://vue-macros.sxzz.moe/features/reactivity-transform.html) en su lugar.
:::

### Vite {#vite}

- Requiere `@vitejs/plugin-vue@>=2.0.0`
- Se aplica a los SFC y a los archivos js(x)/ts(x). Se realiza una comprobación de uso rápido en los archivos antes de aplicar la transformación, por lo que no debería haber ningún costo de rendimiento para los archivos que no utilizan las macros.
- Ten en cuenta que `reactivityTransform` es ahora una opción a nivel de raíz del plugin en lugar de estar anidada como `script.refSugar`, ya que no sólo afecta a los SFC.

```js
// vite.config.js
export default {
  plugins: [
    vue({
      reactivityTransform: true
    })
  ]
}
```

### `vue-cli` {#vue-cli}

- Actualmente sólo afecta a los SFC
- Requiere `vue-loader@>=17.0.0`

```js
// vue.config.js
module.exports = {
  chainWebpack: (config) => {
    config.module
      .rule('vue')
      .use('vue-loader')
      .tap((options) => {
        return {
          ...options,
          reactivityTransform: true
        }
      })
  }
}
```

### `webpack` Plano + `vue-loader` {#plain-webpack-vue-loader}

- Actualmente sólo afecta a los SFC
- Requiere `vue-loader@>=17.0.0`

```js
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          reactivityTransform: true
        }
      }
    ]
  }
}
```
