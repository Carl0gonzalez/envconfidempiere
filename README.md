# 🚧 Entorno aislado para iDempiere 12 en Ubuntu 22.04 (zsh/bash)  

> **Objetivo**: Mantener **todas** las variables y configuraciones (Java 17, Maven 3.9.11, Eclipse, repositorio Maven) **aisladas** dentro de `~/idempiere12`, sin “ensuciar” tu `$HOME` global.

---

## 📁 Estructura recomendada de carpetas

```text
/home/user/idempiere12
├─ apache-maven-3.9.11/           # Maven local (bin incluidos)
├─ eclipse/                       # Eclipse descargado (sin instalar)
├─ sources/
│  ├─ idempiere/                  # Código de iDempiere (release-12)
│  └─ idempiere-target-platform-plugin/
├─ plugins/
│  ├─ com.cds.intcomex/
│  └─ com.cds.intcomex.trax/com.cds.intcomex.trax/
├─ workspace-12/                  # Workspace de Eclipse
├─ .m2/                           # Repositorio Maven (aislado)
│  └─ settings.xml
├─ .mvn/                          # Configuración por-repo de Maven
│  └─ maven.config
└─ .envrc                         # Variables de entorno (direnv)
```

> Usa **`/home/user`** en lugar de tu usuario real para no exponerlo. Sustituye `user` por lo que corresponda en **tu** máquina.

---

## 🧰 Requisitos (una sola vez)

### 1) Instala y activa **direnv**

```bash
sudo apt-get update && sudo apt-get install -y direnv
```

**zsh** – añade a `~/.zshrc`:
```bash
eval "$(direnv hook zsh)"
```

**bash** – añade a `~/.bashrc`:
```bash
eval "$(direnv hook bash)"
```

> Cierra y vuelve a abrir la terminal, o ejecuta `source ~/.zshrc` / `source ~/.bashrc`.

---

## 🌳 Archivo `~//idempiere12/.envrc` (variables del entorno aislado)

Crea el archivo **`/home/user/idempiere12/.envrc`** con este contenido:

```bash
# === iDempiere 12 - Entorno aislado ===
# Ruta raíz del entorno
export IDEMPIERE_ROOT="/home/user/idempiere12"

# Java 17 del sistema (Ubuntu 22.04)
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"

# Maven local
export MAVEN_HOME="$IDEMPIERE_ROOT/apache-maven-3.9.11"
export M2_HOME="$MAVEN_HOME"

# Apps y fuentes
export IDEMPIERE_HOME="$IDEMPIERE_ROOT/sources/idempiere"
export ECLIPSE_HOME="$IDEMPIERE_ROOT/eclipse"
export ECLIPSE_WORKSPACE="$IDEMPIERE_ROOT/workspace-12"

# Maven (repo & settings) – aislados aquí dentro
export MAVEN_USER_SETTINGS="$IDEMPIERE_ROOT/.m2/settings.xml"
export MAVEN_LOCAL_REPO="$IDEMPIERE_ROOT/.m2/repository"

# PATH: prioriza Maven local
export PATH="$MAVEN_HOME/bin:$PATH"

# Alias: garantiza repo/settings locales aunque algún build ignore .mvn
alias mvn='"$MAVEN_HOME/bin/mvn" -s "$MAVEN_USER_SETTINGS" -Dmaven.repo.local="$MAVEN_LOCAL_REPO"'

# Calidad de vida para builds/headless
export JAVA_TOOL_OPTIONS="-Djava.awt.headless=true"
```

**Activa las variables del entorno (una vez por carpeta):**
```bash
cd /home/user/idempiere12
direnv allow
```

> Verás avisos como `PS1 cannot be exported` – **es normal** y no afecta.

---

## 📦 Maven aislado: `.m2` y `.mvn`

### 1) Crea `~//idempiere12/.m2/settings.xml`

```bash
mkdir -p /home/user/idempiere12/.m2
cat > /home/user/idempiere12/.m2/settings.xml << 'XML'
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <!-- Repositorio local aislado -->
  <localRepository>/home/user/idempiere12/.m2/repository</localRepository>

  <!-- Opcional: mirrors/proxy/perfiles si los necesitas -->
  <mirrors>
    <!-- Ejemplo de mirror (descomenta y ajusta si corresponde)
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

### 2) Crea `.mvn/maven.config` **en cada repo que compiles con `mvn`**

Maven sólo lee `.mvn/maven.config` desde el **raíz del proyecto** que estás compilando. Por eso, crea este archivo en **cada** repo relevante, por ejemplo:

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
# Opcionalmente:
# -Dtycho.localArtifacts=ignore
CFG
done
```

> Con esto, **aunque no uses el alias** `mvn` del `.envrc`, los builds seguirán usando **tu repo local aislado** y **settings.xml**.

---

## 🧪 Probar desde terminal

```bash
# siempre arranca desde la carpeta del entorno
cd /home/user/idempiere12
direnv allow        # solo la primera vez o si cambiaste .envrc

# confirman versiones y rutas
which mvn           # => /home/user/idempiere12/apache-maven-3.9.11/bin/mvn
mvn -v              # Apache Maven 3.9.11, Java version: 17

# build de iDempiere (ejemplo)
cd sources/idempiere
mvn -DskipTests verify
```

> Si ves que Maven usa `~/.m2` global, revisa:
> - `which mvn` (debe ser el de `apache-maven-3.9.11/bin`)
> - `mvn -X help:effective-settings | grep -A2 localRepository`

---

## 💡 Eclipse usando el entorno aislado

### Lanzar Eclipse **con** variables del entorno
Siempre lánzalo desde la carpeta del entorno (para heredar `.envrc`):

```bash
cd /home/user/idempiere12
direnv allow
"$ECLIPSE_HOME/eclipse" -data "$ECLIPSE_WORKSPACE" -vm "$JAVA_HOME/bin/java"
```

### Preferencias dentro de Eclipse (m2e)

1. **Java → Installed JREs**  
   - **Add… → Standard VM** → **JRE home**: `/usr/lib/jvm/java-17-openjdk-amd64`  
   - Marcar como **Default**.

2. **Maven → User Settings**  
   - **User Settings**: `/home/user/idempiere12/.m2/settings.xml`  
   - **Update Settings** / **Reindex**.  
   - (Opcional) En el mismo `settings.xml` ya fijamos `<localRepository>…</localRepository>` para que m2e use `/home/user/idempiere12/.m2/repository`.

3. **(Opcional) Run Configurations → Maven Build**  
   - Para jobs específicos: pestaña **JRE** → **Alternate JRE** = **Java 17**.
   - Pestaña **JRE** o **JRE HOME** no es necesaria si ya pusiste Java 17 por defecto.

> **Nota**: Si Eclipse ya estaba abierto **sin** heredar `.envrc`, ciérralo y vuelve a abrirlo con el comando de arriba.

---

## 🧱 `plugin-builder` con el entorno aislado

Desde el terminal (ya dentro de `~/idempiere12` con `direnv` activo):

```bash
cd /home/user/idempiere12/sources/idempiere-target-platform-plugin

# ejemplo: construir tu plugin con revision 12.0.0
./plugin-builder ../plugins/com.cds.intcomex.trax/com.cds.intcomex.trax -Drevision=12.0.0
```

> Gracias a `PATH` (Maven local primero) y a los `.mvn/maven.config` en cada repo, `plugin-builder` también usará **Java 17**, **Maven local** y **repo/settings aislados**.

---

## 🧯 Troubleshooting rápido

- **Maven usa `~/.m2` global**  
  - Verifica `which mvn` → debe apuntar a `apache-maven-3.9.11/bin/mvn` dentro del entorno.  
  - Asegúrate de tener `.mvn/maven.config` en **cada** repo que compiles.

- **Eclipse no toma Java 17**  
  - Preferences → Java → Installed JREs → que JDK 17 esté **Default**.  
  - Si hace falta, en cada *Run Configuration* usa **Alternate JRE** = 17.

- **`direnv: PS1 cannot be exported`**  
  - Es un aviso inofensivo. Ignóralo.

---

## ✅ Checklist final

- [ ] `~/.zshrc` o `~/.bashrc` tiene el hook de direnv  
- [ ] `/home/user/idempiere12/.envrc` creado y `direnv allow` aplicado  
- [ ] `/home/user/idempiere12/.m2/settings.xml` con `<localRepository>…/idempiere12/.m2/repository</localRepository>`  
- [ ] `.mvn/maven.config` presente en **cada repo** que compilas  
- [ ] Eclipse lanzado desde `~/idempiere12` para heredar el entorno  
- [ ] `which mvn` apunta al Maven local del entorno

---

### 📝 Notas
- Sustituye siempre `/home/user` por la ruta real de tu usuario si lo deseas, **pero mantener “user” es válido** y evita exponer tu username en documentación o scripts compartidos.
- Puedes versionar **`.mvn/maven.config`** dentro de cada repo para que toda tu organización use el mismo repo Maven aislado cuando trabajen en este entorno.
