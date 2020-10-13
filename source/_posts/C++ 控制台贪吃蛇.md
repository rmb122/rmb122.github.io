---
title: "C++ 控制台贪吃蛇"
date: 2017-12-14 12:37:25
tags: [cpp, game]
---

无聊写的... 效率感人...  别人写的都是越到后面越难, 我这个是越来越简单 （逃 Update in 2017/12/15 我终于知道什么原因了. 我！T！M！D！的写错了...  脑子一抽把输出写到for()里面了, 导致随着蛇的长度增加重复输出就很卡 QAQ 现在这个版本已经修好了 (顺便加了点功能 逃 Update in 2017/12/17 换了一种输出方式, 效率疯狂up, 顺便修了点bug. 

<!-- more -->

```cpp
#include <iostream>
#include <tchar.h>
#include <cstdlib>
#include <vector>
#include <string>
#include <windows.h> 
#include <random>
#include <ctime> 
#include <thread>

#define LEFT 1
#define RIGHT 2
#define DOWN 3
#define UP 4
#define KEY_DOWN(VK_NONAME) ((GetAsyncKeyState(VK_NONAME) & 0x8000) ? 1:0) 
#define isDIRECTION true //是否显示头部方向

using namespace std;

int CONSIZE;
int direction = UP;
bool isTurned = false;

void gotoxy(int x, int y)
{
	COORD coord; // coordinates  
	coord.X = x; coord.Y = y; // X and Y coordinates  
	SetConsoleCursorPosition(GetStdHandle(STD_OUTPUT_HANDLE), coord); // moves to the coordinates  
}

class Myscreen
{
public:
	string dot = "■";
	string blank = "  ";
	string target = "○";
	string up = "↑";
	string left = "←"; 
	string right = "→";
	string down = "↓";
};

class Mysnake 
{
private:
	vector<vector<int>> snake;  //第一维代表蛇的一个身体, 第二维两个参数代表x和y
	Myscreen screen;
	vector<int> target;

	void output(int direction)
	{		
		for (int i = 0; i < snake.size(); ++i)
		{
			if (isDIRECTION)
			{
				if (i == 0)
				{
					switch (direction)
					{
					case UP:
						gotoxy(snake[i][0] * 2, snake[i][1]);
						cout << screen.up;
						break;
					case DOWN:
						gotoxy(snake[i][0] * 2, snake[i][1]);
						cout << screen.down;
						break;
					case LEFT:
						gotoxy(snake[i][0] * 2, snake[i][1]);
						cout << screen.left;
						break;
					case RIGHT:
						gotoxy(snake[i][0] * 2, snake[i][1]);
						cout << screen.right;
						break;
					}
				}
				else
				{
					gotoxy(snake[i][0] * 2, snake[i][1]);
					cout << screen.dot;
				}
			}
			else
			{
				gotoxy(snake[i][0] * 2, snake[i][1]);
				cout << screen.dot;
			}
		}
	}

	bool detecter()
	{
		int x = snake[0][0];
		int y = snake[0][1];
		if (x == target[0] && y == target[1]) //如果头的位置和目标相同, 返回true
		{
			return true;
		}

		for (int i = 1; i < snake.size() - 1; ++i) //如果头的位置和蛇身体(除了自己的尾巴)相同, 将isDead设为true
		{
			if (snake[0][0] == snake[i][0] && snake[0][1] == snake[i][1])
			{
				isDead = true;
			}
		}

		return false; //都不相同返回false
	}

	void delteLast()
	{
		gotoxy(snake[snake.size() - 1][0]*2, snake[snake.size() - 1][1]);
		cout << screen.blank;
	}

public:
	bool isDead = false;
	int socre = -1; //初始化时会执行一次newtarget, 所以是-1

	void newTarget()
	{
		default_random_engine e(static_cast<unsigned int>(time(0)));
		static std::uniform_int_distribution<unsigned> u(1, CONSIZE);
		while (true)
		{
			int randx = u(e);
			int randy = u(e);

			for (int i = 0; i < snake.size(); ++i)
			{
				if ((randx == snake[i][0]) && (randy == snake[i][1])) //如果目标与蛇的一部分重叠, 就重新生成一个数
				{
					goto retry;
				}	
			}

			target.clear(); //将之前得到的目标清空
			target.push_back(randx);
			target.push_back(randy);
			gotoxy(randx * 2, randy);
			cout << screen.target;
			socre++;
			return;

		retry:
			continue; //编译器要求这里要有一行233
		}	
	}

	void initialize()
	{	
		string commond;
		commond = "mode con cols="+to_string(CONSIZE*2+5) +" lines="+ to_string(CONSIZE + 6); //调整窗口大小
		system(commond.c_str());
		system("cls");//清屏

		for (int i = 0; i < CONSIZE * 2 + 4; ++i) //生成顶部,有边缘的原因, 所以额外+4
		{
			if (i == 0)
			{
				cout << "┌";
				continue;
			}
			if (i == CONSIZE * 2 + 3)
			{
				cout << "┐";
				continue;
			}
			cout << "─";
		}

		gotoxy(0, CONSIZE+1);//跳到底部
		for (int i = 0; i < CONSIZE * 2 + 4; ++i) //生成底部,有边缘的原因, 所以额外+4
		{
			if (i == 0)
			{
				cout << "└";
				continue;
			}
			if (i == CONSIZE * 2 + 3)
			{
				cout << "┘";
				continue;
			}
			cout << "─";
		}
		
		for (int i = 1; i < CONSIZE+1 ; ++i) //生成左右边框
		{
			gotoxy(0, i);
			cout << "│";
			gotoxy(CONSIZE * 2 + 3, i); //跳到右边
			cout << "│";
		}

		gotoxy(CONSIZE/2\*2 -2, CONSIZE + 2); //x=CONSIZE/2\*2 -6(3个汉字) +4(两个边框)
		cout << "贪吃蛇";

		vector<int> body; 
		body.push_back(CONSIZE / 2);
		body.push_back(CONSIZE / 2);
		snake.push_back(body); //生成第一个蛇的身体
		output(UP);
	}

	void forward(int direction)
	{
		vector<int> body;
		switch (direction)
		{
		case UP:
			if (snake[0][1] == 1) //如果在第一行, 直接添加到最后一行
			{
				body.push_back(snake[0][0]);
				body.push_back(CONSIZE);            //添加一个身体到最后一行
				snake.insert(snake.begin(), body);
				if (!detecter()) //如果没有吃到目标删掉最后一个身体
				{		
					delteLast();
					snake.pop_back();
				}
				else
				{
					newTarget();
				}
			}
			else
			{
				
				body.push_back(snake[0][0]);
				body.push_back(snake[0][1] - 1); //添加一个身体到第一个身体的上面	
				snake.insert(snake.begin(), body);
				
				if (!detecter()) //如果没有吃到目标删掉最后一个身体
				{		
					delteLast();
					snake.pop_back();
				}
				else
				{
					newTarget();
				}
			}

			output(direction);
			break;
		case DOWN:

			if (snake[0][1] == CONSIZE) //如果在最后一行, 直接添加到第一行
			{
				body.push_back(snake[0][0]);
				body.push_back(1);               //添加一个身体到第一行		
				snake.insert(snake.begin(), body);
				
				if (!detecter()) //如果没有吃到目标删掉最后一个身体,吃到了不删除相当于增长了
				{		
					delteLast(); //清空最后一个身体处的的区域
					snake.pop_back();
				}
				else
				{
					newTarget();
				}
			}
			else
			{
				body.push_back(snake[0][0]);
				body.push_back(snake[0][1] + 1); //添加一个身体到第一个身体的下面
				snake.insert(snake.begin(), body);
				
				if (!detecter()) //如果没有吃到目标删掉最后一个身体
				{			
					delteLast();
					snake.pop_back();
				}
				else
				{
					newTarget();
				}
			}

			output(direction);
			break;
		case RIGHT:
			if (snake[0][0] == CONSIZE) //如果在最右一列, 直接添加到第一列
			{
				body.push_back(1);
				body.push_back(snake[0][1]);     //添加一个身体到第一列			
				snake.insert(snake.begin(), body);
				
				if (!detecter()) //如果没有吃到目标删掉最后一个身体
				{			
					delteLast();
					snake.pop_back();
				}
				else
				{
					newTarget();
				}
			}
			else
			{
				body.push_back(snake[0][0] + 1);
				body.push_back(snake[0][1]);     //添加一个身体到第一个身体的右面		
				snake.insert(snake.begin(), body);
				
				if (!detecter()) //如果没有吃到目标删掉最后一个身体
				{			
					delteLast();
					snake.pop_back();
				}
				else
				{
					newTarget();
				}
			}

			output(direction);
			break;
		case LEFT:
			if (snake[0][0] == 1) //如果在第一列, 直接添加到最后一列
			{
				body.push_back(CONSIZE);
				body.push_back(snake[0][1]);     //添加一个身体到最后一列
				snake.insert(snake.begin(), body);
				
				if (!detecter()) //如果没有吃到目标删掉最后一个身体
				{		
					delteLast();
					snake.pop_back();
				}
				else
				{
					newTarget();
				}
			}
			else
			{
				body.push_back(snake[0][0] - 1);
				body.push_back(snake[0][1]);     //添加一个身体到第一个身体的左面
				snake.insert(snake.begin(), body);
				
				if (!detecter()) //如果没有吃到目标删掉最后一个身体
				{			
					delteLast();
					snake.pop_back();
				}
				else
				{
					newTarget();
				}
			}
			
			output(direction);
			break;		
		}
	}

	void add(int direction) //就是forward()去掉判断是否只有一个和删除最后一个身体功能的部分
	{
		vector<int> body;
		switch (direction)
		{
		case UP:
			if (snake[0][1] == 1) //如果在第一行, 直接添加到最后一行
			{		
				body.push_back(snake[0][0]);
				body.push_back(CONSIZE);            //添加一个身体到最后一行
				snake.insert(snake.begin(), body);
			}
			else
			{
				body.push_back(snake[0][0]);
				body.push_back(snake[0][1] - 1); //添加一个身体到第一个身体的上面
				snake.insert(snake.begin(), body);
			}
			output(direction);
			break;
		case DOWN:
			if (snake[0][1] == CONSIZE) //如果在最后一行, 直接添加到第一行
			{
				body.push_back(snake[0][0]);
				body.push_back(1);               //添加一个身体到第一行
				snake.insert(snake.begin(), body);
			}
			else
			{
				body.push_back(snake[0][0]);
				body.push_back(snake[0][1] + 1); //添加一个身体到第一个身体的下面
				snake.insert(snake.begin(), body);
			}		
			output(direction);
			break;
		case RIGHT:
			if (snake[0][0] == CONSIZE) //如果在最右一列, 直接添加到第一列
			{
				body.push_back(1);
				body.push_back(snake[0][1]);     //添加一个身体到第一列
				snake.insert(snake.begin(), body);
			}
			else
			{
				body.push_back(snake[0][0] + 1);
				body.push_back(snake[0][1]);     //添加一个身体到第一个身体的右面
				snake.insert(snake.begin(), body);
			}
			output(direction);
			break;
		case LEFT:
			if (snake[0][0] == 1) //如果在第一列, 直接添加到最后一列
			{
				body.push_back(CONSIZE);
				body.push_back(snake[0][1]);     //添加一个身体到最后一列
				snake.insert(snake.begin(), body);
			}
			else
			{
				body.push_back(snake[0][0] - 1);
				body.push_back(snake[0][1]);     //添加一个身体到第一个身体的左面
				snake.insert(snake.begin(), body);
			}
			output(direction);
			break;
		}
	}
};

Mysnake snake;//<=========================方便下面的controlthread检测

void HideCursor()
{
	CONSOLE_CURSOR_INFO cursor_info = { 1, 0 };
	SetConsoleCursorInfo(GetStdHandle(STD_OUTPUT_HANDLE), &cursor_info);
}

bool check(char c)
{
	if (KEY_DOWN(c))
	{
		return true;
	}
	else
	{
		return false;
	}
}


void control()
{
	while (true && !snake.isDead) //死了以后线程结束
	{
		if (check('W') && (direction != DOWN) && (!isTurned))
		{
			isTurned = true;
			direction = UP;
			Sleep(20);
			continue;
		}
		if (check('A') && (direction != RIGHT) && (!isTurned))
		{
			isTurned = true;
			direction = LEFT;
			Sleep(20);
			continue;
		}
		if (check('S') && (direction != UP) && (!isTurned))
		{
			isTurned = true;
			direction = DOWN;
			Sleep(20);
			continue;
		}
		if (check('D') && (direction != LEFT) && (!isTurned))
		{
			isTurned = true;
			direction = RIGHT;
			Sleep(20);
			continue;
		}
		Sleep(20);
	}
}

int getStart()
{
	int difficult;
	cout << "输入你选择的大小(>10)" << endl;
	while (cin >> CONSIZE && (CONSIZE<10))
	{
		cout << "重新输入大小！" << endl;
	}
	cout << "输入你选择的难度(1~10)" << endl;
	while (cin >> difficult && (difficult>10 || difficult <1))
	{
		cout << "重新输入合法的难度！" << endl;
	}
	return difficult;
}

int main()
{
	int timeToSleep = 200 - getStart() * 20; //Sleep()以毫秒为单位

	HideCursor();

	snake.initialize();
	snake.add(UP); //调整出生时的形状
	snake.add(UP);
	snake.add(UP);
	snake.newTarget();

	thread controlThread(control); //启动按键检测 PS:有很小几率出现bug（两个按键一起按... 所以不要作死是没事的← ←）
	controlThread.detach();

	while(!snake.isDead) //碰到自己游戏结束
	{	
		snake.forward(direction);
		isTurned = false;//转过向后再检测下一次按键事件
		Sleep(timeToSleep);	
	}

	gotoxy(CONSIZE / 2 * 2 - 5, CONSIZE + 3);
	cout << "You are dead!";
	gotoxy(CONSIZE / 2 * 2 - 6, CONSIZE + 4);
	cout<<"Your socre is: " << snake.socre << endl;

	Sleep(3000); 
	system("pause");
    return 0;
}
```