# Guía de Instalación de Ruby y Jekyll

Este documento describe los pasos para instalar **Ruby** y **Jekyll** en **macOS/Linux** y en **Windows (Git Bash / RubyInstaller)**.

---

## 🔹 Instalación en macOS / Linux

### 1. Verificar si ya tienes Ruby
```bash
ruby -v
```
- Si aparece una versión (ej. `ruby 3.2.2`), ya tienes Ruby instalado.  
- Si no, continúa al siguiente paso.

---

### 2. Instalar Ruby
#### En macOS (recomendado con Homebrew)
```bash
brew install ruby
```
Agrega Ruby al `PATH` en tu `~/.zshrc` o `~/.bashrc`:
```bash
export PATH="/usr/local/opt/ruby/bin:$PATH"
```
Aplica cambios:
```bash
source ~/.zshrc
```

#### En Linux (ejemplo Ubuntu/Debian)
```bash
sudo apt update
sudo apt install ruby-full build-essential zlib1g-dev
```

Configura `PATH` (añadir al `~/.bashrc` o `~/.zshrc`):
```bash
export GEM_HOME="$HOME/gems"
export PATH="$HOME/gems/bin:$PATH"
```
Aplica cambios:
```bash
source ~/.bashrc
```

---

### 3. Verificar instalación de RubyGems y Bundler
```bash
gem -v
bundle install
gem install bundler
```

---

### 4. Instalar Jekyll
```bash
gem install jekyll
```

---

### 5. Dentro del repositorio clonado ejecuta el siguiente comando para levantar el sitio local

```bash
bundle exec jekyll serve --livereload --incremental --trace
o
bundle exec jekyll serve --livereload --force_polling
o
bundle exec jekyll serve
```

### 6. Para MAC ejecuta el siguiente comando para que compile correctamente cuando se suba a GitHub Actions

gemlock para MAC
```bash
bundle lock --add-platform x86_64-darwin-22
```

---

## 🔹 Instalación en Windows (con Git Bash)

> ⚠️ Recomendación: en Windows es más sencillo instalar **RubyInstaller** que usar Git Bash puro.

### 1. Instalar Ruby con RubyInstaller
1. Descarga el instalador desde 👉 [https://rubyinstaller.org/](https://rubyinstaller.org/)  
2. Ejecuta el instalador y marca la opción **“Add Ruby executables to PATH”**.  
3. Cuando termine, se abrirá **MSYS2** → ejecuta la opción `3` para instalar dependencias.

Verifica instalación:
```bash
ruby -v
```

---

### 2. Instalar Bundler y Jekyll
Abre **Git Bash** o **PowerShell** y ejecuta:
```bash
bundle install
gem install bundler jekyll
```

---

### 3. Dentro del repositorio clonado ejecuta el siguiente comando para levantar el sitio local

```bash
bundle exec jekyll serve --livereload --incremental --trace
o
bundle exec jekyll serve --livereload --force_polling
```

### 4. Para Windows ejecuta el siguiente comando para que compile correctamente cuando se suba a GitHub Actions

gemlock para Windows
```bash
bundle lock --add-platform x86_64-linux
```