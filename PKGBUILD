# Maintainer: Alexander Bocken <alexander@bocken.org>
# Contributor: Posi <posi1981@gmail.com>
# Contributor: Johannes Löthberg <johannes@kyriasis.com>
# Contributor: Sergej Pupykin <pupykin.s+arch@gmail.com>
# Contributor: Timm Preetz <timm@preetz.us>
# Contributor: Michael 'manveru' Fellinger <m.fellinger@gmail.com>
# Contributor: Dave Pretty <david dot pretty at gmail dot com>

# anki -> git rev-parse --short=8 $pkgver
# ftl -> git submodule
declare -gA _tags=(
 [ftl_core]="e22784ecfa30855bad1cabc3fde9cac0457f14ca"
 [ftl_desktop]="e747f9e539c86c3f85b98090898f95c47703c9fb"
 [anki]="9b4dd54"
)
declare -gA _caches=(
    [yarn]="yarn-cache"
    [cargo]="cargo-cache"
)

pkgname=anki
pkgver=24.06
pkgrel=1
pkgdesc="Helps you remember facts (like words/phrases in a foreign language) efficiently"
url="https://apps.ankiweb.net/"
license=('AGPL3')
arch=('x86_64')
conflicts=('anki-bin' 'anki-git' 'anki-qt5')
depends=(
    # anki & aqt
    'python>=3.9'
    'python-beautifulsoup4'
    'python-waitress>=2.0.0'
    'python-requests'

    # anki
    'python-decorator'
    'python-markdown'
    'python-orjson'
    'python-protobuf>=4.21'
    'python-pysocks'
    'python-distro'

    #aqt
    'python-flask-cors' # python-flask required for anki & aqt but a dependency of -cors
    'python-jsonschema'
    'python-send2trash'
    'python-certifi'
    'qt6-multimedia'	# recording voice
    'python-pyqt6-webengine>=6.2'
    'qt6-svg'
)
makedepends=(
    'rsync'
    'ninja'
    'git'
    'cargo'
    'python-installer'
    'python-wheel'
    'nodejs>=18'
    'yarn'
    'mold'
)
optdepends=(
    'lame: record sound'
    'mpv: play sound. prefered over mplayer'
    'mplayer: play sound'
    'texlive-most: render LaTex in cards'
)
changelog="$pkgname.changelog"
source=("$pkgname-$pkgver.tar.gz::https://github.com/ankitects/anki/archive/refs/tags/${pkgver}.tar.gz"
        "anki-core-i18n-${_tags[ftl_core]}.tar.gz::https://github.com/ankitects/anki-core-i18n/archive/${_tags[ftl_core]}.tar.gz"
        "anki-desktop-ftl-${_tags[ftl_desktop]}.tar.gz::https://github.com/ankitects/anki-desktop-ftl/archive/${_tags[ftl_desktop]}.tar.gz"
        "no-update.patch"
        "strip-formatter-deps.patch"
        "strip-type-checking-deps.patch"
        "strip-python-pip-system-certs.patch"
	"fix_sveltekit_yarn.patch"
)
sha256sums=('0531eba4fc6be9b9869be56d123276934365af8121d5b50b978131cbcc4fda9f'
            'b1b7042f98d7c7f2b9bb7054e7e514bd9292749b1a1748b1ad2e8f7375c3e825'
            'acc80996e6f7cb3d43a7ea0f0749fb8f6e910913cc46604fb58b633f52d3d2e2'
            'cc546f4e5af642af89f82be0375800c2721dd904c0a212cf46f6459495b75bff'
            '9858fefa254812980d252b29fc6f32bd19bb83ee7e5a96d72c707626ed5193a7'
            '198bc2ec14439e3ba41a03c4823f07df4b0c559c1dcbdaf678416ed12a720c2e'
            '2506cf9d5b0c47a2c519ec4bb0ef87e7921dca8db5cae39b0dae265d01e253b3'
            '6055c8a43ef2fb9ddebf73e05f6883cbc49dc955ae1451dc28d3d9a891ef9611'
)

prepare() {
    cd "$pkgname-$pkgver"

    patch -p1 < "$srcdir/no-update.patch"
    patch -p1 < "$srcdir/strip-formatter-deps.patch"
    patch -p1 < "$srcdir/strip-type-checking-deps.patch"
    patch -p1 < "$srcdir/strip-python-pip-system-certs.patch"
    patch -p1 < "$srcdir/fix_sveltekit_yarn.patch"
    sed -i 's/opt-level = 1$/opt-level= 3/' Cargo.toml	# optimize more
    sed -i 's/channel = "[0-9\.]*"$/channel = "stable"/' rust-toolchain.toml # use most recent stable rust toolchain
    # Build process wants .git/HEAD to be present. Workaround to be able to use tarballs
    mkdir -p out .git
    touch .git/HEAD
    echo "${_tags[anki]}" > out/buildhash	# manually write the buildhash into out/buildhash to avoid git dependency in build

    # place translations in build dir
    rm -r ftl/core-repo ftl/qt-repo
    ln -sT "${srcdir}"/anki-core-i18n-${_tags[ftl_core]} ftl/core-repo
    ln -sT "${srcdir}"/anki-desktop-ftl-${_tags[ftl_desktop]} ftl/qt-repo

    #force update for 'rustup' package users (not necesarry for 'rust' package users)
    pacman -Qo $(which cargo) | grep -q rustup && rustup update
    # fetch rust packages
    export CARGO_HOME="$srcdir/${_caches[cargo]}"       # do not litter in ~
    cargo fetch --locked --target "$CARCH-unknown-linux-gnu"


    # fetch node packages already in prepare()
    export YARN_CACHE_FOLDER="$srcdir/${_caches[yarn]}" # do not litter in ~
    yarn install --immutable --modules-folder out/node_modules --ignore-scripts
    ln -sf out/node_modules ./

    # mask pip-sync as we provide dependencies ourselves
    local venv="out/pyenv"
    python -m venv --system-site-packages --without-pip "$venv"
    printf '#!/bin/bash\nexit 0' > "$venv/bin/pip-sync"
    chmod +x "$venv/bin/pip-sync"
}

build() {
    cd "$pkgname-$pkgver"

    export YARN_CACHE_FOLDER="$srcdir/${_caches[yarn]}" # do not litter in ~

    #use local binaries instead of downloading them
    export PYTHON_BINARY=$(which python)
    export PROTOC_BINARY=$(which protoc)
    export NODE_BINARY=$(which node)
    export YARN_BINARY=$(which yarn)

    export CARGO_HOME="$srcdir/${_caches[cargo]}"    # do not litter in ~
    export RELEASE=2	        # anki-internal variable for optimization
    				# set to "1" for faster but less optimized build
    # if you want to use your linker of choice, comment out the following lines and disbale the mold line
    # This appears to create issues for memory-constrained (<= 8GB) systems
    # use fat LTO objects, allows for LTO, needed for rust crate "ring"
    # See https://gitlab.archlinux.org/archlinux/packaging/packages/pacman/-/issues/20 and https://github.com/briansmith/ring/issues/1444
    # export CFLAGS+=' -ffat-lto-objects'
    # ./ninja wheels -v
    export OFFLINE_BUILD=1 # do not download anything, disables git checks
    mold -run ./ninja wheels -v
}

package() {
    cd "$pkgname-$pkgver"
    for file in out/wheels/*.whl; do
    	python -m installer --destdir="$pkgdir" $file
    done

    install -Dm644 qt/bundle/lin/anki.desktop "$pkgdir"/usr/share/applications/anki.desktop
    install -Dm644 qt/bundle/lin/anki.png "$pkgdir"/usr/share/pixmaps/anki.png
}
