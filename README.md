<div align="center">

# 🧩 iDempiere 12 — Entorno **Aislado** (Ubuntu 22.04, zsh/bash)

**Objetivo**: Ejecutar iDempiere 12, Maven 3.9.11, Java 17 y Eclipse  
**sin tocar el HOME global**, todo dentro de `~/idempiere12`.

</div>

---

## 📌 Índice

- [✨ Vista rápida](#-vista-rápida)
- [📁 Estructura de carpetas](#-estructura-de-carpetas)
- [🧰 Requisitos](#-requisitos)
- [🌳 .envrc (direnv)](#-envrc-direnv)
- [📦 Maven aislado: `.m2` y `.mvn`](#-maven-aislado-m2-y-mvn)
- [🧪 Pruebas rápidas](#-pruebas-rápidas)
- [🧠 Eclipse con entorno aislado](#-eclipse-con-entorno-aislado)
- [🧱 plugin-builder con entorno aislado](#-plugin-builder-con-entorno-aislado)
- [🧯 Troubleshooting](#-troubleshooting)
- [✅ Checklist final](#-checklist-final)
- [🧾 Apéndice: comandos útiles](#-apéndice-comandos-útiles)

---

## ✨ Vista rápida

| Componente | Versión / Ruta | Notas |
|---|---|---|
| **Java** | OpenJDK **17** (`/usr/lib/jvm/java-17-openjdk-amd64`) | Requerido por iDempiere 12 |
| **Maven** | **3.9.11** (`/home/user/idempiere12/apache-maven-3.9.11`) | Aislado en la carpeta del proyecto |
| **Eclipse** | Carpeta portable `~/idempiere12/eclipse` | Sin instalar, hereda variables |
| **Repo Maven** | `~/idempiere12/.m2/repository` | No usa `~/.m2` global |
| **Workspace** | `~/idempiere12/workspace-12` | Separado del resto |

> **Tip**: Siempre **abre la terminal dentro de `~/idempiere12`**. Así `direnv` aplica el entorno automáticamente.

---

## 📁 Estructura de carpetas

```text
/home/user/idempiere12
├─ apache-maven-3.9.11/           # Maven local (bin incluidos)
├─ eclipse/                       # Eclipse portable (descargado)
├─ sources/
│  ├─ idempiere/                  # Código iDempiere (branch release-12)
│  └─ idempiere-target-platform-plugin/
├─ plugins/
│  ├─ com.cds.intcomex/
│  └─ com.cds.intcomex.trax/com.cds.intcomex.trax/
├─ workspace-12/                  # Workspace de Eclipse
├─ .m2/                           # Repo Maven aislado
│  └─ settings.xml
├─ .mvn/                          # Config Maven por-repo (fallback)
│  └─ maven.config
└─ .envrc                         # Variables de entorno (direnv)
```

> Usa **`/home/user`** en la documentación; así evitas exponer tu usuario real.

---

## 🧰 Requisitos

1. **Instalar direnv**
   ```bash
   sudo apt-get update && sudo apt-get install -y direnv
   ```
2. **Hook de shell**
   - **zsh** → añade a `~/.zshrc`:
     ```bash
     eval "$(direnv hook zsh)"
     ```
   - **bash** → añade a `~/.bashrc`:
     ```bash
     eval "$(direnv hook bash)"
     ```
3. Recarga el shell:
   ```bash
   source ~/.zshrc  # o ~/.bashrc
   ```

---

## 🌳 `.envrc` (direnv)

Crea **`/home/user/idempiere12/.envrc`**:

```bash
# === iDempiere 12 - Entorno Aislado ===
export IDEMPIERE_ROOT="/home/user/idempiere12"

# Java 17 del sistema
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"

# Maven local
export MAVEN_HOME="$IDEMPIERE_ROOT/apache-maven-3.9.11"
export M2_HOME="$MAVEN_HOME"

# Apps y fuentes
export IDEMPIERE_HOME="$IDEMPIERE_ROOT/sources/idempiere"
export ECLIPSE_HOME="$IDEMPIERE_ROOT/eclipse"
export ECLIPSE_WORKSPACE="$IDEMPIERE_ROOT/workspace-12"

# Maven (repo & settings) — aislados
export MAVEN_USER_SETTINGS="$IDEMPIERE_ROOT/.m2/settings.xml"
export MAVEN_LOCAL_REPO="$IDEMPIERE_ROOT/.m2/repository"

# PATH (prioriza Maven local)
export PATH="$MAVEN_HOME/bin:$PATH"

# Alias: fuerza settings/repo locales
alias mvn='"$MAVEN_HOME/bin/mvn" -s "$MAVEN_USER_SETTINGS" -Dmaven.repo.local="$MAVEN_LOCAL_REPO"'

# Calidad de vida para builds/headless
export JAVA_TOOL_OPTIONS="-Djava.awt.headless=true"
```

**Activa** (una sola vez por carpeta):
```bash
cd /home/user/idempiere12
direnv allow
```

> Verás `PS1 cannot be exported` — **es normal**.

---

## 📦 Maven aislado: `.m2` y `.mvn`

### 1) `~/idempiere12/.m2/settings.xml`

```bash
mkdir -p /home/user/idempiere12/.m2
cat > /home/user/idempiere12/.m2/settings.xml << 'XML'
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <!-- Repositorio local aislado -->
  <localRepository>/home/user/idempiere12/.m2/repository</localRepository>

  <!-- Mirrors/Proxy/Profiles opcionales -->
  <mirrors>
    <!--
    <mirror>
      <id>central-fast</id>
      <mirrorOf>central</mirrorOf>
      <url>https://repo1.maven.org/maven2/</url>
    </mirror>
    -->
  </mirrors>
</settings>
XML
```

### 2) `.mvn/maven.config` en **cada** repo que compiles

> Maven lee `.mvn/maven.config` **desde el raíz del proyecto**.  
> Añádelo en *todos* los repos que uses (iDempiere, target-platform, tus plugins):

```bash
for p in \
  /home/user/idempiere12/sources/idempiere \
  /home/user/idempiere12/sources/idempiere-target-platform-plugin \
  /home/user/idempiere12/plugins/com.cds.intcomex \
  /home/user/idempiere12/plugins/com.cds.intcomex.trax/com.cds.intcomex.trax
do
  mkdir -p "$p/.mvn"
  cat > "$p/.mvn/maven.config" << 'CFG'
-s /home/user/idempiere12/.m2/settings.xml
-Dmaven.repo.local=/home/user/idempiere12/.m2/repository
# Opcional:
# -Dtycho.localArtifacts=ignore
CFG
done
```

---

## 🧪 Pruebas rápidas

```bash
# 1) Entra al entorno
cd /home/user/idempiere12
direnv allow    # solo si cambiaste .envrc

# 2) Verifica Maven/Java del entorno
which mvn       # /home/user/idempiere12/apache-maven-3.9.11/bin/mvn
mvn -v          # Maven 3.9.11, Java 17

# 3) Build iDempiere (ejemplo)
cd sources/idempiere
mvn -DskipTests verify
```

> Si ves que usa `~/.m2` global: revisa `which mvn` y el contenido de `.mvn/maven.config` en el repo activo.

---

## 🧠 Eclipse con entorno aislado

> **Siempre** lánzalo desde el entorno para heredar variables:

```bash
cd /home/user/idempiere12
direnv allow
"$ECLIPSE_HOME/eclipse" -data "$ECLIPSE_WORKSPACE" -vm "$JAVA_HOME/bin/java"
```

**Configura dentro de Eclipse:**

1. **Java → Installed JREs**
   - *Add… → Standard VM* → **JRE home**: `/usr/lib/jvm/java-17-openjdk-amd64`
   - Marcar como **Default**.

2. **Maven → User Settings**
   - **User Settings**: `/home/user/idempiere12/.m2/settings.xml`
   - **Update Settings** y **Reindex**.

3. **(Opcional) Run Configurations → Maven Build**
   - Pestaña **JRE** → **Alternate JRE** = **Java 17** (si no está por defecto).

> Si Eclipse se abrió fuera del entorno, ciérralo y relánzalo con el comando de arriba.

---

## 🧱 `plugin-builder` con entorno aislado

```bash
cd /home/user/idempiere12/sources/idempiere-target-platform-plugin

# Ejemplos:
./plugin-builder ../plugins/com.cds.intcomex -Drevision=12.0.0
./plugin-builder ../plugins/com.cds.intcomex.trax/com.cds.intcomex.trax -Drevision=12.0.0
```

- Gracias al `PATH` y a `.mvn/maven.config`, `plugin-builder` utilizará:
  - **Maven local** `apache-maven-3.9.11`
  - **Java 17**
  - **Repo/Settings** dentro de `~/idempiere12/.m2/`

---

## 🧯 Troubleshooting

| Síntoma / Log | Causa probable | Solución |
|---|---|---|
| `PS1 cannot be exported` (direnv) | Aviso inofensivo | Ignorar |
| Maven usa `~/.m2` global | No heredó el alias/variables o falta `.mvn/maven.config` | Verifica `which mvn` y crea `.mvn/maven.config` en el repo activo |
| Eclipse no usa Java 17 | JRE por defecto distinto | Preferences → **Java → Installed JREs** → setear **17** como Default |
| `BUILD Version Error` al iniciar iDempiere | DB seed no coincide con versión | Reimporta seed correcto y ejecuta empaquetado/migraciones de 12.x |
| `Could not find ... in p2` | Caché p2/tycho inconsistente | Elimina `~/.m2/.cache/tycho` **del entorno aislado** y recompila |

---

## ✅ Checklist final

- [ ] Hook de `direnv` en `~/.zshrc`/`~/.bashrc`
- [ ] `~/idempiere12/.envrc` creado y **`direnv allow`** aplicado
- [ ] `~/idempiere12/.m2/settings.xml` con `<localRepository>…/idempiere12/.m2/repository</localRepository>`
- [ ] `.mvn/maven.config` en **cada** repo que compiles
- [ ] Eclipse lanzado con `-data "$ECLIPSE_WORKSPACE"` desde `~/idempiere12`
- [ ] `which mvn` apunta a `apache-maven-3.9.11/bin/mvn` del entorno

---

## 🧾 Apéndice: comandos útiles

```bash
# Ver settings efectivos (confirma repo local)
mvn -X help:effective-settings | grep -A2 localRepository

# Limpiar caché Tycho (SÓLO del entorno aislado)
rm -rf /home/user/idempiere12/.m2/repository/.cache/tycho

# Forzar Maven del entorno (si acaso)
/home/user/idempiere12/apache-maven-3.9.11/bin/mvn -s /home/user/idempiere12/.m2/settings.xml \
  -Dmaven.repo.local=/home/user/idempiere12/.m2/repository -v
```

---

> ¿Quieres que agregue perfiles de proxy corporativo, mirrors o variables adicionales para PostgreSQL?  
> Puedo dejar los bloques listos para copiar/pegar en `settings.xml` o en `.envrc`. 👌
