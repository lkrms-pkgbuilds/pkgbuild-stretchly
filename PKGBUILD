# Maintainer: Luke Arms <luke@arms.to>

_electron_version=31
pkgname=stretchly
pkgver=1.16.0
pkgrel=1
pkgdesc="The break time reminder app"
arch=('i686' 'x86_64')
url="https://hovancik.net/stretchly"
license=('BSD')
depends=('gtk3' 'http-parser' 'libappindicator-gtk3' 'libnotify' 'libxcrypt-compat' 'libxss' "electron$_electron_version")
makedepends=('git' 'nvm' 'jq' 'python')
conflicts=("${pkgname}-bin" "${pkgname}-git")
source=("https://github.com/hovancik/stretchly/archive/v${pkgver}.tar.gz")

sha256sums=('4445e12872ba5e950fbea3f30b5d5a057ca830a7c68483c41475df5baa9037ad')

_ensure_local_nvm() {
    if type nvm &>/dev/null; then
        nvm deactivate
        nvm unload
    fi
    unset npm_config_prefix
    export NVM_DIR=${srcdir}/.nvm
    . /usr/share/nvm/init-nvm.sh || return
}

prepare() {
    cd "${srcdir}/${pkgname}-${pkgver}"
    _ensure_local_nvm
    _node_version=$(jq -r '.engines.node' package.json)
    nvm ls "$_node_version" &>/dev/null ||
        nvm install "$_node_version" || return
}

build() {
    cd "${srcdir}/${pkgname}-${pkgver}"
    _ensure_local_nvm
    _node_version=$(jq -r '.engines.node' package.json)
    nvm use "$_node_version" || return
    # 'husky install' doesn't work outside of a git repository
    [[ -d .git ]] || git init
    npm install --no-save --no-audit --no-progress --no-fund
    # electron-builder only generates /usr/share/* assets for target package
    # types 'apk', 'deb', 'freebsd', 'p5p', 'pacman', 'rpm' and 'sh', so build a
    # pacman package and unpack it
    local _appname _appver _outfile _unpackdir=${srcdir}/${pkgname}-${pkgver}.unpacked
    _appname=$(jq -r '.name' package.json)
    _appver=$(jq -r '.version' package.json)
    _outfile=dist/${_appname}-${_appver}.pacman
    rm -rf "${_unpackdir}"
    mkdir -p "${_unpackdir}"

    local i686=ia32 x86_64=x64
    # Add -c.asar=false to suppress creation of an app archive
    ./node_modules/.bin/electron-builder build \
        --linux pacman \
        --"${!CARCH}" \
        -c.electronDist="/usr/lib/electron$_electron_version" \
        -c.electronVersion="$(<"/usr/lib/electron$_electron_version/version")"
    tar -C "${_unpackdir}" -xf "${_outfile}"

    # Replace absolute path in desktop entry
    sed -Ei "s/^(Exec=).*/\1\\/usr\\/bin\\/${pkgname} %U/" \
        "${_unpackdir}/usr/share/applications/${pkgname}.desktop"
}

package() {
    local _appname _appdir _appfile=
    _appname=$(jq -r '.name' "${srcdir}/${pkgname}-${pkgver}/package.json")
    _appdir=opt/${_appname}/resources/app
    cd "${srcdir}/${pkgname}-${pkgver}.unpacked"
    mv "usr" "${pkgdir}/"
    install -d "${pkgdir}/usr/lib"
    [[ -d $_appdir ]] || {
        _appdir=${_appdir%/*}
        _appfile=/app.asar
    }
    mv "$_appdir" "${pkgdir}/usr/lib/${pkgname}"
    install -Dm644 "${srcdir}/${pkgname}-${pkgver}/LICENSE" "${pkgdir}/usr/share/licenses/${pkgname}/LICENSE"
    install -Dm755 /dev/stdin "${pkgdir}/usr/bin/${pkgname}" <<END
#!/bin/sh
exec /usr/lib/electron${_electron_version}/electron /usr/lib/$pkgname$_appfile "\$@"
END
}
