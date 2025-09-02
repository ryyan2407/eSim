# eSim Installation on Ubuntu 25.04 Bug Report

## Bug #1: Installer does not support Ubuntu 25.04

### Description:
- The main install_eSim.sh script checks OS(Version_ID) and only allows 22.04, 23.04, and 24.04
- When using Ubuntu 25.04, the script exits with “Unsupported Ubuntu Version: 25.04”, blocking the installation
- Upon checking the files in the ‘install-eSim-scripts’ directory, no ‘install-eSim-25.04.sh’ exists

### Cause:
- The script logic doesn’t contain a case for “25.04”, so it always hits the “unsupported” notice and exits

### Workaround:
- Copied the contents of ‘install_eSim-24.04.sh’ into a new file: ‘install_eSim-25.04.sh’ for any further changes
- Edited install_eSim.sh script to add the following to the case block:

	> ` “25.04”) SCRIPT=“$SCRIPT_DIR/install_eSim-25.04” ;; `

Now, the installer can proceed, allowing further debugging based on dependencies

## Bug #2: KiCad 6.0 PPA Repository Missing and Broken – Dependency Installation Blocker Bug

### Description:
- During eSim installation on Ubuntu 25.04, the script tries to add the KiCad 6.0 PPA to install the required packages
- This results in a 404 error when updating package lists, aborting the installation
- The KiCad PPA did not have Release files or GPG keys valid for Ubuntu 25.04, with repeated "NO_PUBKEY" errors
- Multiple attempts to import and dearmor GPG keys failed due to unsupported file formats and signature verification errors

### Cause:
- Ubuntu 25.04 is quite new, and most third-party PPAs only provide packages up to Ubuntu 24.04 as of now and some only up to Ubuntu 23.04
- This leads to an unsupported codename error, which leads to the 404 error

### Solution: Build KiCad from Source
- Due to unresolved repository and key issues, building KiCad directly from the source was chosen as the workaround, ensuring compatibility
- This build process is repeatable and can be containerized in a Docker environment, ensuring clean installations

#### (1) Building KiCad from Source on Ubuntu 25.04
- Initial Dependencies Installation:
	> ` sudo apt install -y libwxgtk3.2-dev libwxgtk-media3.2-dev libwxgtk-webview3.2-dev `
    > 
 	> ` git cmake g++ pkg-config libboost-all-dev libssl-dev libcairo2-dev libglib2.0-dev `
    > 
	> ` libfreetype-dev libcurl4-openssl-dev python3-dev python3-pip doxygen graphviz `
	
- Installed additional dependencies as errors appeared during CMake
	- libglew-dev (for GLEW support)
	- libglm-dev (for GLM)
	- bison and flex (for yacc parser errors)
	- python3-wxgtk4.0 or pip3 install wxPython (for wxPython Phoenix)
	- libgtk-3-dev (for GTK 3 support)
	- swig (version ≥ 3.0 for language bindings)
	- python3-dev (development headers for Python)

#### (2) Handling ngspice Shared Library
- KiCad requires libngspice shared library, missing from typical package installs
- However, KiCad 6.0 source branch lacks the ‘get_libngspice_so.sh’
- Manually cloned and built ngspice with shared library support:
>   ` git clone https://git.code.sf.net/p/ngspice/ngspice `
> 
>	` cd ngspice `
> 
>	` ./autogen.sh `
> 
>	` ./configure --with-ngshared `
> 
>	` make -j$(nproc) `
> 
>	` sudo make install `

#### (3.1) Building and Installing OpenCascade Technology (OCCT)
- OpenCascade packages were unavailable in Ubuntu 25.04 repositories
- Cloned the official OCCT repository and built from source:
>	` git clone https://git.dev.opencascade.org/gitweb?p=occt.git `
>
>	` cd occt `
>
>	` mkdir build && cd build `
>
>	` cmake .. `
>
>	` make -j$(nproc) `
>
>	` sudo make install `
>
>	` sudo ldconfig `
- Installed Tcl/Tk development headers to satisft OCCT build requirements:
> ` 	sudo apt install tcl-dev tk-dev `

#### (3.2) Fixing OpenCascade Library Naming Compatibility for KiCad
- Modern OCCT (8.0) renamed libraries, breaking KiCad’s legacy library name expectations
- Symlinked OCCT libraries to legacy names expected by KiCad, for example:
>	` cd /usr/local/lib `
>
>	` sudo ln -s libTKDEIGES.so libTKIGES.so `
>
>	` sudo ln -s libTKDESTEP.so libTKSTEP209.so `
>
>	` sudo ln -s libTKDESTEP.so libTKSTEPAttr.so `
>
>	` sudo ln -s libTKDESTL.so libTKSTL.so `
>
>	` sudo ln -s libTKDEVRML.so libTKVRML.so `
>
>	` sudo ln -s libTKDESTEP.so libTKSTEPBase.so `
>
>	` sudo ldconfig `

- Created similar symlinks for all other reported missing OCCT libraries during KiCad’s CMake config

#### (4) Explicitly Specifying Python Include and Library Paths in CMake
- Encountered errors where Python headers and library are not found in CMake
Python Code:
	> ` python3 -c "from sysconfig import get_paths; print(get_paths()['include'])" `
    > 
	> ` ldconfig -p | grep libpython3 `
- Run CMake with explicit Python paths

#### (5) Building and Installation KiCad
- Compiled KiCad:
>	` make -j$(nproc) `
>
> 	` sudo make install `
>
>	` sudo ldconfig `
- Verified installation by running KiCad in the terminal

Now, the KiCad installation works fine while installation of eSim.


## Bug #3: NGHDL Installer does not support Ubuntu 25.04

### Description:
- The NGHDL installer script checks the OS version string against a hardcoded list (22.04, 23.04, 24.04)
- When running on Ubuntu 25.04, the script prints “Unsupported Ubuntu version: 25.04 ()” and aborts the process

### Cause:
- The installer script (install-nghdl.sh) rejects any OS version not in the hardcoded list, causing immediate abortion of the process on Ubuntu 25.04
- There is no ‘install-nghdl-25.04.sh’ in the ‘nghdl/install-nghdl-scripts’ directory as well

### Workaround:
- Copied the contents of ‘install-nghdl-24.04.sh’ into a new file: ‘install-nghdl-25.04.sh’ for any further changes
- Edited install-nghdl.sh script to add the following to the case block:
	> ` “25.04”) SCRIPT=“$SCRIPT_DIR/install-nghdl-25.04” ;; `

### Additional Issue:
- The main eSim installer, any manual changes to ‘nghdl/install-nghdl.sh’ are overwritten as the script calls for extraction of NGHDL from the nghdl.zip file
- As a workaround, consider editing the neccessary scripts and then compressing them manually, ensuring that the edited version of the script is called.

Now the installer can proceed, allowing further debugging based on dependencies


## Bug #4: Installer Requires ‘libcanberra-gtk-module’ - Missing in Ubuntu 25.04

### Description:
- The eSim installer aborts the process as it requires ‘libcanberra-gtk-module’, which is not available in Ubuntu 25.04 repositories
- Installing ‘libcanberra-gtk3-module’ does not satisfy the installer’s dependency check

### Cause:
- ‘libcanberra-gtk-module’ is renamed or removed in Ubuntu 25.04, however the installer has a hardcoded package dependency that is now obsolete

### Workaround:
- Edited the ‘install-nghdl-25.-04.sh’ file that was created and removed ‘libcanberra-gtk-module’ from the following line:
	
 > ` 	echo "Installing Gtk Canberra modules..........................."  `
> 
 > `  	sudo apt install -y libcanterra-gtk-module libcanberra-gtk3-module `

## Bug #5: GHDL Build Fails with “Unhandled version llvm 20.1.2”

### Description:
- The GHDL 4.1.0 build process does not recognize llvm 20.1.2 which is installed on Ubuntu 25.04
- The error message “Unhandled version llvm 20.1.2” causes build failure and aborts installation

### Cause:
- GHDL 4.1.0 supports only certain LLVM versions older than 20.x
- The build scripts lack compatibility with newer LLVM versions

### Workaround:
- Install and configure a supported version of LLVM
- Set environmental variables for LLVM tools before building


## Bug #6: GHDL Build Fails Due to Missing clang++ After Installing LLVM 14
### Description: 
- After installing LLVM-14 (llvm-14, llvm-14-dev, llvm-14-tools, clang-14) on Ubuntu 25.04, the GHDL 4.1.0 build initially proceeds, but then fails with the error
		>	` /bin/sh: 1: clang++: not found `
- The build process requires the clang++ C++ compiler binary to be available in the system path, but it cannot find it

### Cause:
- The installer script uses clang++ as the C++ compiler during the LLVM backend build
- On some systems, only the versioned binary (clang++-14) is installed and the generic symlink might be missing.

### Workaround:
- Ensure that Clang 14 is installed:
>	` sudo apt-get install clang-14`
- Create a symlink so the system recognizes clang++:
>	` sudo ln -sf /usr/bin/clang++-14 /usr/bin/clang++ `
- Verify installation:
>	` clang++ --version `
- Restart the installation and confirm the GHDL build finds and uses the correct C++ compiler
