# Documento de Especificación de Requisitos de Software (SRS)

**Nombre del Proyecto:** OTTO 
Significado: Optimized Task & Typing Operator

(Operador Optimizado de Tareas y Escritura)

Por qué funciona: Fonéticamente suena a "Auto" (de automatización). Es un nombre real, muy fácil de recordar y escribir en consola o como comando ejecutable (otto.exe).

**Tipo de Entregable:** Aplicación de Escritorio Nativa (.EXE autónomo)

**Arquitectura:** Python compilado a binario independiente (PyInstaller)

**Versión:** 1.0.0

---

## 1. Introducción

### 1.1 Propósito del documento

Definir de manera exhaustiva los requisitos funcionales, no funcionales, reglas de negocio, limitaciones técnicas y arquitectura operativa para el desarrollo de un software de automatización y grabación de acciones de usuario (*mouse* y teclado). Este documento sirve como directriz técnica para el diseño, implementación mediante agentes de código (Codex / Claude) y validación de calidad del producto final.

### 1.2 Alcance del sistema

El sistema comprende una aplicación de escritorio nativa para Windows (10/11) capaz de interceptar, almacenar, editar y reproducir secuencias de eventos periféricos en entornos locales y sesiones de escritorio remoto (RDP, AnyDesk, RustDesk). El software opera de forma 100% local, sin conexión a servicios en la nube, bases de datos externas ni dependencias preinstaladas de Python en la máquina del usuario final.

### 1.3 Definiciones generales

* **Macro:** Secuencia estructurada de eventos temporizados (clic, movimiento, pulsación de tecla y retraso).
* **Hook de bajo nivel:** Mecanismo del sistema operativo (`WH_MOUSE_LL` y `WH_KEYBOARD_LL` en la API Win32) para interceptar eventos antes de que lleguen a la aplicación destino.
* **Fail-Safe / Kill-Switch:** Interrupción forzada del flujo en ejecución activada por hardware o teclado para prevenir ciclos infinitos o acciones erráticas.
* **Template Matching:** Técnica de visión por computadora para localizar una imagen de referencia (ej. un botón) dentro de un lienzo de pantalla más amplio.
* **Standalone .EXE:** Archivo ejecutable autocontenido que incluye el runtime, bibliotecas y assets sin requerir proceso de instalación.

---

## 2. Objetivo del sistema

Proveer una solución ultraligera, portátil y sin costo recurrente de licencias para la automatización de flujos repetitivos, eliminando la sobrecarga computacional, la dependencia de cuentas corporativas y las limitaciones de ejecución en segundo plano presentes en herramientas comerciales como Microsoft Power Automate Desktop.

---

## 3. Actores involucrados

* **Operador (Usuario Final):** Persona que graba secuencias, configura repeticiones, lanza la ejecución y supervisa la tarea automatizada.
* **Agente Desarrollador (Codex / Claude Code):** Entidad técnica encargada de traducir esta especificación a código fuente modular en Python y generar los scripts de compilación.
* **Sistema Operativo Windows:** Capa receptora de eventos sintéticos y emisora de hooks periféricos a través de la API Win32.

---

## 4. Descripción general del sistema

La aplicación opera bajo un modelo desacoplado compuesto por cuatro submódulos centrales:

```
[ Interfaz Gráfica (CustomTkinter) ]
                 │
                 ▼
[ Orquestador / Gestor de Macros (JSON Engine) ]
        │                               │
        ▼                               ▼
[ Módulo de Captura (Pynput) ]   [ Motor de Inyección (PyAutoGUI / Ctypes) ]
                                                │
                                                ▼
                                 [ Módulo Opcional de Visión (OpenCV) ]

```

1. **Capa de Presentación:** Ventana compacta, limpia y flotante que permite iniciar grabación, pausar, reproducir y configurar parámetros de velocidad.
2. **Motor de Captura:** Hilo en segundo plano que escucha coordenadas absolutas $(X, Y)$, botones de mouse, pulsaciones de teclas y marcas de tiempo relativas ($\Delta t$).
3. **Persistencia:** Almacenamiento en archivos locales estructurados en formato JSON.
4. **Motor de Inyección:** Hilo independiente de alta prioridad que sintetiza las acciones grabadas, validando el estado del Kill-Switch antes de cada instrucción.

---

## 5. Requisitos funcionales

| ID | Requisito | Descripción | Prioridad |
| --- | --- | --- | --- |
| **RF-01** | Grabación de Mouse | Captura de clic izquierdo, clic derecho, doble clic, arrastrar (*drag & drop*), scroll y coordenadas $(X, Y)$ absolutas. | **CRÍTICO** |
| **RF-02** | Grabación de Teclado | Registro de pulsaciones de teclas individuales, combinaciones especiales (*hotkeys* tipo `Ctrl+C`, `Alt+Tab`) y cadenas de texto. | **CRÍTICO** |
| **RF-03** | Preservación de Tiempos | Registro del intervalo exacto en milisegundos entre cada acción para una reproducción fiel al ritmo humano. | **CRÍTICO** |
| **RF-04** | Freno de Emergencia (Kill-Switch) | Aborto inmediato e incondicional de cualquier macro al presionar una tecla global de pánico (por defecto `ESC` o `F9`). | **CRÍTICO** |
| **RF-05** | Modos de Reproducción | Configuración para reproducir: 1 sola vez, $N$ veces, o en bucle infinito hasta activar el Kill-Switch. | **IMPORTANTE** |
| **RF-06** | Modificador de Velocidad | Selector de aceleración para ejecutar la secuencia a $1\text{x}$, $1.5\text{x}$, $2\text{x}$ o $5\text{x}$ respecto al tiempo original. | **IMPORTANTE** |
| **RF-07** | Exportación/Importación | Guardar la macro activa en un archivo `.macro` (JSON) y cargar archivos previos desde el explorador de Windows. | **IMPORTANTE** |
| **RF-08** | Anclaje Visual (Template Matching) | Opción de capturar un área pequeña de pantalla (botón) y buscar sus coordenadas visuales antes de hacer clic, evitando fallos si la ventana se movió. | **MEJORA** |

---

## 6. Requisitos no funcionales

* **RNF-01 (Rendimiento):** Uso de CPU en reposo $\le 0.3\%$. Durante grabación o reproducción, el consumo de CPU no debe exceder el $3\%$ en procesadores de cuatro núcleos o superiores.
* **RNF-02 (Uso de Memoria):** Consumo de memoria RAM estable menor a 85 MB durante la ejecución continua.
* **RNF-03 (Latencia de Eventos):** Desviación temporal del motor de reproducción inferior a 15 milisegundos respecto a la marca grabada.
* **RNF-04 (Portabilidad y Empaquetado):** Distribución en un único archivo binario compilado (`.exe`) con peso menor a 45 MB, operable sin privilegios de instalación.
* **RNF-05 (Seguridad y Privacidad):** El software no debe contener telemetría, no realiza peticiones HTTP/DNS externas y almacena las macros únicamente en el almacenamiento local del equipo.

---

## 7. Reglas de negocio

* **RN-01 (Prioridad del Kill-Switch):** El hilo del freno de emergencia se ejecuta en el nivel más alto del sistema operativo; ninguna macro o bucle puede demorar, interceptar o anular la señal de aborto.
* **RN-02 (Normalización de Pausas Mínimas):** Cualquier delay registrado inferior a 25 milisegundos debe forzarse a un mínimo de 25 ms durante la reproducción para prevenir que el buffer de eventos de Windows descarte comandos pegados.
* **RN-03 (Bloqueo de Coordenadas Negativas):** Si la secuencia detecta coordenadas fuera de la resolución del monitor activo (ej. monitores desconectados), la ejecución debe detenerse preventivamente emitiendo una alerta.
* **RN-04 (Protección contra Pérdida de Foco en RDP):** Al interactuar sobre una ventana de escritorio remoto, si la ventana pierde el foco en el sistema operativo local, la reproducción debe pausarse automáticamente.

---

## 8. Parámetros de validación del sistema

* **Validación de Integridad JSON:** Todo archivo `.macro` importado debe verificar su esquema (campos obligatorios: `version`, `timestamp`, `events` $\rightarrow$ `[type, x, y, key, delay]`).
* **Umbral de Confianza Visual:** Para la función de Template Matching con OpenCV, el coeficiente de correlación normalizado (`cv2.TM_CCOEFF_NORMED`) debe ser $\ge 0.85$ para considerarse un clic válido; de lo contrario, el sistema debe pausar y notificar "Elemento visual no encontrado".
* **Límites de Bucles:** La interfaz no permitirá valores negativos en el campo de iteraciones ($N \ge 1$).

---

## 9. Entradas del sistema

* **Señales de periféricos físicos:** Captura de eventos mediante `pynput.mouse` y `pynput.keyboard`.
* **Archivos locales:** Carga de scripts de automatización `.macro` (JSON).
* **Parámetros GUI:** Clics en la interfaz de control, valor numérico de iteraciones, selector de velocidad y atajo de teclado para inicio/fin.
* **Entrada gráfica (modo visión):** Archivo de imagen `.png` recortado para anclaje visual.

---

## 10. Salidas del sistema

* **Inyección de hardware sintético:** Emisión de señales de mouse y teclado a la cola de mensajes de Windows (`SendInput` de la API de Win32).
* **Archivos persistentes:** Generación del archivo `.macro` estructurado con la información de la sesión.
* **Panel de estado:** Actualización del estado en tiempo real dentro de la interfaz gráfica (`Listo`, `Grabando...`, `Reproduciendo (Paso X/Y)`, `Detenido`).
* **Registro de eventos (Log):** Archivo de texto local opcional (`debug.log`) generado únicamente si se activa el parámetro de depuración.

---

## 11. Restricciones del sistema

* **Plataforma Exclusiva:** Diseñado específicamente para sistemas operativos Microsoft Windows de 64 bits (x64).
* **Barrera de UAC (Control de Cuentas de Usuario):** Si la ventana destino está ejecutada con permisos de Administrador, la aplicación `.exe` también deberá ejecutarse como Administrador para poder inyectar eventos en ella (restricción propia de seguridad del kernel de Windows / UIPI).
* **Compresión de Video en RDP:** En conexiones de escritorio remoto con fluctuaciones severas de ancho de banda, la degradación de imagen puede invalidar temporalmente el módulo de reconocimiento visual por OpenCV.

---

## 12. Supuestos

* La máquina del usuario cuenta con una resolución de pantalla idéntica o adaptada entre el momento de la grabación y el momento de la reproducción cuando se use el modo de coordenadas puras.
* Las soluciones antivirus convencionales permitirán el funcionamiento de los hooks de teclado/mouse al tratarse de un binario firmado internamente o excluido localmente.

---

## 13. Criterios de aceptación

* **CA-01:** El sistema graba con éxito una secuencia de 30 acciones arbitrarias (combinando clics, tipeo y esperas) y la replica idénticamente en una prueba de formulario local.
* **CA-02:** Al presionar la tecla `ESC` o `F9` durante cualquier punto de una reproducción en bucle infinito, la ejecución se interrumpe de forma limpia en menos de 100 ms sin colgar el sistema.
* **CA-03:** El binario compilado `.exe` se ejecuta correctamente en una máquina virtual limpia con Windows 11 sin entornos de desarrollo ni librerías externas instaladas.
* **CA-04:** La aplicación puede exportar e importar el archivo `.macro` conservando intactos todos los parámetros de tiempo y coordenadas.

---

## 14. Futuras mejoras

* **OCR Ligero Local:** Integración de un motor OCR compacto para buscar palabras de texto dinámicas en lugar de plantillas de imagen fijas.
* **Lógica Condicional (If/Else):** Capacidad de bifurcar la acción si una imagen aparece en pantalla (ej. *si existe botón OK, pulsar; si aparece Cancelar, abortar*).
* **Soporte Multi-Monitor Dinámico:** Ajuste algorítmico automático de coordenadas cuando el usuario alterna entre monitores con factores de escala (DPI) distintos (ej. 100% y 125%).

---

## 15. Conclusión

Este documento formaliza los cimientos para construir un activo de automatización propio, seguro, portátil y altamente eficiente. Al desacoplar la ejecución de las herramientas tradicionales pesadas, el usuario obtiene soberanía sobre sus flujos de trabajo repetitivos con una solución que arranca al instante y se adapta con precisión milimétrica a las necesidades de su operación diaria.