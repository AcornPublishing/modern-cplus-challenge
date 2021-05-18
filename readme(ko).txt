소스 코드를 빌드하는 법

Mac OS X / Xcode
----------------------------

사전 준비 사항:
--------------

OpenSSL
1. https://www.openssl.org/ 에서 openssl을 다운로드 한다.
2. 다음 명령어들을 입력해 openssl을 빌드하고 설치한다.

   ./Configure darwin64-x86_64-cc shared enable-ec_nistp_64_gcc_128 no-ssl2 no-ssl3 no-comp --openssldir=/usr/local/ssl/macos-x86_64
   
   make depend
   
   sudo make install

또는 Homebrew를 이용해 brew install openssl으로 설치할 수 있다.

Boost (선택사항)
1. https://brew.sh/에서 Homebrew를 설치한다.
2. 다음 명령어를 입력하면 Boost가 자동으로 다운로드 및 설치된다.
   
   brew install boost
   
3. Boost라이브러리는 설치가 완료된 후 /usr/local/Cellar/boost/1.70.0 (설치한 버전에 따라 차이가 있을 수 있다)에서 사용 할 수 있다.


소스 코드:
--------------

1. 다음 명령어들을 입력해 CMake 스크립트를 빌드한다.
   
   cd build
   
   cmake -G Xcode .. -DOPENSSL_ROOT_DIR=/usr/local/bin -DOPENSSL_INCLUDE_DIR=/usr/local/include/ -DBUILD_TESTING=OFF -DBUILD_CURL_EXE=OFF -DUSE_MANUAL=OFF 
   
   만약 표준 라이브러리 대신 Boost.Filesystem이나 Boost.Optional을 이용하기 위해서는 추가적으로 다음 패러미터들을 설정해 주어야 한다.
     BOOST_FILESYSTEM  ON으로 설정한다
     BOOST_OPTIONAL    ON으로 설정한다
     BOOST_INCLUDE_DIR Boost 헤더 디렉토리의 경로
     BOOST_LIB_DIR     Boost 라이브러리 디렉토리의 경로
   
   cmake -G Xcode .. -DOPENSSL_ROOT_DIR=/usr/local/bin -DOPENSSL_INCLUDE_DIR=/usr/local/include/ -DBUILD_TESTING=OFF -DBUILD_CURL_EXE=OFF -DUSE_MANUAL=OFF -DBOOST_FILESYSTEM=ON -DBOOST_OPTIONAL=ON -DBOOST_INCLUDE_DIR=/usr/local/Cellar/boost/1.70.0/include -DBOOST_LIB_DIR=/usr/local/Cellar/boost/1.70.0/lib

OpenSSL을 Homebrew를 통해 설치한 경우 아래처럼 OpenSSL의 경로를 변경해야 한다.
   cmake -G Xcode .. -DOPENSSL_ROOT_DIR=/usr/local/opt/openssl -DOPENSSL_INCLUDE_DIR=/usr/local/opt/openssl/include/ -DBUILD_TESTING=OFF -DBUILD_CURL_EXE=OFF -DUSE_MANUAL=OFF -DBOOST_FILESYSTEM=ON -DBOOST_OPTIONAL=ON -DBOOST_INCLUDE_DIR=/usr/local/Cellar/boost/1.70.0/include -DBOOST_LIB_DIR=/usr/local/Cellar/boost/1.70.0/lib
   
2. build/cppchallenger.xcodeproj 에서 Xcode 프로젝트를 연다.



Windows / Visual Studio 2017
----------------------------

1. 다음 명령어들을 입력해 CMake 스크립트를 빌드한다.

   cd build
   
   cmake -G "Visual Studio 15 2017" .. -DCMAKE_USE_WINSSL=ON -DCURL_WINDOWS_SSPI=ON -DCURL_LIBRARY=libcurl -DCURL_INCLUDE_DIR=..\libs\curl\include -DBUILD_TESTING=OFF -DBUILD_CURL_EXE=OFF -DUSE_MANUAL=OFF
   
2. build/cppchallenger.sln 에서 Visual Studio 솔루션을 연다.
