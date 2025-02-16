# **Building Qt5 on macOS ARM64 for ParaView Compilation**

## **1. Identifying the Issue**
ParaView requires **Qt5**, but Homebrew installs **Qt6** by default, causing multiple compilation errors.  

To avoid conflicts, **Qt5 was manually installed** with:
```bash
brew install qt@5
```
However, this led to several issues:
- The installed version was **not properly linked** in `/opt/homebrew`.
- CMake and Ninja **continued detecting Qt6 instead of Qt5**.
- Running **rcc** failed due to **incompatibilities in `libzstd.1.dylib`**.

Thus, the decision was made to **compile Qt5 from source** to ensure full compatibility.

---

## **2. Removing Previous Versions**
Before compiling, all previous Qt installations were removed to prevent conflicts:
```bash
brew uninstall --ignore-dependencies qt qt@5
brew autoremove
brew cleanup
```
Then, we verified that Qt6 was no longer present:
```bash
brew list | grep qt
```

Some remnants were found in **`/usr/local/homebrew`**, which were manually removed:
```bash
rm -rf /usr/local/homebrew/Cellar/qt
rm -rf /usr/local/homebrew/Cellar/qt@5
rm -rf /usr/local/homebrew/opt/qt*
rm -rf /usr/local/homebrew/share/qt*
```

Finally, Homebrew was updated:
```bash
brew update
brew upgrade
```

---

## **3. Manual Compilation of Qt5**
To **compile Qt5 from source**, follow these steps:

### **3.1 Cloning the Qt5 Repository**
```bash
git clone --branch 5.15.16 --depth 1 https://code.qt.io/qt/qt5.git
cd qt5
```
Initialize the required submodules:
```bash
./init-repository --module-subset=qtbase,qttools,qtsvg,qtxmlpatterns,qtdeclarative
```

### **3.2 Configuring for macOS ARM64**
```bash
mkdir build
cd build
../configure \
  -prefix /opt/qt5 \
  -opensource -confirm-license \
  -release \
  -platform macx-clang \
  -arch arm64 \
  -nomake examples -nomake tests \
  -skip qtwebengine
```

Skipping **qtwebengine** avoids **Chromium-related compatibility issues**.

---

## **4. Compiling Qt5**
Start the compilation process:
```bash
make -j$(sysctl -n hw.ncpu)
```
This process **took several hours** due to the number of modules being compiled.

If compilation failed, errors were analyzed with:
```bash
make -j1
```
After a successful build, Qt5 was installed with:
```bash
sudo make install
```

Verify that Qt5 was properly installed in `/opt/qt5`:
```bash
ls /opt/qt5/lib/cmake/Qt5
```

---

## **5. Final Configuration**
After installation, environment variables were set so that CMake could detect Qt5:
```bash
export Qt5_DIR="/opt/qt5/lib/cmake/Qt5"
export CMAKE_PREFIX_PATH="/opt/qt5/lib/cmake:$CMAKE_PREFIX_PATH"
```
To apply these settings permanently, they were added to `~/.zshrc`:
```bash
echo 'export Qt5_DIR="/opt/qt5/lib/cmake/Qt5"' >> ~/.zshrc
echo 'export CMAKE_PREFIX_PATH="/opt/qt5/lib/cmake:$CMAKE_PREFIX_PATH"' >> ~/.zshrc
source ~/.zshrc
```

Verify that CMake detected Qt5:
```bash
cmake --find-package -DNAME=Qt5 -DCOMPILER_ID=Clang
```

If CMake still did not detect Qt5, its path was manually set in `CMakeLists.txt`:
```cmake
set(Qt5_DIR "/opt/qt5/lib/cmake/Qt5")
find_package(Qt5 REQUIRED COMPONENTS Widgets Core Gui)
```

---

## **6. Verification**
To ensure the installation worked correctly, we checked:
```bash
qmake --version
```
Expected output:
```
QMake version 3.1
Using Qt version 5.15.16 in /opt/qt5/lib
```

Then, we verified that ParaView detected it:
```bash
cmake ../paraview
```

---

## **Conclusion**
1. **Qt6 was completely removed to prevent conflicts** during ParaView compilation.  
2. **Qt5 was compiled from source**, ensuring compatibility with macOS ARM64.  
3. **Paths were manually configured** to allow CMake to detect Qt5 in `/opt/qt5`.  
4. **Installation was verified** with `qmake` and `cmake`.  

This setup allowed **ParaView to compile without errors**, avoiding broken Qt dependencies.