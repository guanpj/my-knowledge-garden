C 引用
```C++
#include <iostream>
using namespace std;
 
// 函数声明
void swap(int& x, int& y);
void swap1(int* x, int* y);
void swap2(int x, int y);
void swap3(int* x, int* y);
 
int main ()
{
  // 局部变量声明
  int a = 100;
  int b = 200;
 
  cout << "交换前，&a 的值：" << &a << endl;
  cout << "交换前，&b 的值：" << &b << endl;

  cout << "交换前，a 的值：" << a << endl;
  cout << "交换前，b 的值：" << b << endl;
 
  /* 调用函数来交换值 */
  //swap(a, b);
  //swap1(&a, &b);
  //swap2(a, b);
  swap3(&a, &b);
 
  cout << "交换后，a 的值：" << a << endl;
  cout << "交换后，b 的值：" << b << endl;

  cout << "交换后，&a 的值：" << &a << endl;
  cout << "交换后，&b 的值：" << &b << endl;
 
  return 0;
}
 
// 函数定义
void swap(int& x, int& y)
{
  cout << "swap" << endl;
  int temp;
  temp = x;
  x = y;
  y = temp;
}

void swap1(int* x, int* y)
{
  cout << "swap1" << endl;
  int* temp;
  temp = x;
  x = y;
  y = temp;
}

void swap2(int x, int y)
{
  cout << "swap2" << endl;
  int temp;
  temp = x;
  x = y; 
  y = temp;
}

void swap3(int* x, int* y)
{
  cout << "swap3" << endl;
  int temp;
  temp = *x;
  *x = *y;
  *y = temp;
}
```