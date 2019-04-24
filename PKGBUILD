# Gitlab Enterprise Edition v11.10.0
# Maintainer: Steven Cook <scook@velocity.org>

# Note: This package conflicts with Gitlab, which provides the CE (community edition).
# DO NOT upload this package to the Arch Linux AUR; this package is solely for use by Velocity only.
# This package must be manually upgraded for now. In the future, we may provide a custom Arch Linux repository.

_pkgname=gitlab

pkgname=velocity-${_pkgname}-ee
pkgver=11.10.0
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
md5sums=('8f7706b1048b5ebda058dc958252324b'
         'c37b66a348ea827e31e63dd42618c8b5'
         '801952180f695a5f65cca763de2ea3ca'
         '504238dafa7d705a2c3ae856af578e4c'
         '382962bb70d221eef5cc4ccbc8ad1b91'
         'e75d51f06374b2f89cdf5ff6f06cfb45'
         '24438d49a7d7bf5f66946bd1f12bf116'
         '041ac3740f28458f4cbdcfcafbd4e569'
         '92a05461d4b026f648eebd4e3d4704f8')

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
    bundle-2.5 install --no-cache --deployment --without development test aws kerberos

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

    install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}"
    install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/satellites"
    install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/shared/"{,artifacts,lfs-objects}
    install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/builds"
    install -dm700 -o 105 -g 105 "${pkgdir}${_homedir}/uploads"
    install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/backups"
    install -dm750 -o 105 -g 105 "${pkgdir}${_etcdir}"
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
    chown root:105 "${pkgdir}${_etcdir}/gitlab_workhorse_secret"
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

    chown 105:105 "${pkgdir}${_datadir}/db/schema.rb"

    # Install systemd service files
    for service_file in gitlab-unicorn.service gitlab-sidekiq.service gitlab-backup.service gitlab-backup.timer gitlab.target gitlab-mailroom.service; do
        install -Dm644 "${srcdir}/${service_file}" "${pkgdir}/usr/lib/systemd/system/${service_file}"
    done

    install -Dm644 "${srcdir}/gitlab.tmpfiles.d" "${pkgdir}/usr/lib/tmpfiles.d/gitlab.conf"
    install -Dm644 "${srcdir}/gitlab.logrotate" "${pkgdir}/etc/logrotate.d/gitlab"
}

# vim:set ts=4 sw=4 et:
