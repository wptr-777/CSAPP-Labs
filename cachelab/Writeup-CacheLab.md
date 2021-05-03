# Cache Lab



## Part A

> 编写csim.c,模拟cache的运行,需要实现和csim-ref一样的功能,最后跑test-csim即可评分

```shell
Usage: ./csim-ref [-hv] -s <num> -E <num> -b <num> -t <file>
Options:
  -h         Print this help message.
  -v         Optional verbose flag.
  -s <num>   Number of set index bits.
  -E <num>   Number of lines per set.
  -b <num>   Number of block offset bits.
  -t <file>  Trace file.

Examples:
  linux>  ./csim-ref -s 4 -E 1 -b 4 -t traces/yi.trace
  linux>  ./csim-ref -v -s 8 -E 2 -b 4 -t traces/yi.trace
```

用getopt获取命令行参数

fscanf读取文件内容

根据LRU规则操作cache

先定义一个结构体

```c
typedef struct 
{
	int valid_bit, tag, stamp;//cold miss
}cache_line;
cache_line **cache = NULL;
```

根据输入数据进行处理

```c
while(-1 != (opt = (getopt(argc, argv, "hvs:E:b:t:"))))
	{
		switch(opt)
		{
			case 'h':help_mode = 1;
					 break;
			case 'v':verbose_mode = 1;
					 break;
			case 's':s = atoi(optarg);
					 break;
			case 'E':E = atoi(optarg);
					 break;
			case 'b':b = atoi(optarg);
					 break;
			case 't':strcpy(filename, optarg);
					 break;
		}
	}
	if(help_mode == 1)
	{
		system("cat help_info");
		exit(0);
	}
	FILE* fp = fopen(filename,"r");
	if(fp == NULL)
	{
		fprintf(stderr,"The File is wrong!\n");
		exit(-1);
	}
	S = (1 << s); // S equals to 2^s
	cache = (cache_line**)malloc(sizeof(cache_line*) * S);
	for(int i = 0; i < S; i++)
		cache[i] = (cache_line*)malloc(sizeof(cache_line) * E);
	for(int i = 0; i < S; i++)
		for(int j = 0; j < E; j++)
		{
			cache[i][j].valid_bit = 0;
			cache[i][j].tag = cache[i][j].stamp = -1;
		}//initialization
	while(fgets(buffer,1000,fp))
	{
		sscanf(buffer," %c %xu,%d", &type, &address, &temp);//hexdecimal
		switch(type)
		{
			case 'L':update(address);
					 break;
			case 'M':update(address);
			case 'S':update(address);
					 break;
		}
		update_time();
	}
```

### csim.c

```c
#include "cachelab.h"
#include<stdlib.h>
#include<unistd.h>
#include<stdio.h>
#include<limits.h>
#include<getopt.h>
#include<string.h> 
int help_mode, verbose_mode, s, E, b, S,number_hits, number_miss, number_eviction;
//S is the number of sets, E is the associativity, b is number of block bits
char filename[1000];//The file name
char buffer[1000];//The buffer of input
typedef struct 
{
	int valid_bit, tag, stamp;//cold miss
}cache_line;
cache_line **cache = NULL;
void update(unsigned int address)
{
	int max_stamp = INT_MIN, max_stamp_id = -1;
	int  t_address, s_address;// The t value and s value of address
	s_address = (address >> b) & ((-1U) >> (32 - s));//use bit manipulation to get s_address, -1U equals to INT_MAX
	t_address = address >> (s + b);
	for(int i = 0; i < E; i++)//check whether there is a hit
		if(cache[s_address][i].tag == t_address)//which means a hit
		{
			//is_placed = 1;
			cache[s_address][i].stamp = 0;//restart the time stamp control unit
			number_hits++;
			//printf("hit\n");
			return ;//just return now
		}
	//to check whether is an empty line
	for(int i = 0; i < E; i++)
		if(cache[s_address][i].valid_bit == 0)
		{
			cache[s_address][i].valid_bit = 1;
			cache[s_address][i].tag = t_address;
			cache[s_address][i].stamp = 0;
			number_miss++;//compulsory miss
			return;
		}
	//If there is not any empty line, then an eviction will occur
	number_eviction++;
	number_miss++;
	for(int i = 0; i < E; i++)
		if(cache[s_address][i].stamp > max_stamp)
		{
			max_stamp = cache[s_address][i].stamp;
			max_stamp_id = i;
		}
	cache[s_address][max_stamp_id].tag = t_address;
	cache[s_address][max_stamp_id].stamp = 0;
	return;
}

void update_time(void)
{
	for(int i = 0; i < S; i++)
		for(int j = 0; j < E; j++)
			if(cache[i][j].valid_bit == 1)
				cache[i][j].stamp++;
}

int main(int argc,char* argv[])
{
	int opt, temp;
	char type;
	unsigned int address;
	number_hits = number_miss = number_eviction = 0;
	while(-1 != (opt = (getopt(argc, argv, "hvs:E:b:t:"))))
	{
		switch(opt)
		{
			case 'h':help_mode = 1;
					 break;
			case 'v':verbose_mode = 1;
					 break;
			case 's':s = atoi(optarg);
					 break;
			case 'E':E = atoi(optarg);
					 break;
			case 'b':b = atoi(optarg);
					 break;
			case 't':strcpy(filename, optarg);
					 break;
		}
	}
	if(help_mode == 1)
	{
		system("cat help_info");
		exit(0);
	}
	FILE* fp = fopen(filename,"r");
	if(fp == NULL)
	{
		fprintf(stderr,"The File is wrong!\n");
		exit(-1);
	}
	S = (1 << s); // S equals to 2^s
	cache = (cache_line**)malloc(sizeof(cache_line*) * S);
	for(int i = 0; i < S; i++)
		cache[i] = (cache_line*)malloc(sizeof(cache_line) * E);
	for(int i = 0; i < S; i++)
		for(int j = 0; j < E; j++)
		{
			cache[i][j].valid_bit = 0;
			cache[i][j].tag = cache[i][j].stamp = -1;
		}//initialization
	while(fgets(buffer,1000,fp))
	{
		sscanf(buffer," %c %xu,%d", &type, &address, &temp);//hexdecimal
		switch(type)
		{
			case 'L':update(address);
					 break;
			case 'M':update(address);
			case 'S':update(address);
					 break;
		}
		update_time();
	}
	for(int i = 0; i < S; i++)
		free(cache[i]);
	free(cache);
	fclose(fp);
	printSummary(number_hits, number_miss, number_eviction);
	return 0;
}
```

### 运行结果

```shell
                        Your simulator     Reference simulator
Points (s,E,b)    Hits  Misses  Evicts    Hits  Misses  Evicts
     3 (1,1,1)       9       8       6       9       8       6  traces/yi2.trace
     3 (4,2,4)       4       5       2       4       5       2  traces/yi.trace
     3 (2,1,4)       2       3       1       2       3       1  traces/dave.trace
     3 (2,1,3)     167      71      67     167      71      67  traces/trans.trace
     3 (2,2,3)     201      37      29     201      37      29  traces/trans.trace
     3 (2,4,3)     212      26      10     212      26      10  traces/trans.trace
     3 (5,1,5)     231       7       0     231       7       0  traces/trans.trace
     6 (5,1,5)  265189   21775   21743  265189   21775   21743  traces/long.trace
    27

TEST_CSIM_RESULTS=27
```

## Part B

> 优化矩阵转置函数，使得cache miss尽可能少,不超过12个临时变量

### 32 x 32

直接进行转置会造成miss数量过多

32位字节的数据，一个int4字节，每行/列有8个int，于是我们将其分块为8x8，进行处理

但是存在对角线上元素的问题，对其进行特殊处理即可，反正有12个临时变量能用

4个用来循环，剩下8个用来整行读取，而非直接赋值

```c
B[n][m] = A[m][n];
```

先将一整行矩阵读取，然后再写到B上

```c
if(M == 32)
	{
		int i, j, k, v1, v2, v3, v4, v5, v6, v7, v8;
		for (i = 0; i < 32; i += 8)
			for(j = 0; j < 32; j += 8)
				for(k = i; k < (i + 8); ++k)
				{
					v1 = A[k][j];
					v2 = A[k][j+1];
					v3 = A[k][j+2];
					v4 = A[k][j+3];
					v5 = A[k][j+4];
					v6 = A[k][j+5];
					v7 = A[k][j+6];			
					v8 = A[k][j+7];
					B[j][k] = v1;
					B[j+1][k] = v2;
					B[j+2][k] = v3;
					B[j+3][k] = v4;
					B[j+4][k] = v5;
					B[j+5][k] = v6;
					B[j+6][k] = v7;
					B[j+7][k] = v8;
				}
	}
```

### 64 x 64

这个使用 4 x 4分块，在64的规模下，8x8 miss数太多了，不过这样分块也没法满分，有空再优化一下

```c
	else if(M == 64)
	{
		int i, j, k, v1, v2, v3, v4;
		for (i = 0; i < M; i += 4)
			for(j = 0; j < M; j += 4)
				for(k = i; k < (i + 4); ++k)
				{
					v1 = A[k][j];
					v2 = A[k][j+1];
					v3 = A[k][j+2];
					v4 = A[k][j+3];
					B[j][k] = v1;
					B[j+1][k] = v2;
					B[j+2][k] = v3;
					B[j+3][k] = v4;
				}
	}
```

### 61 x 67

不规则矩阵，最后还是用8分块来做了，处理一下对角线，和32x32的做法差不多

```c
	else if(M == 61)
	{
		int i, j, v1, v2, v3, v4, v5, v6, v7, v8;
		int n = N / 8 * 8;
		int m = M / 8 * 8;
		for (j = 0; j < m; j += 8)
			for (i = 0; i < n; ++i)
			{
				v1 = A[i][j];
				v2 = A[i][j+1];
				v3 = A[i][j+2];
				v4 = A[i][j+3];
				v5 = A[i][j+4];
				v6 = A[i][j+5];
				v7 = A[i][j+6];
				v8 = A[i][j+7];
				
				B[j][i] = v1;
				B[j+1][i] = v2;
				B[j+2][i] = v3;
				B[j+3][i] = v4;
				B[j+4][i] = v5;
				B[j+5][i] = v6;
				B[j+6][i] = v7;
				B[j+7][i] = v8;
			}
		for (i = n; i < N; ++i)
			for (j = m; j < M; ++j)
			{
				v1 = A[i][j];
				B[j][i] = v1;
			}
		for (i = 0; i < N; ++i)
			for (j = m; j < M; ++j)
			{
				v1 = A[i][j];
				B[j][i] = v1;
			}
		for (i = n; i < N; ++i)
			for (j = 0; j < M; ++j)
			{
				v1 = A[i][j];
				B[j][i] = v1;
			}
	}
```

