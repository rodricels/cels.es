---
ID: 62
post_title: Cacheando GraphQL con Varnish
author: cels
post_date: 2017-03-01 13:27:11
post_excerpt: >
  El auge del lenguaje de consultas
  GraphQL está comiendo terreno a pasos
  agigantados a las APIs basadas en REST,
  pero todo cambio de paradigma trae
  consigo la ruptura de las viejas reglas
  a las que estamos acostumbrados.
layout: post
permalink: >
  https://cels.es/2017/03/cacheando-graphql-con-varnish/
published: true
---
<blockquote>Warning: artículo en proceso</blockquote>
El auge del lenguaje de consultas GraphQL está comiendo terreno a pasos agigantados a las APIs basadas en REST, pero todo cambio de paradigma trae consigo la ruptura de las viejas reglas a las que estamos acostumbrados.

Con REST teníamos asumidas varias capas de caché al estar basado en HTTP con el que es fácil controlar su flujo, conocer la vida que tendrá una petición y cachearla en los distintos intermediarios de la red, el navegador o el cliente HTTP. Toda esta magia la perdemos utilizando GraphQL.

<img class="aligncenter wp-image-75 size-full" src="https://cels.es/wp-content/uploads/2017/03/cachegraphql.jpg" width="568" height="335" />
<h2>Niveles de caché en GraphQL</h2>
GraphQL no especifica una capa de transporte en concreto, de hecho puedes usarlo con webservices, pero lo más común es utilizar HTTP como un "túnel tonto" sin disponer de la información que nos daban las cabeceras de expiración, ETaga, Max-age, etc. Esto tiene implicaciones a varios niveles, quedando así:
<h6>Aplicación</h6>
Es común que el backend implemente su propia capa de cacheo sobre Memcache o Redis, pero los servidores de GraphQL no crean cabeceras con los tiempos de vida a nivel de campo, grafo o petición de forma nativa, por lo que sólo queda construir una solución ad-hoc que utilice caberas o campos personalizados con la información de los TTLs y llegar a un acuerdo con las subsiguientes capas para que puedan tenerlas en cuenta.
<h6>Cliente</h6>
Los clientes HTTP no pueden verse beneficiados del control de expiración estándar por lo que deberían almacenar y controlar los TTLs de los datos para poder reutilizarlos y evitar peticiones innecesarias. Pero la forma óptima de almacenamiento es a nivel de grafo, de esa forma una petición puede añadir campos o datos de un grafo sin invalidarlo completamente. Esto hace que los clientes deban tener una lógica más compleja, que salvo el <a href="https://github.com/facebook/relay/issues/1369" target="_blank">futuro Relay v2</a>, pocos están abordando ya que tampoco hay un estándar en el lenguaje que defina qué reglas deben cumplir entre cliente y servidor.
<h6>Red</h6>
La capa de cacheo de red guarda las peticiones de forma completa y "gracias" a GraphQL quedan desfasada tal cual se usan ahora. Para poder recuperar esta funcionalidad podemos utilizar Varnish como cacheo a nivel de petición dentro de nuestra infraestructura, pero para poder utilizar CDNs tradicionales tendríamos que replicar las reglas que menciono en sus diversos lenguajes, aunque con el CDN <a href="https://www.fastly.com" target="_blank">Fastly</a> el trabajo sería mucho más sencillo ya que está basado en Varnish.
<h2>GraphQL sobre HTTP</h2>
GraphQL no especifica una capa de transporte en concreto, de hecho puedes usarlo con webservices pero la más común es HTTP que además te permite utilizar tanto GET como POST para las peticiones. Esto hace que debamos diferenciar dos peticiones al mismo endpoint añadiendo al hasing el método HTTP:
<pre class="prettyprint">sub vcl_hash {
  hash_data(req.method);
}</pre>
Además, si se utiliza POST en el endpoint, debemos añadir el cuerpo de la petición al hashing para poder cachearlas y diferenciarlas. Varnish no lo permite por defecto, pero para ello tenemos el módulo <a href="https://github.com/aondio/libvmod-bodyaccess">Bodyaccess</a> que nos da acceso como texto al número de bytes del body que le indiquemos:
<pre>import bodyaccess;

sub vcl_recv {
  # grab some data from request body
  std.cache_req_body(1KB);
}
sub vcl_hash {
  hash_data(req.method);
  bodyaccess.hash_req_body();
}</pre>
<h6>Endpoint con POST</h6>
Si se especifica la cabecera <code>application/json</code> todo el cuerpo de la petición debe estar codificado en JSON. Si se usa la cabecera <code>application/graphql</code> se permite mandar el cuerpo de la petición en el formato nativo de GraphQL. En ambos casos se puede pasar el parámetro <code>query</code> en la URL codificado como JSON, omitiéndolo así del cuerpo.
<table>
<tbody>
<tr style="height: 340.781px;">
<td style="height: 340.781px;">
<pre># application/json
{
  "query":
    "droidByName ($name: name) {
      droid (name: $name) {
        name,
        friends {
          name
        }
      }
    }"
  "operationName": "{...}",
  "variables": {
    "name": "R2-D2"
  }
}
</pre>
</td>
<td style="height: 340.781px;">
<pre># application/graphql
query
  droidByName ($name: name) {
    droid (name: $name) {
      name,
      friends {
        name
      }
    }
  }
operationName: {...}
variables: {
  "name": "R2-D2"
}
</pre>
</td>
</tr>
</tbody>
</table>
<h6>Endpoint con GET</h6>
Usando exclusivamente peticiones GET es la forma más sencilla de actuar.

La composición de la petición usa la forma nativa de GraphQL (no codificada como json) dividiendo los parámetros de la URL:
<pre>https://example.com/graphql?<strong>query</strong>={droidByName($name:name){droid(name:$name){name,friends{name}}}}
&amp;<strong>operationName</strong>={...}&amp;<strong>variables</strong>={"name":"R2-D2"}</pre>
<h6>Mutation y subscription</h6>
Los <code>mutation</code> son peticiones de escritura, actualización o borrado definidas por el servidor de GraphQL, equivalentes a PUT, POST y DELETE en una API REST.

Los <code>subscription</code> permiten al servidor mandar actualizaciones de datos mediante push a los clientes suscritos.

Si se está utilizando GET en la petición tenemos que poder diferenciarlas del resto de peticiones cacheables:
<pre>sub vcl_rev {
  if (req.url ~ "(\?|&amp;)(mutation|subscription)=") {
    return (pass);
  }
}</pre>
Con POST tenemos que inspeccionar el cuerpo de la petición para reconocer que es un mutation o subscription, cosa que Varnish no lo permite por defecto, pero para ello tenemos el módulo <a href="https://github.com/aondio/libvmod-bodyaccess">Bodyaccess</a> que nos da acceso como texto al número de bytes del body que le indiquemos.
<pre>import bodyaccess;

sub vcl_recv {
  # grab some bytes to analyze
  std.cache_req_body(1KB);

  # simple regex, harden it to your needs
  if (bodyaccess.req_body() ~ "(\"|)(mutation|subscription)(\"|)") {
    return (pass);
  }
}
</pre>
<h6>Tratamiento de errores</h6>
GraphQL es agnóstico a la capa de transporte y no se usan los <a href="https://en.wikipedia.org/wiki/List_of_HTTP_status_codes" target="_blank">HTTP status code</a> para saber si ha sido correcta la petición a nivel de datos.

Según la especificación de <a href="http://facebook.github.io/graphql/#sec-Errors" target="_blank">GraphQL sobre errores</a> las respuestas erróneas <strong>deben</strong> contener una lista de <code>errors</code> y <strong>pueden</strong> contener también la devolución de los datos en <code>data</code> si el resultado es sólo parcialmente erróneo.
<pre>[
    'data' =&gt; [
        'fieldWithException' =&gt; null
    ],
    'errors' =&gt; [
        [
            'message' =&gt; 'Exception message thrown in field resolver',
            'locations' =&gt; [
                ['line' =&gt; 1, 'column' =&gt; 2]
            ],
            'path': [
                'fieldWithException'
            ]
        ]
    ]
]
</pre>
Esta es una de las partes más fastidiosas ya que todas las peticiones responden con <code>status code: 200</code>, incluso con errores, y hay que inspeccionar prácticamente todo el cuerpo de la petición para saber si algo ha ido mal.

En nuestro caso lo más seguro es que no nos importe que estos errores se cacheen, pero en caso de no ser así, <strong>con Varnish no podemos inspeccionar el cuerpo de la respuesta del servidor</strong>, así que sólo nos queda cachear todo lo que nos llegue y pasar la responsabilidad al cliente de la API, que sería el encargado de reintentar la petición para complementar o corregir la petición del grafo erróneo.
<blockquote>Aunque hoy en día no se pueda acceder al cuerpo de la respuesta del backend, en el Vmod <a href="https://github.com/aondio/libvmod-bodyaccess" target="_blank">livvmod-bodyaccess</a> se está barajando añadir esta posibilidad.</blockquote>
Una solución a este problema es que el servidor o un middleware inspeccione las respuestas en busca de estos errores y los indique en una cabecera HTTP personalizada que capturemos:
<pre>sub vcl_backend_response {
  if ( beresp.http.X-GraphQL ~ "Not Cacheable" ) {
    set beresp.uncacheable = true;
  }
}</pre>
Gracias a la especificación de referencia, hay una función definida para tratar los errores: <a href="http://graphql.org/graphql-js/error/#formaterror"><code>formatError</code></a> y por ejemplo se puede sobrescribir, por ejemplo con <a href="http://dev.apollodata.com/tools/graphql-server/setup.html#graphqlOptions" target="_blank">GraphServer de Apollo</a> o la implementación de <a href="http://graphql.org/graphql-js/express-graphql/#graphqlhttp" target="_blank">express-graphql</a>, y añadir una cabecera HTTP con <code><a href="https://nodejs.org/api/http.html#http_response_setheader_name_value" target="_blank">response.setHeader()</a></code> de Nodejs.
<h6>Invalidaciones</h6>
TODO
<ul>
 	<li>ban y purge se ven afectadas por tener el mismo req.url</li>
</ul>
&nbsp;

&nbsp;