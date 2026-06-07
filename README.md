# venom

cobra-style network stress testing tool. terminal gui. ascii snake. works on linux, termux, iSH.

## install

### linux
```bash
git clone https://github.com/raphaelurbanek257-hub/venom
cd venom
gcc -O3 venom.c -o venom -lpthread -lncurses -Wall
sudo ./venom
