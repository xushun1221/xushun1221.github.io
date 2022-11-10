# CentOS7安装Cmake


1. `sudo yum install -y gcc gcc-c++ make automake`
2. `sudo yum install openssl-devel`
3. `wget https://github.com/Kitware/CMake/releases/download/v3.24.3/cmake-3.24.3.tar.gz`
4. `tar zxvf cmake-3.24.3.tar.gz`
5. `cd cmake-3.24.3`
6. `./configure`
7. `make`
8. `sudo make install`
9. `cmake --version`
