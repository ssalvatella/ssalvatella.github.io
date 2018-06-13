---
layout: post
title:  "Un poco de web scraping en plusdede.com :mag_right:"
date:   2018-06-11 14:02:41 +0200
categories: python
---
Hoy en día es bastante conocida la plataforma [plusdede.com](https://www.plusdede.com/) en la que se pueden ver cientos de series y películas. Aunque reconozco ser bastante novato
en la técnica de [*web scraping*](https://es.wikipedia.org/wiki/Web_scraping), en este post comentaré un ejemplo que he realizado en `Python` sobre esta web.

Es un ejemplo bastante sencillo pero permite tener una idea básica sobre esta técnica.

Lo que queremos conseguir al terminar este post es:

:point_right: &nbsp;&nbsp; Identificarnos dentro de la web con un usuario y contraseña.

:point_right: &nbsp;&nbsp; Mostrar los capítulos que tenemos pendientes de ver por consola.

---

## Identificarnos en la web

Para empezar, necesitaremos tener **ChromeDriver** en nuestro ordenador. Nos permite hacer uso del motor del navegador Chrome desde Python.
Podemos descargarlo desde [aquí](http://chromedriver.chromium.org/downloads).

Una vez descargado, dejamos el ejecutable en el directorio donde vayamos a trabajar.

Para empezar a programar necesitamos tener instalado el paquete `selenium` en Python. Lo instalamos rápidamente con el comando `pip install selenium`.

Probamos que el paquete esta instalado correctamente con el siguiente código:

{% highlight python %}
from selenium import webdriver

# Arrancamos el navegador
navegador = webdriver.Chrome("chromedriver") 
# Cerramos el navegador
navegador.quit()
{% endhighlight %}

> El parámetro `"chromedriver"` es la ruta al ChromeDriver descargado. En caso de tenerlo en una carpeta distinta a la del script indicar su ruta.

Si ejecutamos este código, veremos cómo se abre el navegador Chrome para después cerrarse. ¡Estamos controlando el navegador desde nuestro script de Python! Es un buen comienzo...

De todas formas, no necesitamos estar constantemente abriendo el navegador para ejecutar nuestro script. Desactivamos la apertura del Chrome de la siguiente manera:

{% highlight python %}
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

opciones = Options()
# Desactivamos la apertura del navegador
opciones.set_headless(headless=True)
# Indicamos que no queremos estar viendo todos los mensajes del navegador
opciones.add_argument("log-level=3") 

# Arrancamos el navegador
navegador = webdriver.Chrome("chromedriver", chrome_options=opciones) 
# Cerramos el navegador
navegador.quit()
{% endhighlight %}

Ahora, cuando ejecutemos el script, ya no nos aparecerá el navegador.

Lo siguiente será hacer nuestra primera función que se encargará de identificarse dentro de la web.

Si vamos a la página de identificación de [`plusdede.com/login`](https://www.plusdede.com/login) veremos una página como la siguiente:

![Página de login de plusdede.com]({{"https://raw.githubusercontent.com/ssalvatella/ssalvatella.github.io/master/assets/img/plusdede_login.PNG " | absolute_url }} "plusdede.com/login")

Lo primero que llama nuestra atención es la presencia del *captcha* que hay que completar para poder entrar. Tendrá que haber una forma de mostrar al usuario el *captcha*
para que este lo complete y poder identificarse correctamente.

En este momento la herramienta de `inspeccionar elemento` de nuestro Firefox o Chrome se vuelve nuestra mejor amiga.
Si hacemos click derecho en la imagen del *captcha* y le damos a inspeccionar elemento podremos ver en el panel derecho que se abre de qué tipo de elemento es.

![Inspeccionando elemento]({{"https://raw.githubusercontent.com/ssalvatella/ssalvatella.github.io/master/assets/img/plusdede_inspeccionar_elemento.PNG" | absolute_url }} "Inspeccionando captcha")

Vemos por lo tanto que se trata de una imagen con su etiqueta `<img>`.

Empezamos nuestra función `login()` de la siguiente forma:

{% highlight python %}
def login(usuario, password):
    # Le decimos al navegador que pida la página para identificarse
    navegador.get("https://www.plusdede.com/login")
    # Obtenemos el elemento web <img> de la página
    captcha = navegador.find_element_by_css_selector("img")
{% endhighlight %}

Para poder mostrarle al usuario del script la imagen del *captcha* deberemos de hacer una función auxiliar que tome una instantánea de ese recuadro de la página y la abra con cualquier visor de imágenes.

Necesitaremos para crear la imagen el paquete `pillow`. 

Lo instalamos rápidamente con el comando `pip install pillow`.

Añadimos en la cabecera del script la importación del nuevo paquete mediante `from PIL import Image`

De esta forma, nuestra función auxiliar quedaría de la siguiente manera:

{% highlight python %}
def guardar_captcha(elemento, ruta):

    # Obtenemos la posición y la medida del elemento
    posicion = elemento.location
    medida = elemento.size

    # Guardamos una imagen de la página en la ruta indicada
    navegador.save_screenshot(ruta)

    # Utilizamos la librería para abrir la imagen en memoria
    image = Image.open(ruta)

    # Obtenemos las coordenadas a recortar de la imagen
    izquierda = posicion['x']
    arriba = posicion['y']
    derecha = posicion['x'] + medida['width']
    abajo = posicion['y'] + medida['height']

    # Recortamos la imagen para dejar solo el captcha
    imagen = image.crop((izquierda, arriba, derecha, abajo))
    # Guardamos la imagen después de recortarla
    imagen.save(ruta, 'png')
    return image
{% endhighlight %}

Con esto, ya tenemos una función que nos guarda la imagen del *captcha* en un archivo `.png`.

Ahora volviendo a nuestra función de `login()`:

{% highlight python %}
def login(usuario, password):
    # Le decimos al navegador que pida la página para identificarse
    navegador.get("https://www.plusdede.com/login")
    # Obtenemos el elemento web <img> de la página
    captcha = navegador.find_element_by_css_selector("img")
    # Guarda el captcha en un archivo
    guardar_captcha(captcha, "captcha.png")
    # Mostramos la imagen en el visor por defecto
    Image.open("captcha.png").show()
    # Le pedidos al usuario que escriba el captcha que le aparece
    texto_captcha = input("Introduce los números de la imagen: ")
{% endhighlight %}

Una vez que podemos hacer que el usuario complete el *captcha*, habrá que rellenar los campos con los datos que tenemos.

Para ello, utilizando de nuevo la herramienta `inspeccionar elemento` del navegador, vemos los identificadores (`id`) de los campos.

En la función `login()` de nuevo, le diremos al navegador que escriba los datos en los campos con esos identificadores.

Después, se hará click en el botón para entrar. Habrá que decirle al navegador que espere hasta que aparezca un elemento que nos diga
que el login se ha realizado correctamente.

Para realizar esa espera habrá que añadir a nuestras importaciones al inicio del script:

{% highlight python %}
from selenium.common.exceptions import TimeoutException
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC
{% endhighlight %}

Y luego en la función `login()`:

{% highlight python %}
def login(usuario, password):
    print("Entrando en Plusdede.com para realizar login...")
    navegador.get(URL + "/login")
    captcha = navegador.find_element_by_css_selector("img")
    guardar_captcha(captcha, "captcha.png") # guarda el captcha en un fichero
    # Mostramos la imagen en el visor por defecto
    Image.open("captcha.png").show()

    # Le pedidos al usuario que escriba el captcha que le aparece
    texto_captcha = input("Introduce los números de la imagen: ")
    # Introducimos en el campo el captcha
    campo_captcha = navegador.find_element_by_id("input-captcha")
    campo_captcha.clear()
    campo_captcha.send_keys(texto_captcha)

    # Se escribe el usuario y la contraseña en el resto de campos de la misma forma
    campo_usuario = navegador.find_element_by_id("input-email")
    campo_usuario.clear()
    campo_usuario.send_keys(usuario)
    campo_password = navegador.find_element_by_name("password")
    campo_password.clear()
    campo_password.send_keys(password)

    # Obtenemos el botón para enviar el formulario
    boton_entrar = navegador.find_element_by_css_selector("button")
    boton_entrar.click() # Hacemos click en él

    # Esperamos un máximo de 3 segundos a que aparezca el elemento 'username' en la página
    try: 
        WebDriverWait(navegador, 3).until(
            EC.presence_of_element_located((By.CLASS_NAME, 'username'))
        )
    except TimeoutException:
        return False
    return True
{% endhighlight %}

En caso de que la identificación se haya realizado correctamente, la función devolverá `True` y en caso contrario `False`.

Podemos probar que todo funciona correctamente con nuestros datos de usuario:

{% highlight python %}
if login("mi_usuario", "mi_password"):
    print("Logeado correctamente!")
else:
    print("Error en el login.")

navegador.quit() # Cerramos el navegador
{% endhighlight %}

## Leer nuestros capítulos pendientes

Vamos a profundizar un pelín más en las posibilidades que nos ofrece el *web scraping*.

Lo siguiente es que nuestro script en `Python` lea que capítulos tenemos pendientes de ver y nos los muestre por consola.

Una tarea así requiere una herramienta versátil para **parsear** la página `html`.

La herramienta perfecta para esto es `BeautifulSoup`. 

Como siempre la instalamos fácilmente mediante `pip install beautifulsoup4`

Y la añadimos a las importaciones del inicio del script `from bs4 import BeautifulSoup`.

Una vez más, utilizaremos `inspeccionar elemento` para entender como organiza la página los capítulos en el código `html`.

![Inspeccionando elemento]( {{ "https://raw.githubusercontent.com/ssalvatella/ssalvatella.github.io/master/assets/img/plusdede-inicio-inspeccionar.PNG" | absolute_url }} "Inspeccionando capítulos pendientes")

Definiremos un nuevo método que se encargue de leer los capítulos de la página de inicio en función de los elementos que hemos visto que 
se encargan de mostrar los capítulos.

{% highlight python %}
def mostrar_capitulos_pendientes():
    print("Tienes pendientes los siguientes capítulos: ")
    # Utilizamos BeautifulSoup para parsear el código de la página del navegador
    pagina = BeautifulSoup(navegador.page_source, "html.parser")
    # Obtenemos todos los elementos que contengan capítulos
    capitulos = pagina.findAll("div", {"class": "media-container"})
    # Recorremos cada capitulo
    for capitulo in capitulos:
        # Obtenemos el nombre del capítulo
        nombre = capitulo.find("div", {"class": "media-title"}).getText()
        # Obtenemos el enlace a la serie
        enlace = capitulo.find("a", {"data-container": "body"})['href']
        # Imprimimos por pantalla cada capítulo pendiente de ver
        print("-> " + nombre + " " + enlace)
{% endhighlight %}

> En el método `find()` o `findAll()`, el primer parámetro indica el tipo de elemento (`<div>`, `<a>`, etc..).
> El segundo parámetro añade más campos a buscar como la `class` o cualquier atributo `html` que deseemos buscar.
> De esta manera, obtenemos los elementos específicos que hemos visto analizando la página.

Volvemos a probar el funcionamiento:

{% highlight python %}
if login("mi_usuario", "mi_password"):
    print("Logeado correctamente!")
    mostrar_capitulos_pendientes()
else:
    print("Error en el login.")

navegador.quit() # Cerramos el navegador
{% endhighlight %}

Y esto nos devuelve:

![Resultado final]({{ "https://raw.githubusercontent.com/ssalvatella/ssalvatella.github.io/master/assets/img/scraping_resultado.PNG" | absolute_url }} "Resultado final")

## Conclusiones

Hemos completado un ejemplo muy sencillo de *web scraping*.

Lo importante es **entender** en que consiste esta **técnica** y el **potencial** que tiene al aportar soluciones para **automatizar procesos**
en los cuales por alguna razón **no tenemos acceso a una API abierta**.

Si quieres acceder al código completo del script puedes hacerlo [aquí](https://gist.github.com/ssalvatella/6acaf5d6f61a29c49d305030f80b67e5).

>¿Quizás podríamos intentar en el futuro descargar automáticamente los capítulos que tenemos pendientes?