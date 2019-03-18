---
title: IO操作
date: 2018-08-26
tags:
- 基础
categories:
- C
---
<!-- toc -->

#### 获取文件大小

```
void main(){
	char* path = "C:\\Users\\dell\\Desktop\\文件加密.png";
	FILE* file = fopen(path, "rb");

	// 重新定位指针到文件末尾
	fseek(file, 0, SEEK_END);

	// 返回当前的文件指针，相对于文件开头的偏移量
	int length = ftell(file);

	printf("%d\n",length);

	getchar();
}
```
<!-- more -->
#### 字符读取

```
void main(){
	char* path = "C:\\Users\\dell\\Desktop\\friends.txt";

	// 可读方式读取
	FILE* file = fopen(path, "r");

	if (file == NULL){
		printf("读取失败...");
		return;
	}

	// 以字符串形式读取内容(中文情况下，如果缓存太小则会有乱码)
	char buffer[50];
	while (fgets(buffer, 50, file)){
		printf("%s\n", buffer);
	}

	fclose(file);
	getchar();
}
```

#### 字符串写入文件

```
void main(){
	char* path = "C:\\Users\\dell\\Desktop\\friends_new.txt";

	FILE* file = fopen(path, "w");

	fputs("hello world !",file);

	fclose(file);
	getchar();
}
```

#### 二进制读取

C 读取文本文件和二进制文件的差别仅仅体现在回车换行符上：

写文本时，每遇到一个'\n'，会将其转换为'\r\n';

读文件时，每遇到一个'\r\n'，会将其转换为'\n';

采用二进制进行读写时，则不会进行任何转换;

```
void main(){
	char* path = "C:\\Users\\dell\\Desktop\\文件加密.png";
	char* write_path = "C:\\Users\\dell\\Desktop\\文件加密_new.png";

	FILE* file = fopen(path, "rb");
	FILE* wfile = fopen(write_path, "wb");


	if (file == NULL){
		printf("文件不存在...");
		return;
	}

	int buffer[50];
	// 当读取结束时返回的是0
	int len = 0;
	// 读取元素的最小单位为int(4个字节)，每次读取50个
	while ((len = fread(buffer, sizeof(int), 50, file))!=0){
		fwrite(buffer, sizeof(int), len, wfile);
	}

	fclose(file);
	fclose(wfile);
	getchar();
}
```

#### 文件加密

可以采用二进制数据异或运算来实现文件的简单加密；原理：

提供一个用来异或运算的key，与进行加密的二进制数据进行异或运算，同值取0，异值取1。
由于key可以是任意的，所以在不知道key的情况下是无法解密的；

解密只需要对key再进行一次异或即可完成；

```
void main(){
	char* path = "C:\\Users\\dell\\Desktop\\文件加密.png";
	char* crypt_path = "C:\\Users\\dell\\Desktop\\文件加密_crypt.png";
	char* decrypt_path = "C:\\Users\\dell\\Desktop\\文件加密_decrypt.png";

  //crypt(path, crypt_path);

  //crypt(crypt_path, decrypt_path);

  //byte_crypt(path, crypt_path, "password");

  //byte_crypt(crypt_path, decrypt_path, "password");

	getchar();
}
```

##### 单数字字符加解密

```
void crypt(char* filePath, char* decryptPath){
	FILE* file = fopen(filePath, "r");
	FILE* wfile = fopen(decryptPath, "w");

	// 一次读取一个字符
	int ch;
	while ((ch = fgetc(file)) != EOF){
		// 此处异或的key为9
		fputc(ch ^ 9, wfile);
	}
	fclose(file);
	fclose(wfile);
}
```

#### 字符串加解密二进制数据

在C中，二进制与字符文件处理的区别只是在回车换行符上的自动转换，所以此处可以直接采用字符的方式处理二进制文件；

采用字符串作为key进行加解密时，可以通过循环与key的每一个字符进行异或来实现加解密；

```
void byte_crypt(char* filePath, char* decryptPath,char* key){
	// 以二进制形式打开
	FILE* file = fopen(filePath, "rb");
	FILE* wfile = fopen(decryptPath, "wb");

	// 一次读取一个字符
	int ch;
	int i = 0;
	int pass_len = strlen(key);
	while ((ch = fgetc(file)) != EOF){
		// 每次异或key中的一个字符，循环进行异或
		fputc(ch ^ key[i%pass_len], wfile);
		i++;
	}
	fclose(file);
	fclose(wfile);
}
```
