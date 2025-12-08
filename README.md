# 🚀 iDempiere 12 — Entorno Aislado (direnv + Maven + Eclipse 2022-06)

> Guía completa para crear un **entorno de desarrollo aislado** para iDempiere 12.  
> Funciona en Ubuntu/Kubuntu 22.04/24.04 y afines.  
> Rutas **genéricas** (reemplaza `\<YOUR_PATH\>` por tu carpeta de trabajo).

---

## 🗂️ Estructura recomendada de carpetas

```text
<YOUR_PATH>/idempiere12/
├─ .envrc                 # Variables y helpers del entorno (direnv)
├─ .m2/                   # Repositorio Maven del entorno
│  └─ settings.xml
├─ .mvn/
│  └─ maven.config        # Forzar settings/repo local (sin comentarios)
├─ .p2/                   # Área p2/Eclipse del entorno (no ensucia ~/.p2)
├─ apache-maven-3.9.11/   # Maven local (opcional si no usas el del sistema)
├─ eclipse/               # Eclipse 2022-06 (SDK)
├─ workspace/             # Workspace aislado
└─ sources/
   └─ idempiere/          # Código fuente de iDempiere 12
```

---

## ✅ Requisitos

- **Java 17 JDK**  
- **direnv** + **zsh** (o bash)
- **Eclipse 2022-06** (paquete *Eclipse IDE for RCP and RAP Developers*)
- **Maven 3.9.11** (local o del sistema)
- **Git**

### Instalación rápida (Debian/Ubuntu)

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk git curl unzip zsh direnv
# (opcional) Maven del sistema
sudo apt install -y maven
```

---

## 📦 Descargas (Maven + Eclipse)

> Si prefieres usar Maven **local**:
```bash
cd <YOUR_PATH>/idempiere12
curl -fsSL https://downloads.apache.org/maven/maven-3/3.9.11/binaries/apache-maven-3.9.11-bin.zip -o maven.zip
unzip maven.zip && rm maven.zip
mv apache-maven-3.9.11 ./apache-maven-3.9.11
```

> Eclipse 2022-06 (Linux x86_64):
```bash
cd <YOUR_PATH>/idempiere12
curl -fsSL https://ftp.osuosl.org/pub/eclipse/technology/epp/downloads/release/2022-06/R/eclipse-rcp-2022-06-R-linux-gtk-x86_64.tar.gz -o eclipse.tar.gz
tar -xzf eclipse.tar.gz && rm eclipse.tar.gz
mv eclipse ./eclipse
```

> (Usa cualquier mirror oficial de Eclipse si este cambia.)

---

## ⚙️ Configuración del entorno (`.envrc`)

Crea **`<YOUR_PATH>/idempiere12/.envrc`** con este contenido (rutas genéricas):

```bash
# ===============================================
#  iDempiere 12 — Entorno aislado (direnv + zsh)
#  Ubicación raíz: <YOUR_PATH>/idempiere12
# ===============================================

# --- Raíz del entorno ---
BASE="<YOUR_PATH>/idempiere12"

# --- Rutas principales ---
export IDEMPIERE_HOME="$BASE/sources/idempiere"
export ECLIPSE_HOME="$BASE/eclipse"
export ECLIPSE_WORKSPACE="$BASE/workspace"

# --- Maven del entorno (ajusta si usas el del sistema) ---
export MAVEN_HOME="$BASE/apache-maven-3.9.11"
export M2_HOME="$MAVEN_HOME"
export MVN_LOCAL_REPO="$BASE/.m2/repository"
export MAVEN_USER_SETTINGS="$BASE/.m2/settings.xml"

# --- P2/Eclipse local (evita ensuciar ~/.p2) ---
export ECLIPSE_P2="$BASE/.p2"

# --- Java 17 (autodetección) ---
JAVA_CANDIDATES=(
  "/usr/lib/jvm/java-17-openjdk-amd64"
  "/usr/lib/jvm/java-1.17.0-openjdk-amd64"
  "/usr/lib/jvm/openjdk-17"
)
unset JAVA_HOME
for j in "${JAVA_CANDIDATES[@]}"; do
  if [[ -x "$j/bin/java" ]]; then
    export JAVA_HOME="$j"
    break
  fi
done
if [[ -z "$JAVA_HOME" ]]; then
  echo "❌ No se encontró Java 17. Instala: sudo apt install -y openjdk-17-jdk"
  exit 1
fi

# --- Asegurar estructura mínima ---
mkdir -p \
  "$BASE/.m2/repository" \
  "$BASE/.mvn" \
  "$ECLIPSE_WORKSPACE" \
  "$BASE/sources" \
  "$ECLIPSE_P2"

# --- settings.xml local ---
if [[ ! -f "$MAVEN_USER_SETTINGS" ]]; then
  cat >"$MAVEN_USER_SETTINGS" <<EOF
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository>$MVN_LOCAL_REPO</localRepository>
</settings>
EOF
fi

# --- maven.config SIN comentarios (Maven falla si hay '#') ---
_MVN_CFG="-s $MAVEN_USER_SETTINGS
-Dmaven.repo.local=$MVN_LOCAL_REPO"
printf "%s\n" "$_MVN_CFG" > "$BASE/.mvn/maven.config"
if [[ -d "$IDEMPIERE_HOME" ]]; then
  mkdir -p "$IDEMPIERE_HOME/.mvn"
  printf "%s\n" "$_MVN_CFG" > "$IDEMPIERE_HOME/.mvn/maven.config"
fi

# --- Flags/opts comunes ---
export MAVEN_OPTS="${MAVEN_OPTS:-} -Dmaven.repo.local=$MVN_LOCAL_REPO -Drevision=12.0.0"
export JAVA_TOOL_OPTIONS="-Djava.awt.headless=true"

# --- PATH (prioriza Java/Maven del entorno) ---
PATH_add "$JAVA_HOME/bin"
PATH_add "$MAVEN_HOME/bin"
PATH_add "$ECLIPSE_HOME"

# --- Aliases útiles ---
alias mvn='"$MAVEN_HOME/bin/mvn" -s "$MAVEN_USER_SETTINGS" -Dmaven.repo.local="$MVN_LOCAL_REPO"'
alias eclipse-choose='"$ECLIPSE_HOME/eclipse" -vm "$JAVA_HOME/bin/java" -data @noDefault -configuration "$ECLIPSE_P2/configuration" -Dosgi.instance.area.default="$ECLIPSE_WORKSPACE"'
alias eclipse-here='"$ECLIPSE_HOME/eclipse" -vm "$JAVA_HOME/bin/java" -data "$ECLIPSE_WORKSPACE" -configuration "$ECLIPSE_P2/configuration"'

# ==========================
#  Panel visual de estado
# ==========================
if command -v tput >/dev/null 2>&1; then
  _bold=$(tput bold); _dim=$(tput dim); _reset=$(tput sgr0)
  _c1=$(tput setaf 6); _c2=$(tput setaf 2); _c3=$(tput setaf 5); _c4=$(tput setaf 4); _cY=$(tput setaf 3)
else
  _bold=''; _dim=''; _reset=''; _c1=''; _c2=''; _c3=''; _c4=''; _cY=''
fi
_print_kv () { printf "  ${_bold}%-16s${_reset} %s\n" "$1" "$2"; }
hr_top="┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓"
hr_mid="┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫"
hr_bot="┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛"

echo ""
echo "${_c1}${hr_top}${_reset}"
printf "┃ ${_bold}iDempiere 12 — Entorno cargado${_reset} %-21s ┃\n" ""
echo "${_c1}${hr_mid}${_reset}"
_print_kv "JAVA_HOME"      "${_c2}$JAVA_HOME${_reset}"
_print_kv "MAVEN_HOME"     "${_c2}$MAVEN_HOME${_reset}"
_print_kv "Repo Maven"     "${_c3}$MVN_LOCAL_REPO${_reset}"
_print_kv "Settings.xml"   "${_c3}$MAVEN_USER_SETTINGS${_reset}"
_print_kv "Eclipse"        "${_c4}$ECLIPSE_HOME${_reset}"
_print_kv "Workspace"      "${_c4}$ECLIPSE_WORKSPACE${_reset}"
_print_kv "Eclipse p2"     "${_c4}$ECLIPSE_P2${_reset}"
_print_kv "IDEMPIERE_HOME" "${_cY}$IDEMPIERE_HOME${_reset}"
echo "${_c1}${hr_bot}${_reset}"
echo "💡 ${_dim}Comandos rápidos:${_reset}  ${_bold}eclipse-choose${_reset}  |  ${_bold}eclipse-here${_reset}  |  ${_bold}mvn -v${_reset}"
echo ""
```

> **Activa el entorno**:
```bash
cd <YOUR_PATH>/idempiere12
direnv allow
# Cada vez que entres a la carpeta, se cargarán las vars automáticamente.
```

---

## 🧬 Clonar iDempiere 12 (sources)

```bash
mkdir -p <YOUR_PATH>/idempiere12/sources
cd <YOUR_PATH>/idempiere12/sources
git clone https://github.com/idempiere/idempiere.git
cd idempiere
git switch release-12.0
```

*(Opcional: si ya tenías el repo copiado desde otro sistema, asegúrate de que la rama sea `release-12.0` y que exista `.mvn/maven.config` como arriba.)*

---

## 🧪 Verificaciones rápidas

```bash
# Mostrar panel (se carga con direnv al entrar)
cd <YOUR_PATH>/idempiere12

# Java & Maven efectivos
which java && java -version
which mvn && mvn -v

# Maven usa el repo/settings del entorno
mvn -q help:effective-settings -DforceStdout | grep -A1 localRepository
# Debe mostrar: <localRepository><YOUR_PATH>/idempiere12/.m2/repository</localRepository>
```

---

## 🧰 Primer build (línea de comandos)

```bash
cd <YOUR_PATH>/idempiere12/sources/idempiere
mvn -U -DskipTests verify
```

> Si hay errores por *settings* o *localRepository*, revisa que **`.mvn/maven.config`** NO tenga `#` ni comentarios y apunta a:
```
-s <YOUR_PATH>/idempiere12/.m2/settings.xml
-Dmaven.repo.local=<YOUR_PATH>/idempiere12/.m2/repository
```

---

## 🖥️ Eclipse 2022-06 con variables del entorno

### Lanzar Eclipse

- Con diálogo de workspace:
```bash
eclipse-choose
```

- Directo al workspace del entorno:
```bash
eclipse-here
```

### Importar iDempiere
1. **File → Import → Existing Maven Projects**  
2. Selecciona `<YOUR_PATH>/idempiere12/sources/idempiere`.  
3. Acepta y espera la indexación.

### Target Platform
- Ve a **`iDempiere`** proyecto raíz → tareas de *Target Platform* según la wiki del proyecto (o usa el *target definition* incluido si aplica).
- Mantén el **p2 area** en `ECLIPSE_P2` (ya forzado por los flags al lanzar Eclipse).

---

## 🧯 Troubleshooting (rápido)

- **“The specified user settings file does not exist”**  
  Revisa en `~/.envrc`/`.envrc` que no haya espacios raros en rutas y que exista:
  - `<YOUR_PATH>/idempiere12/.m2/settings.xml`
  - `<YOUR_PATH>/idempiere12/.mvn/maven.config` (sin comentarios)

- **Maven no usa el repo local del entorno**  
  Confirma:
  ```bash
  mvn -q help:effective-settings -DforceStdout | grep -A1 localRepository
  ```
  Si apunta a `~/.m2/repository`, forzaste el Maven del sistema. Usa el alias `mvn` creado por `.envrc` o invoca:
  ```bash
  "$MAVEN_HOME/bin/mvn" -s "$MAVEN_USER_SETTINGS" -Dmaven.repo.local="$MVN_LOCAL_REPO" -v
  ```

- **Eclipse/p2 ensucia `~/.p2`**  
  Asegúrate de lanzar Eclipse con los flags del alias `eclipse-here`/`eclipse-choose` para usar `ECLIPSE_P2`.

- **Errores de manifiestos/plug-ins en Eclipse**  
  - `Project → Clean…` (limpiar todo).  
  - `Window → Preferences → Plug-in Development → Target Platform` y re-aplicar el target.  
  - Verifica que `JAVA_HOME` apunta a Java 17 y que Eclipse está usando ese VM (`-vm` en el alias).

---

## 🧭 Comandos de referencia

```bash
# Entrar al entorno
cd <YOUR_PATH>/idempiere12
direnv allow

# Lanzar Eclipse (elige workspace)
eclipse-choose

# Lanzar Eclipse (workspace del entorno)
eclipse-here

# Ver variables activas (panel)
cd <YOUR_PATH>/idempiere12  # se imprime al entrar

# Build iDempiere
cd <YOUR_PATH>/idempiere12/sources/idempiere
mvn -U -DskipTests verify
```

---

## 🧩 Notas finales

- Esta guía **no** expone rutas personales: usa `<YOUR_PATH>` en todos los ejemplos.  
- Si mueves la carpeta del entorno a otra ruta, **solo** actualiza `BASE` en `.envrc`.  
- Evita comentarios `#` dentro de `.mvn/maven.config` (Maven los interpreta como entradas inválidas).  

---
