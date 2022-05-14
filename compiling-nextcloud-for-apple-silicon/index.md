# Compiling Nextcloud for Apple Silicon


My 2016 MacBook Pro has really been showing its age, and the butterfly keyboard has been increasingly tough to type on, so I've been in the market for a replacement. While I don't love the closed source approach Apple is taking with their Apple Silicon ARM64 Chips, their recent MacBook Pro lineup sets a totally new standard for power in a notebook.

I got my order in for the M1 Max, which is my first ARM64 based notebook, and started migrating over to it. As the first M1 based notebooks were released over a year ago, virtually all of the programs I want available have already been compiled to use the native M1 architecture, with one notable exception.

Nextcloud, which I use as a partial G-Suite replacement, is still only compiled for Intel based macs which require Rosetta 2 in order to run. This wouldn't be that big of a deal as Rosetta 2 seems to be pretty seamless, but there are several reports of [huge amounts of CPU usage](https://github.com/nextcloud/desktop/issues/2659#issuecomment-830787727) by the Intel version of the app. Because Nextcloud is always running in the background, this isn't a "once and awhile" issue.

Several enterprising folks with a little help from the Nextcloud team have successfully compiled a version of the client side Nextcloud app for Apple Silicon. Building on the work they put together in Github thread I was able to get a working installation, but it took some finesse. For anyone who wants to get a version compiled themselves, here are the specific steps I followed:

### 1. Install Homebrew

If you don't have it installed, open up a terminal and get the excellent package manager homebrew up and running. More information on Homebrew can be found [here](https://docs.brew.sh/Installation). You'll be prompted to download XCode command line tools, and will have to enter your login password to install:
```zsh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
### 2. Set up the working directory

Create a folder in your home directory to build the related packages:
```zsh
mkdir ~/ncsilicon
```

### 3. Install Dependencies from Homebrew

Homebrew will save us from having to compile many the dependencies manually. It looks like some people were running a version of Inkscape through Rosetta for icon generation. Inkscape is actually syntax compatible with librsvg, which has Apple Silicon support out of the box:
```zsh
brew install pcre2 harfbuzz freetype cmake librsvg
```

### 4. Build Qt5 for Apple Silicon

Even though Qt6.2 was just released with native Apple Silicon support, the Nextcloud application isn't compatible with it yet. Nextcloud still uses Qt5, and version 5.15 is the latest freely and publically available version. There are newer versions than 5.15 of Qt5 available behind a paywall, but 5.15 works really well for our needs. As there is no Apple Silicon binary available, we have to build it ourselves.

Clone the Qt5 repo and check out version 5.15:
```zsh
cd ~/ncsilicon
git clone git://code.qt.io/qt/qt5.git
cd qt5
git checkout 5.15
```

Next we need to pull down all the related modules for the build. This will take a while, and is probably the longest step of this whole process:
```zsh
./init-repository
```

Qt5 actually has a missing header we need to add in order for it to compile correctly. This appears to be ignored by Big Sur's version of the clang compiler (clang 12), but Monterey (clang 13) complains and fails if you try to compile without it. Add this header with:
```zsh
sed -i -e 's/#include <qpa\/qplatformgraphicsbuffer.h>/#include <CoreGraphics\/CGColorSpace.h> \n#include <qpa\/qplatformgraphicsbuffer.h>/g' ~/ncsilicon/qt5/qtbase/src/plugins/platforms/cocoa/qiosurfacegraphicsbuffer.h 
```

Best practice is to build in a different folder than your source code, so the commands below create a new folder and get the stage set for the build:
```zsh
cd ~/ncsilicon
mkdir qt5-5.15-macOS-release
cd qt5-5.15-macOS-release
../qt5/configure -release -prefix ./qtbase -nomake examples -nomake tests QMAKE_APPLE_DEVICE_ARCHS=arm64 -opensource -confirm-license -skip qt3d -skip qtwebengine
```

Assuming no errors are thrown, we're ready to build. This will take a bit, but not as long as initiating the repository did. We don't need to run make install here because we built the libraries in the same location we'll be using them.
```zsh
make -j10
```

### 5. Build qtkeychain for Apple Silicon

Now that Qt5 is done, we need to build qtkeychain. Same deal as above, we need to clone the repo and set up a build folder:
```zsh
cd ~/ncsilicon
git clone git@github.com:frankosterfeld/qtkeychain.git
cd qtkeychain
mkdir build
cd build
```

Then build the binary. We do need to run make install here in order for the libraries to be included in the Qt5 folder.
```zsh
cmake .. -DCMAKE_INSTALL_PREFIX=~/ncsilicon/qt5-5.15-macOS-release/qtbase -DCMAKE_BUILD_TYPE=Release -DBUILD_TRANSLATIONS=OFF
make install
```

### 6. Build OpenSSL for Apple Silicon

Now that qtkeychain is done, we need to build OpenSSL. We need to clone the repo and switch to the version used by Nextcloud (1.1.1):
```zsh
cd ~/ncsilicon
git clone git@github.com:openssl/openssl.git
cd openssl
git checkout OpenSSL_1_1_1-stable
```

Then build the binary. OpenSSL uses caffinate to build:
```zsh
export MACOSX_DEPLOYMENT_TARGET=10.13
./Configure shared enable-rc5 zlib darwin64-arm64-cc no-asm
caffeinate make
```

Copy the libraries you want to /usr/local/lib. Alternatively you can run make install, but that'll install far more files including documentation:
```zsh
sudo cp libssl.1.1.dylib libcrypto.1.1.dylib libcrypto.a libssl.a /usr/local/lib
cd /usr/local/lib
sudo ln -s libssl.1.1.dylib libssl.dylib
sudo ln -s libcrypto.1.1.dylib libcrypto.dylib
```

### 7. Build Nextcloud Apple Silicon

Now that we have installed the dependencies from Homebrew, and compiled Apple Silicon versions of Qt5, Qtkeychain, and OpenSSL, we're ready to build the Nextcloud desktop app.

First clone the repo and set up a build folder:
```zsh
cd ~/ncsilicon
git clone git@github.com:nextcloud/desktop.git
cd desktop
mkdir build
cd build
```

We need to set up environment variables for the build, these tell the compiler where to find all of the libraries we just compiled such as Qt5 and qtkeychain.
```zsh
export OPENSSL_ROOT_DIR=~/ncsilicon/openssl
export Qt5LinguistTools_DIR=~/ncsilicon/qt5-5.15-macOS-release/qtbase
export Qt5_DIR=~/ncsilicon/qt5-5.15-macOS-release/qtbase
export Qt5Keychain_DIR=~/ncsilicon/qt5-5.15-macOS-release/qtbase/lib/cmake/Qt5Keychain
```

Now its build time. This will take a little time but nothing too long. We'll use make install here which will package the application up in the Applications directory.
```zsh
cmake .. -DCMAKE_INSTALL_PREFIX=/Applications -DCMAKE_BUILD_TYPE=Release
make -j10
make install
```

### 8. Start using the native Nextcloud binary!
Go ahead and launch the app and you should be set! If MacOS complains about the app not being from a known developer, hold down the Control button and right click on it, and select "Open". It will prompt you with a few "are you sure" messages but then you're set.

### 9. Optional - Sign the application yourself
If you're signed up for the developer program and have a certificate, you can sign the app yourself by replacing "YOUR NAME AND DEVELOPER ID" with your organization name and ID, assuming your certificate is installed:
```zsh
codesign --force --sign "Developer ID Application: YOUR NAME AND DEVELOPER ID" --deep --verbose /Applications/Nextcloud.app
```
