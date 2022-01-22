###### 一、指针初识

```c
int i = 2;
int *ptr_i = &i;

//i的地址：0x7ffee438a468
printf("%p\n", &i);
//ptr_i为i的地址：0x7ffee438a468
printf("%p\n", ptr_i);
//指针自己的地址：0x7ffee4d98460
printf("%p\n", &ptr_i);
//该指针指向的地址对应的值：2
printf("%d\n", *ptr_i);

//将指针指向的地址赋值给另外一个指针：0x7ffeee6e7468
//void * 类型表示未确定类型的指针。C、C++ 规定 void * 类型可以通过类型转换强制转换为任何其它类型的指针
void *ptr2 = ptr_i;
printf("%p\n", ptr2);
```

###### 二、内存管理

1. **malloc(int num); ** //在堆区分配一块指定大小的内存空间，用来存放数据
2. **realloc(void \*address, int newsize);**//该函数重新分配内存，把内存扩展到newsize
3. **calloc(int num, int size)**;//在内存中动态地分配 num 个长度为 size 的连续空间，并将每一个字节都初始化为 0
4. **free(void \*address);**//释放 address 所指向的内存块,释放的是动态分配的内存空间

###### 三、值传递和指针传递

```c
void swapValue(int x, int y) {
    int temp = x;
    x = y;
    y = temp;
}
//交换的是指针指向地址存的值，并非交换地址
void swapPointer(int* x, int* y) {
    int temp = *x;
    *x = *y;
    *y = temp;
}

int main() {
    printf("Hello, World2!\n");
    int i = 1;
    int j = 2;
    swapValue(i, j);
    //i: 1, j: 2
    printf("i: %d, j: %d\n", i, j);

    //交换前地址：i: 0x7ffee012d46c, j: 0x7ffee012d468
    printf("交换前地址：i: %p, j: %p\n", &i, &j);
    swapPointer(&i, &j);
    //交换后地址：i: 0x7ffee012d46c, j: 0x7ffee012d468
    printf("交换后地址：i: %p, j: %p\n", &i, &j);
    //i: 2, j: 1
    printf("i: %d, j: %d", i, j);
}

```

