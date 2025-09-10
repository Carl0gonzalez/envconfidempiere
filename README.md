# iDempiere 12 — Entorno **aislado** con `direnv` (bash/zsh)

![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-E95420?logo=ubuntu&logoColor=white)
![Java 17](https://img.shields.io/badge/Java-17-007396?logo=java&logoColor=white)
![Maven 3.9.11](https://img.shields.io/badge/Maven-3.9.11-C71A36?logo=apache-maven&logoColor=white)
![direnv](https://img.shields.io/badge/direnv-enabled-2F855A)

Este documento explica cómo **cargar variables de entorno solo al entrar en** `/home/user/idempiere12`, usando [`direnv`](https://direnv.net/). Así compilas y ejecutas **iDempiere 12** con Maven y Eclipse locales, sin “ensuciar” tu entorno global.

> ### Estructura asumida
> ```text
> /home/user/idempiere12
> ├─ sources/
> │  └─ idempiere/               # fuentes de iDempiere (rama release-12)
> ├─ apache-maven-3.9.11/        # Maven binario local
> ├─ eclipse/                    # Eclipse descomprimido listo para usar
> └─ (opcional) jdk-17/          # si prefieres JDK local; si no, usa el del sistema
> ```

---

## 🚀 Inicio rápido (TL;DR)
1. Instala y engancha `direnv` al shell (zsh o bash).
2. Crea `/.envrc` dentro de `/home/user/idempiere12` con JAVA 17, Maven y Eclipse del proyecto.
3. `direnv allow` para autorizar el archivo.
4. Verifica con `java -version` / `mvn -version` **dentro** de la carpeta.
5. Compila iDempiere: `mvn -U -V clean verify` desde `sources/idempiere`.

---

## 1) Requisitos
- Ubuntu 22.04
- **Java 17** instalado (sistema o local)
- `git`
- `direnv` (lo instalaremos abajo)

---

## 2) Instalar y enganchar `direnv`

### zsh
```bash
sudo apt update && sudo apt install -y direnv
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
exec zsh
```

### bash
```bash
sudo apt update && sudo apt install -y direnv
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
exec bash
```

> `exec zsh` / `exec bash` recarga tu shell. Si prefieres, cierra y abre la terminal.

---

## 3) Crear el `.envrc` en `/home/user/idempiere12`

Crea el archivo con el siguiente contenido (ajusta si usas otro Maven/Eclipse/JDK).

```bash
# /home/user/idempiere12/.envrc
# ==== Entorno local iDempiere 12 (direnv) ====
BASE="$HOME/idempiere12"

# Rutas base
export IDEMPIERE_HOME="$BASE/sources/idempiere"
export MAVEN_HOME="$BASE/apache-maven-3.9.11"
export M2_HOME="$MAVEN_HOME"
export ECLIPSE_HOME="$BASE/eclipse"
export ECLIPSE_WORKSPACE="$BASE/workspace"

# --- Java 17 ---
# Opción A: usar el JDK 17 del sistema (autodetección de rutas comunes)
JAVA_CANDIDATES=(
  "/usr/lib/jvm/java-17-openjdk-amd64"
  "/usr/lib/jvm/java-1.17.0-openjdk-amd64"
  "/usr/lib/jvm/openjdk-17"
)
for j in "${JAVA_CANDIDATES[@]}"; do
  if [[ -x "$j/bin/java" ]]; then
    export JAVA_HOME="$j"
    break
  fi
done

# Opción B (alternativa): usar un JDK local dentro del proyecto
# export JAVA_HOME="$BASE/jdk-17"

# Validación mínima
if [[ -z "$JAVA_HOME" || ! -x "$JAVA_HOME/bin/java" ]]; then
  echo "ERROR: No se encontró Java 17. Ajusta JAVA_HOME en .envrc."
  exit 1
fi

# PATH locales primero (función PATH_add es de direnv stdlib)
PATH_add "$JAVA_HOME/bin"
PATH_add "$MAVEN_HOME/bin"
PATH_add "$ECLIPSE_HOME"

# Nota: direnv NO permite exportar PS1 por seguridad. 
# Si quieres un indicador visual, usa el apartado 'Prompt opcional' más abajo.
```

Autoriza el archivo (solo la primera vez o cuando lo modifiques):
```bash
cd /home/user/idempiere12
direnv allow
```

Cada vez que **entres** a `/home/user/idempiere12` (o subcarpetas) se cargarán estas variables; al **salir**, se descargarán automáticamente.

---

## 4) Verificar el entorno
```bash
java -version       # Debe mostrar 17.x
mvn -version        # Debe usar Maven 3.9.11 desde /home/user/idempiere12/apache-maven-3.9.11
echo "$IDEMPIERE_HOME"
```

---

## 5) Compilar iDempiere 12
```bash
cd /home/user/idempiere12/sources/idempiere
git checkout release-12
git pull --rebase

# build recomendado (fuerza actualización de P2/Tycho si cambió algo)
mvn -U -V clean verify
# o usando el wrapper del repo (si existe):
# ./mvnw -U -V clean verify
```

### Si aparece un error P2/Tycho (repositorios Eclipse/Jetty)
Limpia cachés y reintenta:
```bash
rm -rf ~/.m2/repository/.cache/tycho ~/.m2/repository/p2
# (opcional) si persiste, limpiar Jetty:
rm -rf ~/.m2/repository/org/eclipse/jetty
mvn -U -V clean verify
```

> Si ves `UnsupportedClassVersionError` con “class file 61 vs 55”, significa que Maven está corriendo con Java 11. Revisa que `JAVA_HOME` apunte a Java 17 y que `java -version` y `mvn -version` reflejen 17.

---

## 6) Arrancar Eclipse con workspace dedicado
```bash
"$ECLIPSE_HOME"/eclipse -data "$ECLIPSE_WORKSPACE" &
```

> En Eclipse, si usas *Target Platforms*, selecciona la de iDempiere (`org.idempiere.p2.targetplatform` o su mirror) para evitar errores de dependencias en el IDE.

---

## 7) Prompt opcional (indicador visual del entorno)

`direnv` **no** permite modificar `PS1` desde `.envrc`. Si quieres ver algo como `(idempiere12)` cuando las variables están activas, añade uno de estos snippets a tu shell:

### zsh (`~/.zshrc`)
```bash
setopt PROMPT_SUBST
show_idempiere12() {
  if [[ "$IDEMPIERE_HOME" == "$HOME/idempiere12/sources/idempiere" ]]; then
    echo "(idempiere12) "
  fi
}
PROMPT='$(show_idempiere12)'"$PROMPT"
```

### bash (robusto, evita duplicados) — `~/.bashrc`
```bash
# Prefijo de prompt cuando el entorno de idempiere12 está activo
__orig_ps1="${__orig_ps1:-$PS1}"
__update_ps1() {
  if [[ "$IDEMPIERE_HOME" == "$HOME/idempiere12/sources/idempiere" ]]; then
    PS1="(idempiere12) $__orig_ps1"
  else
    PS1="$__orig_ps1"
  fi
}
PROMPT_COMMAND="__update_ps1;${PROMPT_COMMAND}"
```

> Aplica tras `exec zsh` o `exec bash` (o reinicia terminal).

---

## 8) Buenas prácticas
- Añade `.envrc` a tu `.gitignore` si `/home/user/idempiere12` es repo:
  ```bash
  echo ".envrc" >> /home/user/idempiere12/.gitignore
  ```
- Mantén Java/Maven/Eclipse usados por iDempiere **solo** en el PATH de este directorio (gracias a `direnv`).
- Para cambiar de versión de Maven o Java local, ajusta rutas en `.envrc` y ejecuta `direnv reload`.

---

## 9) Troubleshooting rápido

| Problema | Causa probable | Solución |
|---|---|---|
| `mvn` usa Java 11 | `JAVA_HOME`/`PATH` no apuntan a 17 | Valida `java -version`/`mvn -version` y corrige `JAVA_HOME` en `.envrc`; `direnv reload` |
| Error P2/Tycho/Jetty | Caché P2/tycho inconsistente | Borra `~/.m2/repository/.cache/tycho` y `~/.m2/repository/p2`; ejecuta `mvn -U` |
| No se carga entorno | `direnv` no está enganchado o falta `allow` | Verifica `eval "$(direnv hook …)"` en tu shell y ejecuta `direnv allow` en `/home/user/idempiere12` |

---

## 10) Desactivar / desinstalar
- Desactivar temporalmente en el dir: `direnv deny`
- Quitar gancho del shell:
  - zsh: elimina `eval "$(direnv hook zsh)"` de `~/.zshrc`
  - bash: elimina `eval "$(direnv hook bash)"` de `~/.bashrc`

---

🧩 **Listo**: al trabajar dentro de `/home/user/idempiere12` tendrás **Java 17**, **Maven 3.9.11** y **Eclipse** locales, y tus builds de **iDempiere 12** quedarán totalmente **aislados** del entorno global.
