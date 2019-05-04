---
title: C++ IO操作
date: 2019-03-24
categories:
- C++
---
<!-- toc -->

#### 字符操作

```
void main(){
	char* fileName = "d://dest.txt";

	// 创建输出流
	ofstream fout(fileName);

	// 创建成功
	if (!fout.bad()){
		fout << "hello" << endl;
		fout << "world!" << endl;
		fout.close();
	}

	// 创建输出流
	ifstream fin(fileName);
	if (!fin.bad()){
		char ch;
		while (fin.get(ch))
		{
			cout << ch;
		}
		fin.close();
	}

	system("pause");
	}
```

#### 二进制数据操作

```
void main(){
	char* src = "d://src.png";
	char* dest = "d://dest.png";

	// 参数2类似于"wb", "rb"
	ifstream fin(src, ios::binary);
	ofstream fout(dest, ios::binary);

	if (fin.bad() || fout.bad()) return;

	// 没有到达文件末尾
	while (!fin.eof()){
		char buff[1014] = { 0 };
		fin.read(buff, 1024);

		fout.write(buff, fin.gcount());
	}

	fout.close();
	fin.close();

	system("pause");
	}
```

#### 对象操作

```
class Person{
public:
	Person(){}

	Person(char* name){
		this->name = name;
	}

	void print(){
		cout << name << endl;
	}
private:
	char* name;
};

void main(){
	Person p("Jack");

	char* dest = "d://person.data";

	ofstream fout(dest, ios::binary);

	fout.write((char*)&p, sizeof(Person));
	fout.close();

	// 这种初始化只是防止成为野指针，并没有有效的内存空间
	// Person  *p1 = NULL;

	Person  p2;
	ifstream fin(dest, ios::binary);

	// 传p2的首地址，为p2内存赋值
	// 传指针，则指针必须是已经初始化过，有指向的
	// 对象其实是读到p2的内存中
	fin.read((char*)&p2, sizeof(Person));
	p2.print();
	fin.close();

	system("pause");
}
```
