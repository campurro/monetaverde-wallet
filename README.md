# monetaverde-wallet

Monetaverde is a cryptonight based cryptocurrency (ticker : MCN)

This is the monetaverde gui wallet source code and binary release for MacOs, Windows and linux

[Main website](https://mcn.green)

[BitcoinTalk main announcement thread](https://bitcointalk.org/index.php?topic=5069658)

[monetaverde-wallet gui (source and binaries)](https://github.com/mcnproject/monetaverde-wallet)


Compilation tips :

Dependencies needed : boost >= 1.58, CMake >= 3.1, GCC >=4.7.3, Qt >=5.0

Currently built release is compiled with boost 1.60 and Qt 5.10

* Clone the repository :
```
$ git clone https://github.com/mcnproject/monetaverde-wallet.git
```

* Update core submodule :
```
$ cd monetaverde-wallet
$ git submodule update --init
$ git submodule foreach git pull origin master
```
* Build :
```
$ mkdir build
$ cd build
$ cmake -DSTATIC=ON -DCMAKE_BUILD_TYPE=RELEASE ..
$ make
```
* Trouble :
  * Cmake complain not finding boost libs : specify your boost installation path :
  ```
    -DBOOST_ROOT="Your_boost_root_path"
  ```
  * Cmake complain not finding Qt libs : specify your Qt installation path :
  ```
    -DBOOST_ROOT="Your_Qt_root_path"
  ```
  * Your Processor is quiet old and doesn't support aes or avx :
  ```
    $ PORTABLE=1 make
  ```
