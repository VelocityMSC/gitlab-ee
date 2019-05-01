# Gitlab Enterprise Edition v11.10.0
# Maintainer: Steven Cook <scook@velocity.org>

# Note: This package conflicts with Gitlab, which provides the CE (community edition).
# DO NOT upload this package to the Arch Linux AUR; this package is solely for use by Velocity only.
# This package must be manually upgraded for now. In the future, we may provide a custom Arch Linux repository.

_pkgname=gitlab

pkgname=velocity-${_pkgname}-ee
pkgver=11.10.2
pkgrel=1
pkgdesc="Project management and code hosting application"
arch=('x86_64')
url="https://gitlab.com/gitlab-org/gitlab-ee"
license=('MIT')
conflicts=("${_pkgname}")
provides=("${_pkgname}")
options=(!buildflags)
depends=('ruby2.5' 'ruby2.5-bundler' 'git' 'gitlab-workhorse' 'gitlab-gitaly' 'openssh' 'redis' 'libxslt' 'icu' 're2' 'http-parser' 'nodejs')
makedepends=('cmake' 'postgresql' 'mariadb' 'yarn' 'go' 'nodejs')
optdepends=(
    'postgresql: PostgreSQL database backend'
    'mysql: MySQL database backend'
    'python2-docutils: reStructuredText markup language support'
    'smtp-server: Mail server to receive e-mail notifications'
)
backup=(
    "etc/webapps/${_pkgname}/application.rb"
    "etc/webapps/${_pkgname}/gitlab.yml"
    "etc/webapps/${_pkgname}/resque.yml"
    "etc/webapps/${_pkgname}/unicorn.rb"
    "etc/logrotate.d/${_pkgname}"
)
source=(
    "${_pkgname}-ee-${pkgver}.tar.gz::https://gitlab.com/gitlab-org/gitlab-ee/-/archive/v${pkgver}-ee/gitlab-ee-v${pkgver}.tar.gz"
    "gitlab-unicorn.service"
    "gitlab-sidekiq.service"
    "gitlab-backup.service"
    "gitlab-mailroom.service"
    "gitlab-backup.timer"
    "gitlab.target"
    "gitlab.tmpfiles.d"
    "gitlab.logrotate"
)
install=gitlab.install
sha512sums=(
    'b1cf8309578b67556e822fc67aa097981ef997b7043cf06209cd4395bb49413e3925fe9070c934867234129ad019e4bf429ba1f8aef71d680801318d27224d25'
    '528ffc56bc93f457c0e40ac1dd10b0b565e757d9962102c531ee1084536d8a17796485b704468f051edceb8aea8f8dfa1df3f5682972d5c2c02571b18c7c0568'
    '28cd84a329566724c493ecaa90f23f1f01cdab3673ee4a3ecb7dfc8e33223b858a2fc23a13c2b4be2fd933b26fdfbb781ae10f1a84b248ba2ab3eefc4419f1f7'
    'c711c31a0a7b5a0b8d997827f0895422df7f2c9d81aafc371fe8e09e25ae1097531df14e4728737b860becef0bf98c34b421ef4411844a571b839b25ca1141fc'
    '66f5c364a6f0f271786da98ea661714a5b6e401a2e5bc88504cd694e6528ab6beef04ee106c28297b2aaf8002792609e87ea5e9307c9e43ab8c8788100844508'
    '2b7ce0cd735c31892678edddb6d61fafefe19b57a5c1c93662be488ed2dd425ab968e00f8b916e1ee4a8c097c36905621fd334c29d5d7609ac307aba069d65c1'
    'bf33b818e4ea671c16f58563997ba5fe0a09090e5c03577ff974d31324d4e9782b85a9bb4f1749b97257ce93400c692de935f003770d52b5994c9cab9aee57c6'
    'e382077187957f66f03678ed6315881b4968d0bd93426c0e23167020fbad596445eb6f21c63c5341e6cfd953f70d968b4b0e99b5bbcd9226c915c2cf8395e543'
    'a1f52d6ca36b32580062dede23ccdde5633238310b28c6c47deb2ce4496f4e5ffea0de2a49bcb1e0e38fc82b66b0cc91a5e86854716c7e848127769b43eb5067'
)

_datadir="/usr/share/webapps/${_pkgname}"
_etcdir="/etc/webapps/${_pkgname}"
_homedir="/var/lib/${_pkgname}"
_logdir="/var/log/${_pkgname}"
_srcdir="gitlab-ee-v${pkgver}"

prepare() {
    # Get first 7 characters from sha1 which has 40 characters in total
    local revision=$(ls -d ${_srcdir}* | rev | cut -c 34-40 | rev)

    cd "${_srcdir}"*

    # GitLab tries to read its revision information from a file.
    echo "${revision}" > REVISION

    export SKIP_STORAGE_VALIDATION='true'

    # Patching config files:
    echo "Patching paths in and username gitlab.yml..."
    sed -e "s|# user: git|user: git|" \
        -e "s|/home/git/gitaly/bin|/usr/bin|" \
        -e "s|/home/git/repositories|${_homedir}/repositories|" \
        -e "s|/home/git/gitlab-satellites|${_homedir}/satellites|" \
        -e "s|# path: /mnt/gitlab|path: ${_homedir}/shared|" \
        -e "s|/home/git/gitlab-shell|/usr/share/webapps/gitlab-shell|" \
        -e "s|tmp/backups|${_homedir}/backups|" \
        -e "s|/home/git/gitlab/tmp/sockets/private/gitaly.socket|${_homedir}/sockets/gitlab-gitaly.socket|" \
        config/gitlab.yml.example > config/gitlab.yml

    echo "Patching paths and timeout in unicorn.rb..."
    sed -e "s|/home/git/gitlab/tmp/.*/|/run/gitlab/|g" \
        -e "s|/var/run/|/run/|g" \
        -e "s|/home/git/gitlab|${_datadir}|g" \
        -e "s|${_datadir}/log/|${_logdir}/|g" \
        config/unicorn.rb.example > config/unicorn.rb

    # We need this one untouched because otherwise assets will fail
    cp config/database.yml.postgresql config/database.yml.postgresql.orig

    echo "Patching username in database.yml.{mysql,postgresql}..."
    sed -i -e "s|username: git|username: gitlab|" config/database.yml.mysql
    sed -i -e "s|username: git|username: gitlab|" config/database.yml.postgresql

    echo "Patching redis connection in resque.yml"
    sed -e "s|production: unix:/var/run/redis/redis.sock|production: redis://localhost:6379|" \
        config/resque.yml.example > config/resque.yml.patched

    echo "Setting up systemd service files ..."
    for service_file in gitlab-sidekiq.service gitlab-unicorn.service gitlab.logrotate gitlab-backup.service gitlab-mailroom.service; do
        sed -i "s|<HOMEDIR>|${_homedir}|g" "${srcdir}/${service_file}"
        sed -i "s|<DATADIR>|${_datadir}|g" "${srcdir}/${service_file}"
        sed -i "s|<LOGDIR>|${_logdir}|g" "${srcdir}/${service_file}"
    done
}

build() {
    cd "${srcdir}/${_srcdir}"*
    echo "Fetching bundled gems..."

    # Gems will be installed into vendor/bundle
    bundle-2.5 install --no-cache --without mysql development test --deployment

    # We'll temporarily stick this in here so we can build the assets
    cp config/database.yml.postgresql.orig config/database.yml
    cp config/resque.yml.example config/resque.yml
    sed -i 's/url.*/nope.sock/g' config/resque.yml

    yarn install --production --pure-lockfile
    bundle-2.5 exec rake gitlab:assets:compile RAILS_ENV=production NODE_ENV=production NODE_OPTIONS="--max_old_space_size=4096"
    bundle-2.5 exec rake gettext:compile RAILS_ENV=production

    # After building assets, clean this up again
    rm config/database.yml config/database.yml.postgresql.orig
    mv config/resque.yml.patched config/resque.yml
}

package() {
cd "${srcdir}/${_srcdir}"*
    depends+=('gitlab-shell')

    install -d "${pkgdir}/usr/share/webapps"

    cp -r "${srcdir}/${_srcdir}"* "${pkgdir}${_datadir}"

    # Remove unneeded directories: node_modules is only needed during build
    rm -r "${pkgdir}${_datadir}/node_modules"

    # https://gitlab.com/gitlab-org/omnibus-gitlab/blob/194cf8f12e51c26980c09de6388bbd08409e1209/config/software/gitlab-rails.rb#L179
    for dir in spec qa rubocop app/assets vendor/assets; do
        rm -r "${pkgdir}${_datadir}/${dir}"
    done

    chown -R root:root "${pkgdir}${_datadir}"
    chmod 755 "${pkgdir}${_datadir}"

    install -dm750 -o 979 -g 979 "${pkgdir}${_homedir}"
    install -dm750 -o 979 -g 979 "${pkgdir}${_homedir}/satellites"
    install -dm750 -o 979 -g 979 "${pkgdir}${_homedir}/shared/"{,artifacts,lfs-objects}
    install -dm750 -o 979 -g 979 "${pkgdir}${_homedir}/builds"
    install -dm700 -o 979 -g 979 "${pkgdir}${_homedir}/uploads"
    install -dm750 -o 979 -g 979 "${pkgdir}${_homedir}/backups"
    install -dm750 -o 979 -g 979 "${pkgdir}${_etcdir}"
    install -dm755 "${pkgdir}/usr/share/doc/${_pkgname}"

    ln -fs /run/gitlab "${pkgdir}${_homedir}/pids"
    ln -fs /run/gitlab "${pkgdir}${_homedir}/sockets"
    ln -fs ${_datadir}/log "${pkgdir}${_homedir}/log"

    rm -rf "${pkgdir}${_datadir}/public/uploads" && ln -fs "${_homedir}/uploads" "${pkgdir}${_datadir}/public/uploads"
    rm -rf "${pkgdir}${_datadir}/builds" && ln -fs "${_homedir}/builds" "${pkgdir}${_datadir}/builds"
    rm -rf "${pkgdir}${_datadir}/tmp" && ln -fs /var/tmp "${pkgdir}${_datadir}/tmp"
    rm -rf "${pkgdir}${_datadir}/log" && ln -fs "${_logdir}" "${pkgdir}${_datadir}/log"

    # Fixes https://bugs.archlinux.org/task/59762
    ln -s "${_datadir}/config/boot.rb" "${pkgdir}"/${_etcdir}/boot.rb

    mv "${pkgdir}${_datadir}/.gitlab_workhorse_secret" "${pkgdir}${_etcdir}/gitlab_workhorse_secret"
    chmod 660 "${pkgdir}${_etcdir}/gitlab_workhorse_secret"
    chown root:979 "${pkgdir}${_etcdir}/gitlab_workhorse_secret"
    ln -fs "${_etcdir}/gitlab_workhorse_secret" "${pkgdir}${_datadir}/.gitlab_workhorse_secret"

    ln -fs /etc/webapps/gitlab-shell/secret "${pkgdir}${_datadir}/.gitlab_shell_secret"

    sed -i "s|require_relative '../lib|require '${_datadir}/lib|" config/application.rb

    # Fix for ruby-2.5 and bundle-2.5
    sed -i "s|bundle|bundle-2.5|g" "${pkgdir}${_datadir}/lib/tasks/gitlab/check.rake"
    grep -rl "bin/env ruby" "${pkgdir}${_datadir}" | xargs sed -i "s|bin/env ruby$|bin/env ruby-2.5|g"
    sed -i \
        -e "s|ruby --version|ruby-2.5 --version|g" \
        -e "s|gem --version|gem-2.5 --version|g" \
        -e "s|bundle --version|bundle-2.5 --version|g" \
        -e "s|rake --version|rake-2.5 --version|g" \
        "${pkgdir}${_datadir}/lib/tasks/gitlab/info.rake"

    # Install config files
    for config_file in application.rb gitlab.yml unicorn.rb resque.yml; do
        mv "config/${config_file}" "${pkgdir}${_etcdir}/"
        [[ -f "${pkgdir}${_datadir}/config/${config_file}" ]] && rm "${pkgdir}${_datadir}/config/${config_file}"
        ln -fs "${_etcdir}/${config_file}" "${pkgdir}${_datadir}/config/"
    done

    # Install database symlink
    ln -fs "${_etcdir}/database.yml" "${pkgdir}${_datadir}/config/database.yml"

    # Install secrets symlink
    ln -fs "${_etcdir}/secrets.yml" "${pkgdir}${_datadir}/config/secrets.yml"

    # Install license and help files
    mv README.md MAINTENANCE.md CONTRIBUTING.md CHANGELOG.md PROCESS.md VERSION config/*.{example,mysql,postgresql} "${pkgdir}/usr/share/doc/${_pkgname}"
    install -Dm644 "LICENSE" "${pkgdir}/usr/share/licenses/${_pkgname}/LICENSE"

    # https://gitlab.com/gitlab-org/gitlab-ce/issues/765
    cp -r "${pkgdir}${_datadir}/doc" "${pkgdir}${_datadir}/public/help"
    find "${pkgdir}${_datadir}/public/help" -name "*.md" -exec rm {} \;
    find "${pkgdir}${_datadir}/public/help/" -depth -type d -empty -exec rmdir {} \;

    chown 979:979 "${pkgdir}${_datadir}/db/schema.rb"

    # Install systemd service files
    for service_file in gitlab-unicorn.service gitlab-sidekiq.service gitlab-backup.service gitlab-backup.timer gitlab.target gitlab-mailroom.service; do
        install -Dm644 "${srcdir}/${service_file}" "${pkgdir}/usr/lib/systemd/system/${service_file}"
    done

    install -Dm644 "${srcdir}/gitlab.tmpfiles.d" "${pkgdir}/usr/lib/tmpfiles.d/gitlab.conf"
    install -Dm644 "${srcdir}/gitlab.logrotate" "${pkgdir}/etc/logrotate.d/gitlab"
}

# vim:set ts=4 sw=4 et:
