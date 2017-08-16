def rocmtestnode(variant, body) {
    def image = 'miopen'
    def cmake_build = { compiler, flags ->
        def cmd = """
            echo \$HSA_ENABLE_SDMA
            mkdir -p $WINEPREFIX
            rm -rf build
            mkdir build
            cd build
            CXX=${compiler} CXXFLAGS='-Werror' cmake ${flags} .. 
            CTEST_PARALLEL_LEVEL=4 dumb-init make -j32 check doc MIOpenDriver
        """
        echo cmd
        sh cmd
    }
    node('rocmtest && fiji') {
        stage("checkout ${variant}") {
            // env.HCC_SERIALIZE_KERNEL=3
            // env.HCC_SERIALIZE_COPY=3
            env.HSA_ENABLE_SDMA=0 
            // env.HSA_ENABLE_INTERRUPT=0
            env.WINEPREFIX="/jenkins/.wine"
            checkout scm
        }
        stage("image ${variant}")
        {
            docker.build("${image}", "--build-arg PREFIX=/usr/local .")
        }
        withDockerContainer(image: image, args: '--device=/dev/kfd') {
            timeout(time: 1, unit: 'HOURS') {
                body(cmake_build)
            }
        }
    }
}

def rocmtest(m) {
    def builders = [:]
    for(e in m) {
        def label = e.key;
        def action = e.value;
        builders[label] = {
            rocmtestnode(label, action)
        }
    }
    parallel builders
}

rocmtest opencl: { cmake_build ->
    stage('Clang Tidy') {
        sh '''
            rm -rf build
            mkdir build
            cd build
            CXX='clang++-3.8' cmake -DBUILD_DEV=On .. 
            make tidy
        '''
    }
    stage('Clang Format') {
        sh '''
            find . -iname \'*.h\' \
                -o -iname \'*.hpp\' \
                -o -iname \'*.cpp\' \
                -o -iname \'*.h.in\' \
                -o -iname \'*.hpp.in\' \
                -o -iname \'*.cpp.in\' \
                -o -iname \'*.cl\' \
            | grep -v 'build/' \
            | xargs -n 1 -P 1 -I{} -t sh -c \'clang-format-3.8 -style=file {} | diff - {}\'
        '''
    }
    stage('Clang Debug') {
        cmake_build('clang++-3.8', '-DBUILD_DEV=On -DCMAKE_BUILD_TYPE=debug')
    }
    stage('Clang Release') {
        cmake_build('clang++-3.8', '-DBUILD_DEV=On -DMIOPEN_TEST_ALL=On -DCMAKE_BUILD_TYPE=release')
    }
    stage('GCC Debug') {
        cmake_build('g++-5', '-DBUILD_DEV=On -DCMAKE_BUILD_TYPE=debug')
    }
    stage('GCC Release') {
        cmake_build('g++-5', '-DBUILD_DEV=On -DMIOPEN_TEST_ALL=On -DCMAKE_BUILD_TYPE=release')
    }
} hip: { cmake_build ->
    stage('Hip Tidy') {
        sh '''
            rm -rf build
            mkdir build
            cd build
            CXX='hcc' cmake -DBUILD_DEV=On .. 
            make tidy
        '''
    }
    // stage('Hip Debug') {
    //     cmake_build('hcc', '-DBUILD_DEV=On -DCMAKE_BUILD_TYPE=debug')
    // }
    stage('Hip Release') {
        cmake_build('hcc', '-DBUILD_DEV=On -DMIOPEN_TEST_ALL=On -DCMAKE_BUILD_TYPE=release')
    }
} windows: { cmake_build ->
    stage('Windows Release') {
        cmake_build('x86_64-w64-mingw32-g++', '-DBUILD_DEV=On -DCMAKE_TOOLCHAIN_FILE=/usr/local/x86_64-w64-mingw32/cmake/toolchain.cmake -DCMAKE_BUILD_TYPE=release')
    }
}
