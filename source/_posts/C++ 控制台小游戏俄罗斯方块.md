---
title: "C++ 控制台小游戏俄罗斯方块"
date: 2018-01-26 19:54:27
tags: [cpp, game]
---

同样是寒假之前就开的坑, 在期末考试之前就写了一大半, 今天把坑给填上了, 效果还不错. 多线程操控变量真的是坑巨多啊, 干脆不用算了.  

<!-- more -->

```cpp
#include <iostream>
#include <cstdlib>
#include <vector>
#include <string>
#include <Windows.h>
#include <random>
#include <ctime> 

using namespace std;

#define WIDTH 15 //在控制台中一个单元格的宽度为2, 高度为1
#define HIGHT 30
#define KEY_DOWN(VK_NONAME) ((GetAsyncKeyState(VK_NONAME) & 0x8000) ? 1:0) 

#define LEFT 1
#define RIGHT 2
#define DOWN 3
#define ROTATE 4

void HideCursor()
{
	CONSOLE_CURSOR_INFO cursor_info = { 1, 0 };
	SetConsoleCursorInfo(GetStdHandle(STD_OUTPUT_HANDLE), &cursor_info);
}

struct point
{
	int x = 0;
	int y = 0;

	point() :x(0), y(0) {};
	point(int _x, int _y) :x(_x), y(_y) {};

	point operator- (point input)
	{
		input.x -= x;
		input.y -= y;
		return input;
	}

	point operator+ (point input)
	{
		input.x += x;
		input.y += y;
		return input;
	}

};

class myScreen
{
public:
	string dot = "■";
	string blank = "  ";

	void print(int x, int y ,string input)
	{
		COORD coord; // coordinates  
		coord.X = x * 2; coord.Y = y; // X and Y coordinates  
		SetConsoleCursorPosition(GetStdHandle(STD_OUTPUT_HANDLE), coord); // moves to the coordinates
		cout << input;
	}

	void printVector(vector<point> &temp, string input)
	{
		for (unsigned int i = 0; i < temp.size(); ++i)
		{
			print(temp[i].x, temp[i].y, input);			
		}
	}

	void printFixedBlock(vector<vector<int>> &temp,int currColor)
	{
		for (unsigned int i = 0; i < WIDTH; ++i)
		{
			for (unsigned int j = 0; j < HIGHT; ++j)
			{
				if (temp[i][j] != 0) 
				{
					SetConsoleTextAttribute(GetStdHandle(STD_OUTPUT_HANDLE), temp[i][j]); //移动方块的同时保持颜色不变
					print(i + 1, j + 1, dot);
				}
				
				if (temp[i][j] == 0) print(i + 1, j + 1, blank);
			}
		}
		SetConsoleTextAttribute(GetStdHandle(STD_OUTPUT_HANDLE), currColor); //恢复成原来的颜色
	}
};

class myTetrisBlocks
{ //S、Z、L、J、I、O、T
private:
	vector<point> S1 = { point(0,0),point(0,1),point(1,0),point(-1,1) };
	vector<point> S2 = { point(0,0),point(0,-1),point(1,0),point(1,1) };

	vector<point> Z1 = { point(0,0),point(0,-1),point(-1,0),point(-1,1) };
	vector<point> Z2 = { point(0,0),point(0,1),point(-1,0),point(1,1) };

	vector<point> L1 = { point(0,0),point(1,0),point(-1,0),point(-1,-1) };
	vector<point> L2 = { point(0,0),point(0,1),point(1,-1),point(0,-1) };
	vector<point> L3 = { point(0,0),point(1,0),point(-1,0),point(1,1) };
	vector<point> L4 = { point(0,0),point(0,1),point(-1,1),point(0,-1) };

	vector<point> J1 = { point(0,0),point(1,0),point(-1,0),point(1,-1) };
	vector<point> J2 = { point(0,0),point(0,1),point(1,1),point(0,-1) };
	vector<point> J3 = { point(0,0),point(1,0),point(-1,0),point(-1,1) };
	vector<point> J4 = { point(0,0),point(0,1),point(-1,-1),point(0,-1) };

	vector<point> I1 = { point(0,0),point(1,0),point(2,0),point(-1,0) };
	vector<point> I2 = { point(0,0),point(0,-1),point(0,1),point(0,2) };

	vector<point> O1 = { point(0,0),point(1,0),point(1,1),point(0,1) };

	vector<point> T1 = { point(0,0),point(1,0),point(-1,0),point(0,-1) };
	vector<point> T2 = { point(0,0),point(1,0),point(0,1),point(0,-1) };
	vector<point> T3 = { point(0,0),point(1,0),point(-1,0),point(0,1) };
	vector<point> T4 = { point(0,0),point(-1,0),point(0,1),point(0,-1) };


public:
	//========================================S
	vector<point> getS1(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < S1.size(); ++i)
		{
			temp.emplace_back(S1[i] + center);
		}
		return temp;
	}

	vector<point> getS2(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < S2.size(); ++i)
		{
			temp.emplace_back(S2[i] + center);
		}
		return temp;
	}

	//========================================Z
	vector<point> getZ1(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < Z1.size(); ++i)
		{
			temp.emplace_back(Z1[i] + center);
		}
		return temp;
	}

	vector<point> getZ2(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < Z2.size(); ++i)
		{
			temp.emplace_back(Z2[i] + center);
		}
		return temp;
	}

	//========================================L
	vector<point> getL1(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < L1.size(); ++i)
		{
			temp.emplace_back(L1[i] + center);
		}
		return temp;
	}

	vector<point> getL2(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < L2.size(); ++i)
		{
			temp.emplace_back(L2[i] + center);
		}
		return temp;
	}

	vector<point> getL3(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < L3.size(); ++i)
		{
			temp.emplace_back(L3[i] + center);
		}
		return temp;
	}

	vector<point> getL4(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < L4.size(); ++i)
		{
			temp.emplace_back(L4[i] + center);
		}
		return temp;
	}
	//========================================J
	vector<point> getJ1(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < J1.size(); ++i)
		{
			temp.emplace_back(J1[i] + center);
		}
		return temp;
	}

	vector<point> getJ2(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < J2.size(); ++i)
		{
			temp.emplace_back(J2[i] + center);
		}
		return temp;
	}

	vector<point> getJ3(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < J3.size(); ++i)
		{
			temp.emplace_back(J3[i] + center);
		}
		return temp;
	}

	vector<point> getJ4(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < J4.size(); ++i)
		{
			temp.emplace_back(J4[i] + center);
		}
		return temp;
	}
	//========================================I
	vector<point> getI1(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < I1.size(); ++i)
		{
			temp.emplace_back(I1[i] + center);
		}
		return temp;
	}

	vector<point> getI2(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < I2.size(); ++i)
		{
			temp.emplace_back(I2[i] + center);
		}
		return temp;
	}

	//========================================O
	vector<point> getO1(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < O1.size(); ++i)
		{
			temp.emplace_back(O1[i] + center);
		}
		return temp;
	}

	//========================================T
	vector<point> getT1(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < T1.size(); ++i)
		{
			temp.emplace_back(T1[i] + center);
		}
		return temp;
	}

	vector<point> getT2(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < T2.size(); ++i)
		{
			temp.emplace_back(T2[i] + center);
		}
		return temp;
	}

	vector<point> getT3(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < T3.size(); ++i)
		{
			temp.emplace_back(T3[i] + center);
		}
		return temp;
	}

	vector<point> getT4(point center)
	{
		vector<point> temp;
		for (unsigned int i = 0; i < T4.size(); ++i)
		{
			temp.emplace_back(T4[i] + center);
		}
		return temp;
	}
};

class myTetris :private myScreen, private myTetrisBlocks
{
private:
	vector<vector<int>> fixedBlock; //int值为0表示没有方块, 非0的int值表示颜色, 
	vector<point> movingBlock;
	point movingCenter;
	default_random_engine e;
	int movingType;
	int score = 0;
	int currColor;
	
	void setColor(int color)
	{
		currColor = color;
		SetConsoleTextAttribute(GetStdHandle(STD_OUTPUT_HANDLE), color);
	}
public:
	bool isDead = false;

	myTetris()
	{
		e.seed(static_cast<unsigned int>(time(0)));//将时间作为随机数生成器种子

		vector<int> temp(HIGHT, 0);
		for (int i = 0; i < WIDTH; ++i) //初始化容器大小
		{
			fixedBlock.push_back(temp);
		}

		string commond;
		commond = "mode con cols=" + to_string(WIDTH * 2 + 4) + " lines=" + to_string(HIGHT + 6); //调整窗口大小
		system(commond.c_str());
		system("cls");//清屏

		for (int i = 0; i < WIDTH + 2; ++i) //生成顶部,有边缘的原因, 所以额外+4
		{
			if (i == 0)
			{
				print(i, 0, "┌");
				continue;
			}
			if (i == WIDTH + 1)
			{
				print(i, 0, "┐");
				continue;
			}
			print(i, 0, "─");
		}

		for (int i = 0; i < WIDTH + 2; ++i) //生成底部,有边缘的原因, 所以额外+4
		{
			if (i == 0)
			{
				print(i, HIGHT + 1, "└");
				continue;
			}
			if (i == WIDTH + 1)
			{
				print(i, HIGHT + 1, "┘");
				continue;
			}
			print(i, HIGHT + 1, "─");
		}

		for (int i = 1; i < HIGHT + 1; ++i) //生成左右边框
		{
			print(0, i, "│");
			print(WIDTH + 1, i, "│");
		}

		print(WIDTH / 2 - 5, HIGHT + 2, "俄罗斯方块  得分: 0"); //标题
	};

	void newBlock()
	{
		movingBlock.clear();
		
		static std::uniform_int_distribution<unsigned int> b(1, 19);
		static std::uniform_int_distribution<unsigned int> c(1, 7);
		unsigned int randNum = b(e);
		setColor(c(e)); //生成颜色随机的方块

		movingCenter = point(WIDTH / 2, 2);
		switch (randNum)
		{
		case 1:
			movingBlock = getS1(movingCenter);
			movingType = 1;
			break;

		case 2:
			movingBlock = getS2(movingCenter);
			movingType = 2;
			break;

		case 3:
			movingBlock = getZ1(movingCenter);
			movingType = 3;
			break;

		case 4:
			movingBlock = getZ2(movingCenter);
			movingType = 4;
			break;

		case 5:
			movingBlock = getL1(movingCenter);
			movingType = 5;
			break;

		case 6:
			movingBlock = getL2(movingCenter);
			movingType = 6;
			break;

		case 7:
			movingBlock = getL3(movingCenter);
			movingType = 7;
			break;

		case 8:
			movingBlock = getL4(movingCenter);
			movingType = 8;
			break;

		case 9:
			movingBlock = getJ1(movingCenter);
			movingType = 9;
			break;

		case 10:
			movingBlock = getJ2(movingCenter);
			movingType = 10;
			break;

		case 11:
			movingBlock = getJ3(movingCenter);
			movingType = 11;
			break;

		case 12:
			movingBlock = getJ4(movingCenter);
			movingType = 12;
			break;

		case 13:
			movingBlock = getI1(movingCenter);
			movingType = 13;
			break;

		case 14:
			movingBlock = getI2(movingCenter);
			movingType = 14;
			break;

		case 15:
			movingBlock = getO1(movingCenter);
			movingType = 15;
			break;


		case 16:
			movingBlock = getT1(movingCenter);
			movingType = 16;
			break;


		case 17:
			movingBlock = getT2(movingCenter);
			movingType = 17;
			break;

		case 18:
			movingBlock = getT3(movingCenter);
			movingType = 18;
			break;


		case 19:
			movingBlock = getT4(movingCenter);
			movingType = 19;
			break;

		}

		if (!checkBlock(movingBlock))//检查新生成的方块是否与固定方块重叠
		{
			isDead = true;
		}

		printVector(movingBlock, dot);
	}

	bool checkBlock(vector<point> &temp) //检查方块是否越界或者会与已有方块重叠
	{
		for (unsigned int i = 0; i<temp.size(); ++i)
		{
			if (temp[i].y > HIGHT || temp[i].x > WIDTH || temp[i].x <= 0 || fixedBlock[temp[i].x - 1][temp[i].y - 1] != 0)
			{
				return false;
			}
		}
		return true;
	}

	void rotate()
	{
		printVector(movingBlock, blank);
		vector<point> temp;

		switch (movingType)
		{
		case 1:
			temp = getS2(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 2;
			}		
			break;

		case 2:
			temp = getS1(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 1;
			}
			break;

		case 3:
			temp = getZ2(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 4;
			}
			break;

		case 4:
			temp = getZ1(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 3;
			}
			break;

		case 5:
			temp = getL2(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 6;
			}
			break;

		case 6:
			temp = getL3(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 7;
			}
			break;

		case 7:
			temp = getL4(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 8;
			}
			break;

		case 8:
			temp = getL1(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 5;
			}
			break;

		case 9:
			temp = getJ2(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 10;
			}
			break;

		case 10:
			temp = getJ3(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 11;
			}
			break;

		case 11:
			temp = getJ4(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 12;
			}
			break;

		case 12:
			temp = getJ1(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 9;
			}
			break;

		case 13:
			temp = getI2(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 14;
			}
			break;

		case 14:
			temp = getI1(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 13;
			}
			break;

		case 15:
			//15是方块, 跳过 
			break;

		case 16:
			temp = getT2(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 17;
			}
			break;


		case 17:
			temp = getT3(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 18;
			}
			break;

		case 18:
			temp = getT4(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 19;
			}
			break;


		case 19:
			temp = getT1(movingCenter);
			if (checkBlock(temp))
			{
				movingBlock = temp;
				movingType = 16;
			}
			break;
		}

		printVector(movingBlock, dot);
	}

	bool moveDawn() 
	{
		bool touchDown = false;
		printVector(movingBlock, blank); //将上次的图像清空

		for (unsigned int i = 0; i < movingBlock.size(); ++i)
		{
			movingBlock[i].y += 1; //向下移一位
			if (movingBlock[i].y > HIGHT || fixedBlock[movingBlock[i].x - 1][movingBlock[i].y - 1] != 0) //检查是否碰到底部或者碰到固定的方块
			{				
				touchDown = true;
			}
		}
		
		if (touchDown)
		{	
			for (unsigned int i = 0; i < movingBlock.size(); ++i)
			{			
				movingBlock[i].y -= 1; //如果碰到了其他的方块, 退回一位
				fixedBlock[movingBlock[i].x - 1][movingBlock[i].y - 1] = currColor; //录入到固定方块的容器中去		
			}
			printVector(movingBlock, dot); //输出这次的图像
			return true;
		}
		else
		{
			movingCenter.y += 1;
			printVector(movingBlock, dot); //输出这次的图像
			return false;
		}		
	}

	void moveLeft()
	{
		bool touchLeft = false;
		printVector(movingBlock, blank);	

		for (unsigned int i = 0; i < movingBlock.size(); ++i)
		{
			movingBlock[i].x -= 1; //向左移一位
			if (movingBlock[i].x <= 0 || fixedBlock[movingBlock[i].x - 1][movingBlock[i].y - 1] != 0) //检查是否碰到左边界或者碰到固定的方块
			{
				touchLeft = true;
			}
		}

		if (touchLeft)
		{
			for (unsigned int i = 0; i < movingBlock.size(); ++i)
			{
				movingBlock[i].x += 1; //如果碰到了其他的方块, 退回一位
			}
			printVector(movingBlock, dot); //输出这次的图像
			return;
		}
		else
		{
			movingCenter.x -= 1;
			printVector(movingBlock, dot); //输出这次的图像
			return;
		}
	}

	void moveRight()
	{
		bool touchRight = false;
		printVector(movingBlock, blank);

		for (unsigned int i = 0; i < movingBlock.size(); ++i)
		{
			movingBlock[i].x += 1; //向右移一位
			if (movingBlock[i].x > WIDTH || fixedBlock[movingBlock[i].x - 1][movingBlock[i].y - 1] != 0) //检查是否碰到右边界或者碰到固定的方块
			{
				touchRight = true;
			}
		}

		if (touchRight)
		{
			for (unsigned int i = 0; i < movingBlock.size(); ++i)
			{
				movingBlock[i].x -= 1; //如果碰到了其他的方块, 退回一位
			}
			printVector(movingBlock, dot); //输出这次的图像
			return;
		}
		else
		{
			movingCenter.x += 1;
			printVector(movingBlock, dot); //输出这次的图像
			return;
		}
	}

	void checkLines() //检查是否将连成一行
	{
		int addPoints = 10;
		for (int i = 0; i < HIGHT; ++i)
		{
			bool isFullOfLine = true;
			for (int j = 0; j < WIDTH; ++j)
			{
				if (fixedBlock[j][i] == 0) isFullOfLine = false; //检查这一行是否是满的
			}
			if (isFullOfLine) //如果是满的将上面的所有方块向下移一行
			{
				score += addPoints;
				addPoints *= 2; //使得一次清空多行分数加倍
				for (int k = i; k > 0; --k)
				{
					for (int j = 0; j < WIDTH; ++j)
					{
						fixedBlock[j][k] = fixedBlock[j][k - 1];
					}
				}
			}

		}	
		printFixedBlock(fixedBlock,currColor);
		setColor(7);//切回白色
		print(WIDTH / 2 + 4, HIGHT + 2, to_string(score)); //输出更新后的分数
		setColor(currColor);
	}

	void exit()
	{
		setColor(BACKGROUND_RED);
		print(WIDTH / 2 - 1, HIGHT + 3, "GAME OVER!");
		cout << endl;
		setColor(7);
	}
};

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

int main()
{
	int diffcult;
	cout << "输入难度(1~10): ";

	while (cin >> diffcult && (diffcult < 1 || diffcult >10))
	{
		cout << "请输入合法的难度!" << endl;
		cout << "输入难度(1~10): ";
	}
        diffcult = 11 - diffcult;
        int time = diffcult * 10;

	HideCursor();
	myTetris tetris;
	tetris.newBlock();
	int noKeyDownCount = 0;
	int moveCount = 0;
	

	while (true)  //操作逻辑是这样, 如果什么按键都没按, 达到一定次数就向下移一位, 同时次数清零. 如果是平移或者旋转, 达到一定次数同样向下移一位, 同时次数清零. 因为多线程容易导致BUG(方块错位等等), 最后还是选择单线程算了. 在单线程下这应该是最优解了0.0
	{
		if (check('O') || tetris.isDead)
		{
			tetris.exit();
			break;
		}

		if (noKeyDownCount == 10 || moveCount == diffcult)
		{
			if (tetris.moveDawn())
			{
				tetris.checkLines();
				tetris.newBlock();
			}
			noKeyDownCount = 0;
			moveCount = 0;
		}

		if (check('W'))
		{
			tetris.rotate();
			moveCount++;
			Sleep(100);
			continue;
		}
		if (check('A'))
		{
			tetris.moveLeft();
			moveCount++;
			Sleep(100);
			continue;
		}
		if (check('D'))
		{
			tetris.moveRight();
			moveCount++;
			Sleep(100);
			continue;
		}
		if (check('S'))
		{
			if (tetris.moveDawn())
			{
				tetris.checkLines();
				tetris.newBlock();
			}
			moveCount = 0;
			Sleep(20);
			continue;
		}
		noKeyDownCount++;
		Sleep(time);
	}
	
	Sleep(3000);
	system("pause");
    return 0;
}
```