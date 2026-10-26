# Contigo · Documentación del proyecto

**App de seguimiento para pacientes bariátricos y de GLP-1**
Última actualización: septiembre 2026

---

## 1. Qué es

Web app para que Aimee (médica) monitoree a sus pacientes entre consultas, y para que las pacientes registren su día a día de forma rápida y visual.

Dos tipos de paciente:

- **GLP-1** — semaglutida, tirzepatida (dosis semanal, efectos secundarios, adherencia)
- **Post-cirugía bariátrica** — timeline de recuperación, proteína, hidratación, dumping

**Referencia de diseño:** la app Flo. Botones circulares grandes, interacción por emoji, muchas gráficas, cero fricción para registrar.

**Formato:** web app (PWA). Se abre desde el navegador, se puede "instalar" en la pantalla de inicio del celular y se ve como app nativa. No está en App Store — se comparte por link.

---

## 2. Stack

| Capa | Tecnología |
|---|---|
| Frontend | HTML + CSS + JavaScript (un solo archivo) |
| Gráficas | Chart.js |
| Backend / Base de datos | Firebase (Firestore) |
| Autenticación | Firebase Auth (Email/Password) |
| Hosting | cPanel en designar.com.mx |
| Edición local | Claude Code |

> Migrado de Supabase a Firebase en septiembre 2026. La cuenta y datos de Supabase quedaron abandonados; no hay migración de datos entre plataformas.

---

## 3. Accesos y rutas

### Firebase
- **Consola:** https://console.firebase.google.com
- **Config del SDK web** (apiKey, authDomain, projectId, etc.) está pegado directamente en `login.html` y `contigo_final.html`.

> La `apiKey` de Firebase es pública por diseño (va dentro del HTML). La seguridad real la dan las reglas de Firestore (`firestore.rules`), no la llave.

### En línea
- **App:** `designar.com.mx/contigo.html`
- **Login:** `designar.com.mx/logincontigo.html`
- **Prototipo original:** `designar.com.mx/appaimeeprot1.html`

### Local
```
/Users/jelanithompson/Downloads/APP AIMEE/contigo_final.html
```

---

## 4. Base de datos

### Colecciones (Firestore)

| Colección | Para qué |
|---|---|
| `perfiles` | Un doc por usuario, id = uid de Firebase Auth. Guarda nombre, rol (`paciente` / `doctora`), email |
| `pacientes` | Ficha clínica: tipo de tratamiento, peso inicial, meta, fecha de cirugía o inicio de GLP-1, `user_id`, `doctora_id` |
| `registros` | Registro diario, id = `{paciente_id}_{fecha}` (permite upsert por día): síntomas, proteína, agua, peso, dosis aplicada |
| `consultas` | Citas y notas de la doctora |

Reglas de seguridad en `firestore.rules` (pegar en Firebase Console → Firestore → Rules).

### Semáforo
Firestore no tiene vistas del lado del servidor como Postgres. La lógica que antes vivía en `resumen_pacientes` ahora se calcula en el cliente, dentro de `initApp()` en `contigo_final.html`: por cada paciente de la doctora se leen sus últimos 14 `registros` y se calculan `dias_sin_registrar`, `nausea_prom4` y `dias_prot_baja` en JS antes de pintar el semáforo.

### Usuarios
Las cuentas de Supabase (Aimee y Regina) no se migraron — hay que crearlas de nuevo desde `login.html` (pestaña "Crear cuenta") una vez configurado el proyecto de Firebase. A diferencia de Supabase, Firebase Auth no exige confirmar el correo por SQL: el login funciona inmediatamente después de crear la cuenta.

---

## 5. Estado actual

### ✅ Funciona
- `login.html` y `contigo_final.html` migrados a Firebase Auth + Firestore (código listo, falta pegar el `firebaseConfig` real de la consola)
- Todo el diseño y las pantallas del prototipo (wizard, registro diario, gráficas, panel de doctora)
- `finishWizard()` → `sbSavePaciente()` guarda en `pacientes` (Firestore)
- `saveReg()` guarda en `registros` (Firestore, upsert por día)
- `saveConsulta()` guarda en `consultas` (Firestore)
- Panel de la doctora calcula el semáforo en cliente a partir de `registros` reales (ver sección 4)
- La app está subida y accesible en el hosting (aún sirviendo el HTML con Supabase — falta resubir tras la migración)

### ❌ Pendiente
| # | Qué | Detalle |
|---|---|---|
| 1 | Crear proyecto de Firebase | Auth (Email/Password) + Firestore + pegar `firebaseConfig` en ambos HTML |
| 2 | Recrear usuarios | Aimee y Regina ya no existen en Firebase, hay que darse de alta de nuevo desde `login.html` |
| 3 | Ruteo por rol en `initApp()` | Una paciente puede llegar a la vista de doctora |
| 4 | Carcasa de teléfono | El wrapper CSS de demo aparece en escritorio; hay que quitarlo |
| 5 | Botones decorativos | Configuración (tuerca), consultas, perfil — eran visuales en el prototipo, nunca tuvieron función |
| 6 | Resubir a hosting | El HTML en `designar.com.mx` sigue siendo la versión con Supabase |

---

## 6. Errores conocidos y sus causas

Estos son los tropiezos que ya se identificaron. Vale la pena tenerlos presentes para no repetirlos:

**`SyntaxError: Unexpected string`**
Un template literal que contiene la cadena `</script>` rompe el parser de HTML — el navegador cierra el bloque de script ahí mismo. Solución: partir la cadena (`'</scr' + 'ipt>'`) o evitar meter HTML completo dentro de template literals.

**`initApp is not defined` / `sbSavePaciente is not defined`**
Orden de carga. Las funciones se llaman antes de que su script exista. Todo el código de Firebase debe cargar **antes** de cualquier llamada, o envolver el arranque en `DOMContentLoaded`.

**Integraciones en conflicto**
Se intentaron varios enfoques de inyección de script en iteraciones distintas y quedaron mezclados en el mismo archivo. Al retomar, conviene partir de una versión limpia en vez de parchear encima.

**Firebase config sin pegar**
Si `firebaseConfig` sigue con los valores `"PASTE_AQUI"`, todas las llamadas a Auth/Firestore fallan silenciosamente o con error de inicialización. Pegar el objeto real desde Firebase Console → Project Settings.

---

## 7. Siguientes pasos sugeridos

En orden, del más desbloqueante al menos:

1. **Crear el proyecto de Firebase** — Auth (Email/Password) + Firestore, pegar `firebaseConfig` en `login.html` y `contigo_final.html`, subir `firestore.rules` en la consola
2. **Recrear las cuentas** de Aimee y Regina desde `login.html`
3. **Verificar el wizard y el registro diario** en Firestore (Console → Firestore Database → Data) — el código ya escribe en `pacientes` y `registros`, falta probar de punta a punta
4. **Ruteo por rol** — que `initApp()` lea `perfiles.rol` y mande a la vista correcta, sin permitir la otra
5. **Pulido** — quitar carcasa de teléfono, activar botones decorativos, PWA (manifest + icono para pantalla de inicio)
6. **Resubir a hosting** en designar.com.mx

El paso 1 es el único bloqueante real — el resto del código de guardado ya está conectado.

---

## 8. Roadmap más allá del piloto

- Piloto cerrado con Regina y Aimee (estado actual)
- Alta de pacientes reales de Aimee (5–10)
- Alertas automáticas / notificaciones
- Si más adelante se quiere App Store: React Native o FlutterFlow, con el diseño y la lógica ya resueltos aquí

---

## 9. Contexto para retomar con Claude Code

Al abrir una sesión nueva, este archivo sirve de contexto completo. El resumen de una línea:

> Web app HTML+JS migrada de Supabase a Firebase (Auth + Firestore) en septiembre 2026. Diseño terminado, guardado del wizard y del registro diario ya conectados a Firestore en el código. Falta crear el proyecto real de Firebase y pegar su config, recrear las cuentas de Aimee y Regina, y forzar el ruteo por rol.
