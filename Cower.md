Cower
=====

Instructions on installing and using Cower to maintain Arch AUR packages. 

### 1. Downdoad the tarball
### 2. Untar the package:

```bash
tar xzvf cower.tar.gz 
```

### 3. cd into the directory:

```bash
cd cower
```

### 4. Use the _makepkg_ with sync flag _-s_:

```bash
makepkg -s
```

### 5. Install Cower to your sytem:

```bash
makepkg -i
```
### 6. Usage
#### 6.1 Search function:
Searches the AUR for packages

```bash
cower -s {package name}
```

#### 6.2 Download and install
Download the package then install using the __makepkg__ command:

```bash
cower -d {package name}
```

### 7. Updating AUR packages
Use the __cower__ command with the __-u__ flag for finding packages that needs updating and __-d__ for downloading them.

```bash
cower -ud
```

#### 7.1 Installing updates:
The following pattern is useful for finding the exeuting the __makepkg__ command on all the PKBUILD so that we don't have to do it for each package

```bash
find . -name PKGBUILD -execdir makepkg -si \;
```

Instructional video can be found [here](https://www.youtube.com/watch?v=JKPBfyJUeMg)
