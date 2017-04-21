# shedding light on shadow brokers
## because in the shadow the smoke can't seen

### Analizando fuzzbunch

DoublePulsar y EternalBlue son las herramientas que todos ejecutaron ya, es por eso que para variar vamos a mirar como esta codeado el framework en python y ver si le podemos sacar algun partido

tambien vamos a cargar el eternalblue pero debuggeandolo para ver que es lo que hace y no fiarnos solo en los prints

el vector que vamos a usar para analizar el python, es mirar el codigo y cuando no nos cierra algo, le agregamos un:

```python
import pdb;pdb.set_trace()
```

para breakear la ejecucion y stepear desde ahi

pongamos uno de prueba para comenzar el traceo, luego al ejecutarlo breakea automaticamente

![](docs/img/1-FB_DIR.PNG)

en la anterior imagen setea todas las variables de los directorios que va a usar posteriormente

FB_DIR apunta al directorio base de fuzzbunch y desde ahi accede a los plugins, payloads, etc...

en la siguiente imagen muestro los directorios de plugins que no estan en el leak

![](docs/img/2-plugin-directories.PNG)

tal vez haya que esperar a un nuevo leak por parte de shadow brokers

despues llegamos al __main__, donde llama a la intuitiva setup_and_run:

![](docs/img/3-main.png)

este metodo crea una instancia del objeto Fuzzbunch, veamos el constructor del objeto:

![](docs/img/4-constructorLeeConfiguracion.png)

configfile es FB_CONFIG con el valor:

```python
os.path.join(FB_DIR, "Fuzzbunch.xml")
```

readconfig parsea ese archivo xml y carga la informacion en el hash configvars, el cual almacena las opciones globales.

Ni bien retorna ese metodo setea la variable enablecolor evaluando el dato del xml

![](docs/img/6-eval.png)

### Bug en archivo de configuracion

encontramos un bug de seguridad en fuzzbunch, claramente el impacto es minimo porque el xml ya viene con el framework.

El archivo original tiene un texto con un True o False booleano de python:

![](docs/img/7-1-original.png)

como este texto es evaluado con eval(), podriamos lograr una ejecucion de comandos con:

![](docs/img/7-test.png)

dejando la variable enablecolor con el stdout del dir:

![](docs/img/9-commandinjection.png)

la ejecucion de comandos corrio satisfactoriamente, y ese comando va a correr con los privilegios del usuario que ejecuto fuzzbunch

### Siguiendo en el constructor de Fuzzbunch

Fuzzbunch es heredado de FbCmd y como es logico en el constructor crea el padre:

![](docs/img/11-ejecutando-abuelo.png)

FbCmd hereda de cmd.Cmd, ese objeto abuelo es parte de la libreria estandar de python y se usa para configurar una interfaz interactiva al estilo metasploit o gdb

### Comandos interactivos

antes de cmd.Cmd para hacer una consola interactiva uno codeaba:

```python
while True
    command = raw_input("prompt> ")
    if command == "fruta"
        print "Ejecutando fruta"
    else if command == "note"
        print "note running"
```

ahora con cmd.Cmd podemos lograr el mismo efecto con un codigo mas limpio:

```python
import cmd

class HelloWorld(cmd.Cmd):
    
    def do_fruta(self, line):
        print "Ejecutando fruta"
    
    def do_note(self, line):
        print "note running"

if __name__ == '__main__':
    HelloWorld().cmdloop()
```
con eso en mente solo tenemos que ver el metodo do_something donde something es el comando

los comandos disponibles son:

![](docs/img/23-commands.png)

el comando "use" nos deja cargar los plugins, entonces le vamos a dedicar un capitulo

### command "use"

vamos a cargar eternalblue para analizar como cargan los plugins:

![](docs/img/26-eternalblue.png)

"use" parcea el archivo xml del plugin para cargar los parametros que estan entre el tag <inputparameters>

para solicitar un option del plugin se configura asi:

![](docs/img/24-options.png)

y si queremos dar opciones para elegir usamos paramchoice:

![](docs/img/25-paramchoices.png)

estas opciones son para configurar TargetPort, Shellcode y diferentes opciones que necesita el exploit

### Run y Attach al ejecutable

una vez finalizada la carga de options del plugin pasa a ejecutar el exe con Popen y seteando un par de argumentos

![](docs/img/27-comando.png)

para analizar el ejecutable vamos a ponerle un while true en el entry point y partimos desde ahi

![](docs/img/28-loop.png)

esto nos va a permitir hacer un Attach al process y debuggearlo antes de que ejecute su codigo


