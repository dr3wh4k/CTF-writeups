# TryHackMe – Fool's Mate

**Dificultad:** Fácil
**Categoría:** Web Exploitation / Lógica del lado del cliente vs. servidor
**Tags:** `burp-suite` `api-abuse` `client-side-validation` `web`

## Resumen

Fool's Mate presenta una pequeña aplicación web de ajedrez ("Endgame Trainer") en la que el jugador debe resolver un puzle de mate en una jugada. Cuando se intenta jugar la jugada ganadora desde la interfaz, la propia aplicación la bloquea con un mensaje de advertencia. El objetivo de la sala es entender por qué ocurre esto y saltarse esa restricción para obtener la flag.

La vulnerabilidad de fondo es un clásico: **toda la validación "peligrosa" se hace en el cliente (JavaScript), y el servidor confía ciegamente en lo que el navegador le envía.**

---

## 1. Reconocimiento

Empezamos con un escaneo de puertos estándar contra la IP de la máquina:

```bash
nmap -sC -sV -oN nmap_fools_mate.txt <IP_OBJETIVO>
```

Resultado (dos puertos abiertos):

| Puerto | Servicio |
|--------|----------|
| 22/tcp | SSH      |
| 80/tcp | HTTP     |

No hay mucho más que enumerar a nivel de red, así que el foco pasa directamente a la aplicación web.

Un `gobuster` rápido y una revisión de `robots.txt` no revelan rutas ocultas adicionales de interés; la superficie de ataque real está en la propia app.

```bash
gobuster dir -u http://<IP_OBJETIVO> -w /usr/share/wordlists/dirb/common.txt
```

---

## 2. Enumeración de la aplicación web

Al navegar a `http://<IP_OBJETIVO>` se carga **Endgame Trainer**, una aplicación de ajedrez interactiva. Se presenta un tablero con la posición cargada y le toca mover a las blancas, con un mate en una jugada disponible: mover la torre de **a1 a a8** (Ra8#).

Si se intenta jugar esa jugada directamente desde el tablero, aparece un pop-up de estilo "error de Windows" con un mensaje del tipo:

> *"I'll shut down your PC if you play that."*

Es decir, la propia interfaz **impide de forma deliberada** que ganes la partida por esa vía.

Inspeccionando el código fuente / el comportamiento del cliente (por ejemplo, interceptando `fetch` desde la consola del navegador), se observa que:

- El tablero usa la librería `chess.js` para validar las jugadas **en el cliente**.
- Existe una función de comprobación previa (tipo `preMoveCheck`) que detecta si la jugada entrante lleva a jaque mate y, si es así, **ni siquiera llega a enviar la petición** al servidor.
- Cada jugada legítima se envía como una petición `POST` a `/api/move` con un cuerpo JSON simple, sin token ni autenticación:

```json
{ "from": "a1", "to": "a3" }
```

El servidor responde con el nuevo estado del tablero (FEN), el estado de la partida y, en su caso, la jugada de respuesta del "bot".

**Esto es la pista clave:** si la validación que bloquea el mate vive solo en el JavaScript del cliente, el servidor probablemente nunca comprueba si la jugada final es "la prohibida".

---

## 3. Explotación

El plan es evitar por completo la lógica de bloqueo del cliente y hablar directamente con el backend.

### Opción A – Burp Suite (interceptar y modificar la petición)

1. Configurar el navegador para pasar por el proxy de Burp Suite.
2. En el tablero, jugar una jugada "inocente" con la torre, por ejemplo **a1 → a3**, para generar tráfico legítimo hacia `/api/move`.
3. Interceptar esa petición `POST` en Burp.
4. Modificar el cuerpo JSON de la petición interceptada, cambiando el destino de la jugada de `a3` a `a8`:

   ```diff
   - {"from": "a1", "to": "a3"}
   + {"from": "a1", "to": "a8"}
   ```

5. Reenviar (`Forward`) la petición modificada.

Como el servidor **no vuelve a validar del lado del backend** si esa jugada concreta es la "prohibida" (solo comprueba que sea una jugada de ajedrez legal), acepta el movimiento, calcula que es jaque mate y devuelve la flag en la respuesta JSON, junto con el estado `"checkmate"`.

### Opción B – Llamar a la API directamente desde la consola del navegador

Alternativamente, sin necesidad de proxy, se puede lanzar la petición directamente desde la consola de DevTools:

```javascript
fetch('/api/move', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({from: 'a1', to: 'a8'})
})
.then(r => r.json())
.then(console.log);
```

La respuesta del servidor confirma el jaque mate y entrega la flag:

```json
{
  "ok": true,
  "move": "a1a8",
  "status": "checkmate",
  "winner": "white",
  "flag": "THM{...}"
}
```

Ambos caminos llegan al mismo sitio: el "candado" que impide ganar solo existe en el navegador, nunca en el servidor.

---

## 4. Causa raíz

- **Confianza en el cliente:** la aplicación delega una regla de negocio crítica (impedir que el usuario gane) exclusivamente a JavaScript en el navegador.
- **Falta de validación en el servidor:** el endpoint `/api/move` acepta cualquier jugada de ajedrez sintácticamente válida sin comprobar reglas adicionales del "juego" (como si esa jugada concreta debe estar bloqueada).
- **Ausencia de autenticación/integridad en la API:** no hay tokens, firmas ni ningún mecanismo que impida modificar o repetir peticiones a `/api/move`.

Esto es una instancia clásica de **OWASP: Broken Access Control / Trust Boundary Violation** — confiar en controles que el atacante controla por completo (el navegador).

---

## 5. Lecciones aprendidas

- La validación del lado del cliente es una capa de **experiencia de usuario**, no de **seguridad**. Cualquier lógica que importe (permisos, reglas de negocio, límites) debe reforzarse también, y sobre todo, en el servidor.
- Antes de dar por buena una restricción visible en la interfaz, merece la pena comprobar si el backend la aplica de verdad: interceptar el tráfico con Burp Suite (o simplemente mirar las llamadas `fetch`/`XHR` desde DevTools) es un primer paso rápido y muy rentable.
- Los endpoints de API "sin pinta de interesantes" (como un simple `/api/move` de un juego) pueden esconder lógica de negocio explotable si no se replican las comprobaciones del cliente en el servidor.

---

## 6. Herramientas utilizadas

- `nmap`
- `gobuster`
- Navegador + DevTools (consola / pestaña Network)
- Burp Suite (Proxy / Repeater)

---

*Writeup con fines educativos, elaborado tras completar la sala en TryHackMe. No se incluye la flag real para respetar las normas de la plataforma.*
