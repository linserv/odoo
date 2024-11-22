pipeline {
    agent any
    environment {
        ODOO_VERSION = "${env.BRANCH_NAME}" // Dynamically set from the branch name
        SWIFT_CONTAINER = 'odoo'
        ENTERPRISE_SWIFT_CONTAINER = 'enterprise'
    }
    stages {
        stage('Setup Environment') {
            steps {
                script {
                    echo "Branch: ${BRANCH_NAME}"
                    
                    // Ensure the branch name is valid for an Odoo version
                    if (!(BRANCH_NAME == 'master' || BRANCH_NAME ==~ /^\d+\.\d+$/)) {
                        error "Invalid branch name '${BRANCH_NAME}'. Expected 'master' or a version like '17.0'."
                    }
                    
                    echo "Building Odoo version: ${ODOO_VERSION}"
                }
            }
        }
        stage('Build Odoo Debian Package') {
            steps {
                script {
                    sh """
                    #!/bin/bash
                    set -e

                    ODOO_VERSION="${env.ODOO_VERSION}"
                    TODAY=\$(date +%Y%m%d)
                    PACKAGE_NAME="odoo"
                    PACKAGE_VERSION="\${ODOO_VERSION}.\${TODAY}"

                    BUILD_DIR="\${PACKAGE_NAME}_\${PACKAGE_VERSION}_all"

                    echo "Preparing build directory: \${BUILD_DIR}"
                    rm -rf "\${BUILD_DIR}"
                    mkdir -p "\${BUILD_DIR}/usr/bin" \
                             "\${BUILD_DIR}/usr/lib/python3/dist-packages/\${PACKAGE_NAME}" \
                             "\${BUILD_DIR}/usr/lib/\${PACKAGE_NAME}" \
                             "\${BUILD_DIR}/etc/\${PACKAGE_NAME}" \
                             "\${BUILD_DIR}/var/lib/odoo" \
                             "\${BUILD_DIR}/usr/share/\${PACKAGE_NAME}" \
                             "\${BUILD_DIR}/DEBIAN"

                    echo "Cloning Odoo source code for version \${ODOO_VERSION}"
                    git clone --depth=1 --branch="\${ODOO_VERSION}" https://github.com/linserv/odoo.git
                    rm -rf odoo/.git
                    rm -f odoo/.gitignore

                    echo "Copying files to build directory"
                    rsync -av odoo/odoo/ "\${BUILD_DIR}/usr/lib/python3/dist-packages/\${PACKAGE_NAME}"
                    rsync -av odoo/addons/ "\${BUILD_DIR}/usr/lib/\${PACKAGE_NAME}"

                    cp odoo/odoo-bin "\${BUILD_DIR}/usr/bin/\${PACKAGE_NAME}"
                    cp odoo/requirements.txt "\${BUILD_DIR}/usr/share/\${PACKAGE_NAME}/requirements.txt"

                    echo "Generating control file"
                    cat <<EOF > "\${BUILD_DIR}/DEBIAN/control"
                    Package: \${PACKAGE_NAME}
                    Version: \${PACKAGE_VERSION}
                    Section: base
                    Priority: optional
                    Architecture: all
                    Depends: python3, adduser
                    Maintainer: Linserv Aktiebolag <info@linserv.se>
                    Description: Odoo \${ODOO_VERSION} nightly build
                    A nightly build of Odoo version \${ODOO_VERSION}.
                    EOF

                    echo "Building Debian package"
                    dpkg-deb --build "\${BUILD_DIR}"

                    echo "Moving package"
                    mv "\${BUILD_DIR}.deb" "odoo_\${PACKAGE_VERSION}_all.deb"
                    rm -rf "\${BUILD_DIR}"
                    rm -rf odoo
                    """
                }
            }
        }
        stage('Build Enterprise Tarball') {
            steps {
                script {
                    sh """
                    #!/bin/bash
                    set -e

                    ODOO_VERSION="${env.ODOO_VERSION}"
                    TODAY=\$(date +%Y%m%d)
                    ENTERPRISE_TAR="enterprise_\${ODOO_VERSION}.\${TODAY}.tar.gz"

                    echo "Cloning Odoo Enterprise source code for version \${ODOO_VERSION}"
                    git clone --depth=1 --branch="\${ODOO_VERSION}" https://github.com/linserv/enterprise.git
                    rm -rf enterprise/.git
                    rm -f enterprise/.gitignore

                    echo "Creating tarball"
                    tar -czf \${ENTERPRISE_TAR} -C enterprise .

                    echo "Cleaning up"
                    rm -rf enterprise
                    """
                }
            }
        }
        stage('Upload to Swift') {
            steps {
                script {
                    sh """
                    #!/bin/bash
                    set -e
                    source /var/lib/jenkins/swift_env

                    ODOO_VERSION="${env.ODOO_VERSION}"
                    TODAY=\$(date +%Y%m%d)
                    DEBIAN_PACKAGE="odoo_\${ODOO_VERSION}.\${TODAY}_all.deb"
                    ENTERPRISE_TAR="enterprise_\${ODOO_VERSION}.\${TODAY}.tar.gz"

                    echo "Uploading Debian package to Swift"
                    swift upload ${SWIFT_CONTAINER} "\${DEBIAN_PACKAGE}"

                    echo "Uploading Enterprise tarball to Swift"
                    swift upload ${ENTERPRISE_SWIFT_CONTAINER} "\${ENTERPRISE_TAR}"

                    echo "Cleaning up local files"
                    rm -f "\${DEBIAN_PACKAGE}" "\${ENTERPRISE_TAR}"
                    """
                }
            }
        }
    }
    post {
        always {
            script {
                echo "Pipeline finished for branch ${BRANCH_NAME}"
            }
        }
        failure {
            script {
                echo "Pipeline failed for branch ${BRANCH_NAME}"
            }
        }
    }
}

